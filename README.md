# n8n Workflows

A small collection of [n8n](https://n8n.io) workflows for personal finance automation, self-hosted on a home server.

> **Security:** all workflows in this repo are scrubbed. No real credentials, tokens, account IDs, or private URLs are included. Placeholders (`<ACCOUNT_ID>`, `<BRIDGE_URL>`, etc.) must be replaced with your own values after import.

## Workflows

| Workflow | File | Description |
|---|---|---|
| Maybank Statement → Actual Budget | [`workflows/maybank-pdf-to-actual-budget.json`](workflows/maybank-pdf-to-actual-budget.json) | Polls Gmail for Maybank statement emails, downloads the PDF attachment, extracts transactions via a local bridge service, and imports them into Actual Budget (deduplicated). |

---

## Maybank Statement PDF → Actual Budget auto-import

Automatically imports your monthly Maybank e-statement (sent by email as a password-protected PDF) into [Actual Budget](https://actualbudget.org/), so your budget always reflects your real bank balance.

### How it works

```
┌─────────────────┐     ┌──────────────────────┐     ┌──────────────────────────┐
│  Gmail Trigger   │────▶│  Prepare PDF Payload  │────▶│  Extract via Bridge       │
│  (poll, every    │     │  (Code node)          │     │  (HTTP POST /extract)     │
│   5 min)         │     │  binary → base64      │     │  PDF → text → JSON txns   │
└─────────────────┘     └──────────────────────┘     └────────────┬─────────────┘
                                                                   │
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────▼─────────────┐
│  Summary         │◀────│  Push to Actual       │◀────│  Map to Actual Format     │
│  (import result) │     │  (HTTP POST           │     │  (Code node)              │
│                  │     │   /transactions)      │     │  RM float → integer sen  │
└─────────────────┘     └──────────────────────┘     └───────────────────────────┘
```

**Data flow, step by step:**

1. **Gmail Trigger** (polling) — checks Gmail every 5 minutes for messages matching the Gmail search query `from:m2u@stmts.maybank2u.com.my subject:Statement has:attachment` (Maybank's e-statement sender). Because `simple: false` is set, the trigger downloads attachments into binary data.
2. **Prepare PDF Payload** (Code node) — reads the PDF attachment from the item's binary data and converts it to base64. Outputs `{ pdf_base64, bank: 'maybank', accountId: '<ACCOUNT_ID>', filename }`.
3. **Extract via Bridge** (HTTP Request) — `POST <BRIDGE_URL>/extract` with `{ pdf_base64, bank }`. The bridge service (a small Node/Express app using `@actual-app/api`; not part of this repo) decrypts the PDF, converts it to text, parses transactions, and returns `{ account_name, opening_balance, closing_balance, transactions: [{date, description, amount, balance_after}] }`. Authenticated with the `x-bridge-token` header via a **Header Auth** credential.
4. **Map to Actual Format** (Code node) — converts amounts from RM float to **integer sen** (Actual's format; negative = expense), builds the `imported_id` dedup key (`date-amount-description`), and maps `description → payee_name`.
5. **Push to Actual** (HTTP Request) — `POST <BRIDGE_URL>/transactions` with `{ accountId, transactions }`. The bridge calls `importTransactions()` — Actual deduplicates via `imported_id`, so re-running the workflow never creates duplicate transactions.
6. **Summary** (Code node) — logs the import result (`added`, `updated`, `errors`).

### Prerequisites

- An **n8n** instance (tested on n8n 2.x, self-hosted)
- A **Gmail account** with Maybank e-statements enabled (`m2u@stmts.maybank2u.com.my` sender)
- A **bridge service** exposing:
  - `POST /extract` — `{ pdf_base64, bank }` → `{ transactions: [...] }`
  - `POST /transactions` — `{ accountId, transactions: [...] }` → `{ added, updated, errors }`
  - Auth: `x-bridge-token` header on all endpoints (except `/health`)
- An **Actual Budget** server reachable by the bridge

### Setup

#### 1. Import the workflow

1. Open your n8n editor.
2. **Workflows → ⋯ (menu) → Import from File**.
3. Select `workflows/maybank-pdf-to-actual-budget.json`.
4. The workflow will show with missing credentials — link them in the next step.

#### 2. Create and link credentials

The workflow references two credentials by placeholder ID. Create both, then open each node and select the matching credential:

| Credential type | Used by node(s) | Notes |
|---|---|---|
| **Gmail OAuth2 API** | `Gmail Trigger1` | Create a Google Cloud OAuth client with the **Gmail API** enabled (see "Gmail OAuth setup" below). |
| **Header Auth** | `Extract via Bridge`, `Push to Actual` | Header **Name** = `x-bridge-token`, **Value** = your bridge token (the same value configured as `BRIDGE_TOKEN` in the bridge's environment). |

> ⚠️ The credential `id` placeholders (`<GMAIL_OAUTH_CREDENTIAL_ID>`, `<HEADER_AUTH_CREDENTIAL_ID>`) are intentionally fake — after import, n8n will show the credential as unlinked. Just open each node, choose the credential you created, and save.

#### 3. Replace the placeholders

Open the Code and HTTP nodes and replace:

| Placeholder | Where | What to put |
|---|---|---|
| `<ACCOUNT_ID>` | `Prepare PDF Payload`, `Map to Actual Format` | Your Actual Budget account ID for the Maybank account (get it from the bridge `GET /accounts` endpoint, or via the Actual web UI). |
| `<BRIDGE_URL>` | `Extract via Bridge`, `Push to Actual` | The bridge's base URL, e.g. `http://<server-ip>:5010` or `https://bridge.example.com` (no trailing slash). |
| `<GMAIL_OAUTH_CREDENTIAL_ID>` / `<HEADER_AUTH_CREDENTIAL_ID>` | node `credentials` | Filled automatically when you link credentials in the UI (step 2). |

#### 4. Activate the trigger

1. Open **Gmail Trigger1** and confirm the poll schedule (default: every 5 minutes).
2. Toggle the workflow to **Active** (top-right switch).

That's it — when Maybank's statement email arrives, the workflow imports it within ~5 minutes. The `imported_id` dedup key ensures re-imports don't duplicate transactions.

### Known issues & pitfalls (learned the hard way)

These were all real failures encountered while building and debugging this workflow on n8n 2.33.5:

1. **`$env` access is blocked in nodes (by default).** Using `={{ $env.BRIDGE_TOKEN }}` in an HTTP header fails with `access to env vars denied` unless you set both `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` and `NODE_FUNCTION_ALLOW_ENV=true` in the n8n container environment (restart required). **Best practice: use a Header Auth credential instead** — that's what this workflow does. It keeps secrets in n8n's encrypted credential store, not in the workflow.
2. **Gmail Trigger has no `typeVersion: 2`.** The installed node supports only versions `1`–`1.4`. Importing JSON with `typeVersion: 2` makes the node appear broken/red. This file uses `typeVersion: 1.4`.
3. **`simple: false` is required to download attachments.** With `simple: true` (default) the trigger returns metadata only — attachments are never downloaded.
4. **`simple: false` changes the output shape.** Subject/from move into `payload.headers` (an array). A Filter node comparing `$json.subject` fails with a type error and silently drops all items. Fix: rely on the Gmail `q` filter in the trigger (as this workflow does) and don't add a Filter node.
5. **`$binary` is NOT valid in Code node v2** — it evaluates to `undefined`. Iterate `$input.all()` and read `item.binary` instead.
6. **Binary data "filesystem-v2" trap.** In n8n 2.x, `item.binary.attachment_0.data` returns the literal string `"filesystem-v2"` (a filesystem pointer), not base64. You MUST use `await this.helpers.getBinaryDataBuffer(i, key)` to get the real bytes — this workflow does exactly that.
7. **`require('fs')` is banned in Code nodes** (sandboxed task runner). Do file I/O in the bridge service, not in n8n.
8. **OAuth redirect URL stuck on localhost.** n8n builds the Google OAuth callback from `http://localhost:5678` unless it knows its public URL. Set `WEBHOOK_URL`, `N8N_EDITOR_BASE_URL`, `N8N_HOST`, `N8N_PROTOCOL`, `N8N_PROXY_HOPS` in the n8n environment and restart; Google OAuth requires HTTPS (e.g. via Tailscale Serve or a reverse proxy).

### Gmail OAuth setup (one-time)

1. Google Cloud Console → create a project (or reuse one) → **APIs & Services → Library → enable Gmail API**.
2. **OAuth consent screen**: External + Testing; add your Gmail address under **Test users**.
3. **Credentials → Create Credentials → OAuth client ID → Web application**.
4. Open the n8n credential (type **Gmail OAuth2 API**) *first* to see its **OAuth Redirect URL** (e.g. `https://<your-n8n-host>/rest/oauth2-credential/callback`), and paste it exactly into Google's **Authorized redirect URIs**.
5. Paste Client ID + Client Secret back into n8n and click **Sign in with Google**.
6. Testing-mode tokens expire after 7 days — reconnect in the credential modal when needed.

### Bridge API contract (reference)

```
POST /extract
  headers: x-bridge-token: <token>
  body:    { "pdf_base64": "<base64>", "bank": "maybank", "password": "<optional>" }
  → 200:   { "account_name": "...", "opening_balance": 123.45,
             "closing_balance": 67.89,
             "transactions": [ { "date": "2026-06-01", "description": "...",
                                 "amount": -50.00, "balance_after": 71.23 } ] }

POST /transactions
  headers: x-bridge-token: <token>
  body:    { "accountId": "<uuid>",
             "transactions": [ { "date": "2026-06-01", "amount": -5000,
                                 "payee_name": "...", "imported_id": "..." } ] }
  → 200:   { "added": [...], "updated": [...], "errors": [...] }
```

Notes:
- `amount` in `/transactions` is **integer sen** (RM 1.00 → 100), negative = expense.
- `imported_id` is the dedup key — Actual's `importTransactions()` skips IDs it has already seen.

## License

[MIT](LICENSE)
