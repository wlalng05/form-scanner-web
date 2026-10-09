# Form Scanner (POC)

Mobile web app: photograph a paper form → Gemini reads it (via n8n) → review/edit → save to Google Sheets.

```
Phone ──► form-scanner-web.cmart-diaz.workers.dev   (this repo, Cloudflare Worker)
              │  GET  /webhook/fs-events
              │  POST /webhook/fs-decode ──► Gemini
              │  POST /webhook/fs-save   ──► Google Sheets
              ▼
          cmd-systems.app.n8n.cloud
```

## Repo layout

| Path | Purpose |
|---|---|
| `public/index.html` | The whole app (HTML, CSS, JS in one file). Settings are in the `CONFIG` block near the top of the script. |
| `wrangler.jsonc` | Tells Cloudflare to serve the `public/` folder. |
| `n8n/*.json` | The three n8n workflows, importable into n8n. |

---

## Setup (one time)

### 1. Google Sheet

Spreadsheet `10pttg9Xpkw0qEzcs70qvO70nPi5DmnXVJIlA0SsuWPc` with three tabs. Header rows must match exactly (lowercase).

**Events**: `event_id | event_name | target_tab | is_default | active`
**Fields**: `event_id | order | field_key | form_label | field_type | required | options | ai_hint`
**Data_EVT001**: `timestamp | event_id | name | email | edited`

- `field_type`: `text`, `longtext`, `email`, `phone`, `number`, `date`, `dropdown`, `checkbox` (blank = text)
- `options` (dropdown only): comma-separated, e.g. `BSIT, BSCS, BSCE`
- `required`, `is_default`, `active`: `TRUE` / `FALSE` (a blank `active` counts as TRUE)
- A data tab's header must contain `timestamp`, `event_id`, every `field_key` of that event, and `edited`. Columns not in the header are not written.

### 2. n8n: import the workflows

For each file in `n8n/` (`FS - Get Events`, `FS - Decode Form`, `FS - Save Record`):

1. In n8n: **Create workflow → ⋯ menu → Import from File** and pick the JSON. (If you already made empty placeholder workflows with these names, import into them or delete them first.)
2. Open every node with a credential warning and select your credential:
   - **Read Events**, **Read Fields**, **Append to Sheet** → your Google Sheets OAuth2 credential
   - **Gemini – Read Form** (Decode workflow only) → your Google Gemini credential
3. **Save**, then **Publish / Activate** the workflow. Production webhooks only work while the workflow is active.

Quick check after activating Get Events: open
`https://cmd-systems.app.n8n.cloud/webhook/fs-events` in a browser. You should see JSON with your event and its fields.

### 3. Cloudflare: deploy the web app

Push this repo (with `wrangler.jsonc` and `public/`) to `wlalng05/form-scanner-web`. Cloudflare's Git integration builds and deploys automatically.

- Build command: *(empty)*
- Deploy command: `npx wrangler deploy` (the default)
- Settings → Domains & Routes → **workers.dev: Enabled**

Then open `https://form-scanner-web.cmart-diaz.workers.dev` on your phone.

---

## Everyday changes

| I want to… | Do this |
|---|---|
| Add a field to an event | Add a row in **Fields**, add the same `field_key` as a column in the event's data tab. No code change. |
| Add an event | Add a row in **Events**, its rows in **Fields**, and create its data tab with the header row. |
| Change the default event | Set `is_default` to TRUE on one row (the app remembers a user's last choice on that phone). |
| Change the AI model | n8n → FS – Decode Form → **Build Gemini Request** → `MODEL` constant. Default `gemini-flash-latest`; try a Pro model if handwriting accuracy is poor. |
| Debug a workflow step by step | Open the workflow in n8n, click **Execute workflow** (listening for a test event), then open the app with `?test` at the end of the URL. The app then calls the test webhooks and you can see every node's data in n8n. |

## Turning on Header Auth (before go-live)

1. In each of the three **Webhook** nodes: Authentication → **Header Auth** → select `FS – App Token` (header name `X-App-Token`).
2. In `public/index.html`, set `CONFIG.APP_TOKEN` to the token value and push.

The token is visible to anyone who inspects the page. It stops bots and casual misuse, not a determined person; add a real login for production.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| "Workflow not found (404)" | The workflow isn't active, or the webhook path differs from `fs-events` / `fs-decode` / `fs-save`. |
| "Could not reach the server" on the phone, but the URL works in a desktop browser | CORS. Check each Webhook node → Options → **Allowed Origins (CORS)** is exactly `https://form-scanner-web.cmart-diaz.workers.dev` (no trailing slash). As a test, set it to `*`. |
| "AI request failed: API key not valid" | Re-check the Google Gemini credential in n8n. |
| "AI request failed … 429" / quota | Gemini free-tier rate limit. Wait a minute and retry. |
| Event list is empty | Event `active` is FALSE, or the event has no rows in **Fields**. The `/fs-events` JSON lists warnings. |
| Row saved but a column is blank | The data tab header doesn't match the `field_key` exactly (case and spelling). |
| Phone numbers lose the leading 0 | Shouldn't happen (values are written RAW). If you edited the Append node, set Options → Cell Format → **RAW**. |
