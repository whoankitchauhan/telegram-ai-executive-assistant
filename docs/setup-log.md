# Setup Log

A more detailed, chronological account of how this was built and debugged — kept because the debugging was most of the actual learning.

## 1. Telegram bot

- Created via [@BotFather](https://t.me/BotFather) on Telegram (`/newbot`), got the API token.

## 2. n8n via Docker

```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```
- `-v n8n_data:/home/node/.n8n` persists workflows/credentials across container restarts (the `--rm` flag removes the container on stop, but the named volume survives).

## 3. Public webhook via ngrok

Telegram requires an HTTPS webhook — a local `http://localhost:5678` isn't reachable from Telegram's servers.

```bash
ngrok http 5678
```

Then n8n is restarted with the ngrok URL so it registers the correct webhook address:
```bash
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n \
  -e N8N_WEBHOOK_URL=https://YOUR-NGROK-URL.ngrok-free.dev/ \
  docker.n8n.io/n8nio/n8n
```

**Note:** ngrok's free tier generates a new random subdomain on every restart (unless it happens to reuse one), so this URL needs to be re-checked and swapped in each session.

## 4. Base pipeline

- `Telegram Trigger` (event: "On message") → `Send a text message`, hardcoded echo reply first — to confirm the full round trip works before adding any AI.
- Debugged an early `ECONNRESET` / SSL handshake error on the Telegram credential — root cause turned out to be network-level interference from a restrictive network (college hostel WiFi blocking/inspecting Docker's outbound HTTPS traffic). Confirmed via `docker exec -it n8n wget -qO- https://api.telegram.org`, fixed by switching to a mobile hotspot.

## 5. AI Agent

- Added `AI Agent` node between trigger and send.
- Chat model: **Groq**, chosen for its generous free tier (OpenAI's free trial credits are no longer offered as of mid-2025).
- Initial model: `openai/gpt-oss-120b`. Later switched to `openai/gpt-oss-20b` to resolve rate-limit errors (8,000 tokens/minute cap on Groq's free tier).

## 6. Google Calendar OAuth

- Enabled the Calendar API in Google Cloud Console.
- Created an OAuth 2.0 Client (type: Web application).
- Redirect URI: `<n8n-url>/rest/oauth2-credential/callback` — must match **exactly** (including protocol and trailing behavior) or Google returns `redirect_uri_mismatch`.
- **Gotcha:** if n8n's `N8N_WEBHOOK_URL` is set to an ngrok address, n8n generates its OAuth redirect using that same ngrok domain — not `localhost` — even during a Google sign-in initiated from a `localhost` browser tab. The redirect URI registered in Google Cloud has to match whichever base URL n8n is actually using at the time.
- **Gotcha:** ngrok's free-tier "you are about to visit this site" interstitial page can interrupt the OAuth callback redirect, causing the flow to fail with a generic `Unauthorized` error at the very last step. Fix: temporarily start n8n *without* `N8N_WEBHOOK_URL` set (so it uses plain `localhost`) to complete the Google OAuth flow, then restart with the ngrok URL for normal Telegram operation. The OAuth token persists in the Docker volume across restarts.
- Since the app is in Google's "Testing" publishing status (not verified/published), refresh tokens expire roughly weekly and the credential needs to be manually reconnected — a known limitation, not a bug.

## 7. Calendar tools on the AI Agent

Four Google Calendar tool nodes attached to the Agent's Tool input:
- **Get many events** (`Resource: Event`, `Operation: Get Many`) — `After`/`Before` fields set via `$fromAI("after")` / `$fromAI("before")` so the model controls the search range.
- **Create an event** (`Operation: Create`) — `start_time`/`end_time`/`title` via `$fromAI(...)`.
- **Update an event** (`Operation: Update`) — requires `event_id`, obtained by the model calling Get Many first.
- **Delete an event** (`Operation: Delete`) — same `event_id` requirement.

**Timezone bug:** early testing showed events being created at the correct clock time but with the calendar's internal `timeZone` field showing `America/New_York` instead of `Asia/Kolkata`, because the model wasn't given an explicit timezone anchor. Fixed by adding the current date/time and an explicit `+05:30` ISO offset requirement to the system prompt.

## 8. Memory

- **Short-term:** `Simple Memory` node (n8n's windowed conversation buffer), keyed by `{{ $('Telegram Trigger').item.json.message.chat.id }}` so each Telegram user gets an isolated memory thread. Window length: 10 messages initially, reduced to 5 to cut token usage.
- **Long-term:** an n8n Data Table (`user_facts`) with `chat_id` and `fact` columns, exposed to the Agent as two tools — `Insert row` (save a new fact) and `Get row(s)` (retrieve facts filtered by `chat_id`).
- Debugging note: the model initially didn't call the memory-read tool reliably — the system prompt had to explicitly instruct it to check memory "before doing anything else," rather than leaving it as an optional-sounding instruction.

## 9. Reply formatting

- `Append n8n Attribution` toggled off on the Send Message node, to remove the default "This message was sent automatically with n8n" footer.
- System prompt explicitly instructs the model to avoid raw JSON, ISO timestamps, or timezone codes in user-facing replies.

## 10. Publishing

- Workflow toggled to **Active/Published** so it listens continuously, instead of only responding while "Execute workflow" test mode is manually triggered.
- Note: an active Telegram Trigger workflow can't run in test mode simultaneously — this is a Telegram-specific n8n limitation, not a bug.
