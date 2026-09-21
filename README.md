# FreeScout customer ticketing PoC

A local proof of concept showing how a customer-facing ticketing service can be built on FreeScout. Agents work in FreeScout; customers raise and follow requests through a small portal (Brightline Help, a made-up broadband brand). Everything runs in Docker, including fake mail servers, so nothing leaves your machine.

The PoC demonstrates two integration styles:

- **Email channel (free, FreeScout core only).** The portal emails the support mailbox, FreeScout fetches it over IMAP and opens a ticket, and agent replies go back to the customer by email.
- **API channel (API & Webhooks module).** The portal creates tickets through the FreeScout REST API, shows each customer their requests and the conversation, lets them reply, and updates live when an agent acts, driven by FreeScout webhooks.

```
                         ┌──────────────────────────┐
  Customer browser ────▶ │ portal  :3000 (Node)     │
        ▲   SSE          │  - holds the API key     │
        └────────────────│  - verifies webhooks     │
                         └───┬──────────────┬───────┘
          email channel      │ SMTP         │ REST API (api channel)
                             ▼              ▼
                ┌────────────────┐   ┌──────────────────────────┐
                │ greenmail      │   │ freescout-app  :8080     │◀── agents
                │ support@ inbox │◀──│  IMAP fetch every minute │
                └────────────────┘   │  webhooks ──▶ portal     │
                                     └───────┬──────────┬───────┘
                                             │ SMTP     │
                                             ▼          ▼
                                   ┌────────────────┐  ┌──────────────┐
                                   │ mailpit :8025  │  │ freescout-db │
                                   │ customer inbox │  │ MariaDB      │
                                   └────────────────┘  └──────────────┘
```

| URL | What it is |
| --- | --- |
| http://localhost:8080 | FreeScout, the agent side |
| http://localhost:3000 | Customer portal |
| http://localhost:8025 | Mailpit, which catches every email FreeScout sends |

## Prerequisites

Docker Engine with the Compose plugin (`docker compose version`), about 2 GB of free RAM, and free host ports 3000, 3025, 3143, 8025 and 8080.

## 1. Start the stack

```bash
cp .env.example .env
docker compose up -d
docker compose logs -f freescout-app   # first boot takes 2–5 minutes (migrations, asset build)
```

When the log settles, open http://localhost:8080 (use exactly this URL, it must match `APP_URL`) and sign in with `admin@helpdesk.local` / `Admin12345`.

## 2. Create the support mailbox

In FreeScout go to **Manage → Mailboxes → New Mailbox**, name it `Customer Support` and use the address `support@helpdesk.local`.

Open the new mailbox's **Connection Settings**.

On **Sending Emails**, choose SMTP with server `mailpit`, port `1025` and no encryption. Leave username and password empty (Mailpit accepts any value if the form insists).

On **Receiving Emails**, use:

| Field | Value |
| --- | --- |
| Server | `greenmail` |
| Port | `3143` |
| Username | `support@helpdesk.local` |
| Password | anything, e.g. `support` (GreenMail runs with auth disabled) |
| Protocol | IMAP |
| Encryption | None |
| Validate certificate | off |
| Folders | `INBOX` |

Save and run the connection check. Optionally enable **Auto Reply** on the mailbox so customers get an acknowledgement, and set **Manage → Settings → Mail Settings** to SMTP `mailpit:1025` so agent notification emails also land in Mailpit.

The hostnames `greenmail` and `mailpit` are container names on the Compose network, which is why they work from inside FreeScout but not from your browser.

## 3. Demo A: email channel (no paid modules)

Send a test email straight from your terminal (curl speaks SMTP):

```bash
cat > /tmp/ticket.eml <<'EOF'
From: Ravi Kumar <ravi@example.com>
To: support@helpdesk.local
Subject: Internet drops every evening

It drops around 8pm every day and the router light turns red.
EOF

curl --url smtp://localhost:3025 \
  --mail-from ravi@example.com --mail-rcpt support@helpdesk.local \
  --upload-file /tmp/ticket.eml
```

Within about a minute the ticket shows up under the mailbox's Unassigned folder. Reply to it as the agent, then open Mailpit at http://localhost:8025 to see the email the customer receives.

Now do the same from the customer's side: open http://localhost:3000, enter a customer email and send a request. The portal is in email mode by default (`TICKET_CHANNEL=email`), so it only confirms the request was sent and points the customer to their inbox.

## 4. Demo B: API channel with live updates

This needs the official **API & Webhooks** module (https://freescout.net/module/api-webhooks/), a paid module with a lifetime licence per FreeScout instance.

1. Buy the module, then in FreeScout go to **Manage → Modules**, find API & Webhooks, enter the licence key and activate it.
2. Go to **Manage → Settings → API & Webhooks**. Copy the API key and the webhook secret.
3. Check the API works and note your mailbox id:

   ```bash
   export FS_KEY=paste-api-key-here
   curl -s -H "X-FreeScout-API-Key: $FS_KEY" http://localhost:8080/api/mailboxes
   ```

4. Register the webhook. The URL uses the `portal` container name because FreeScout calls it from inside Docker:

   ```bash
   curl -i -X POST http://localhost:8080/api/webhooks \
     -H "X-FreeScout-API-Key: $FS_KEY" -H "Content-Type: application/json" \
     -d '{"url":"http://portal:3000/webhooks/freescout","events":["convo.created","convo.assigned","convo.status","convo.agent.reply.created","convo.customer.reply.created"]}'
   ```

   You can do the same on the API & Webhooks settings page instead.

5. Edit `.env`:

   ```bash
   TICKET_CHANNEL=api
   FS_API_KEY=paste-api-key-here
   FS_WEBHOOK_SECRET=paste-webhook-secret-here
   FS_MAILBOX_ID=1
   ```

6. Recreate the portal so it picks up the new values: `docker compose up -d portal`, then check `docker compose logs portal` shows `channel=api`.

### Demo script

1. Customer: open http://localhost:3000 in one window, sign in as `ravi@example.com`, send a request. It appears in the list with its ticket number and the status "With support". The header shows "Live updates on".
2. Agent: in a second window, open the ticket in FreeScout, assign it to yourself and reply. The customer's portal updates on its own: the reply appears, the status changes to "Waiting on you" and a notice pops up. Mailpit also shows the reply email.
3. Agent: add an internal note. It never reaches the portal.
4. Customer: reply from the portal. In FreeScout the conversation goes back to Active with the customer's message.
5. Agent: close the ticket. The portal shows "Resolved", and the customer can still reply to reopen it.
6. Isolation: sign in to the portal as a different email. Ravi's tickets are not listed, and requesting `/api/tickets/<Ravi's id>` returns 404.

## How the portal maps to FreeScout

| Portal endpoint | FreeScout call |
| --- | --- |
| `POST /api/tickets` (email) | SMTP message to `support@helpdesk.local` |
| `POST /api/tickets` (api) | `POST /api/conversations` with a `customer` thread, then `GET /api/conversations/{id}` |
| `GET /api/tickets` | `GET /api/conversations?customerEmail=…&status=active,pending,closed` |
| `GET /api/tickets/:id` | `GET /api/conversations/{id}?embed=threads`, ownership check, notes and log lines removed |
| `POST /api/tickets/:id/replies` | `POST /api/conversations/{id}/threads` with `type: customer` |
| `POST /webhooks/freescout` | receives FreeScout webhooks, verifies `X-FreeScout-Signature` |
| `GET /api/events` | Server-Sent Events stream that forwards webhook updates to the customer's browser |

A few choices worth pointing out in a review:

- The API key never reaches the browser. The portal is the only thing that talks to FreeScout.
- `status` on the list call defaults to active only in FreeScout, so the portal asks for `active,pending,closed` explicitly.
- Every ticket read or reply first checks that the conversation belongs to the signed-in customer and answers 404 otherwise, so ids can't be probed.
- Thread bodies are email HTML. The server converts them to text and the browser inserts them as text nodes, so nothing from an email is ever parsed as HTML in the portal. Customer input is escaped before it is sent to FreeScout.
- Webhooks are verified with `base64(HMAC-SHA1(raw body, secret))` using a constant-time compare, which is why the webhook route reads the raw body before the JSON parser runs.
- Sign-in is a placeholder: the email typed by the user is trusted. In a real product the email comes from your own authenticated session.

## Troubleshooting

**FreeScout doesn't load or keeps redirecting.** Give the first boot a few minutes and watch `docker compose logs -f freescout-app`. Open the site at exactly http://localhost:8080; `127.0.0.1` or another port breaks sessions because it doesn't match `APP_URL`.

**Emails aren't turning into tickets.** Re-run the connection check on Receiving Emails. You can trigger a fetch by hand and see the error directly:

```bash
docker exec -it freescout-app bash
artisan freescout:fetch-emails
```

If the output mentions an IMAP SEARCH error, uncomment `FREESCOUT_APP_SINCE_WITHOUT_QUOTES_ON_FETCHING=true` in `docker-compose.yml` and run `docker compose up -d freescout-app`. Also check **Manage → System** for cron and queue status.

**Agent replies don't show up in Mailpit.** Check the mailbox's Sending Emails settings point at `mailpit:1025`, and look at **Manage → System** to confirm background jobs are running, since outgoing mail is queued.

**API calls return 404 for every endpoint.** The API & Webhooks module isn't activated.

**API calls return 401.** `FS_API_KEY` is wrong or the portal wasn't recreated after editing `.env`.

**The portal never updates live.** Check the webhook URL is `http://portal:3000/webhooks/freescout` (not localhost), look at the webhook log on the API & Webhooks settings page, and follow `docker compose logs -f portal`. A `signature mismatch` line means `FS_WEBHOOK_SECRET` doesn't match the secret in FreeScout.

**Start over from scratch.** `docker compose down -v` removes all containers and data volumes.

## Beyond the PoC

For a real deployment, replace GreenMail and Mailpit with your actual support mailbox (Microsoft 365, Google Workspace or your own mail server) over IMAP and SMTP with TLS, put FreeScout and the portal behind an HTTPS reverse proxy, pin the container image tags, and schedule database backups. In the portal, take the customer identity from your existing login, store the mapping between your customers and FreeScout customer ids, and move webhook handling onto a queue so slow processing never makes FreeScout retry.

If you don't need a custom-branded UI, FreeScout's paid End-User Portal module gives customers a ready-made portal and contact form widget for submitting tickets. A free community-built API module (github.com/mikeyperes/freescout-api-webhooks) also exists, but its endpoints and auth header differ from the official API this portal targets, and it is young, so treat it as unvetted.
