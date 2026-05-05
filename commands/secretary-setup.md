---
description: Set up the virtual secretary (Gmail mode, Fio tokens, configuration)
thinking: enabled
---

Help the user set up their virtual secretary. Walk through each step interactively.

## Step 1: Create the secretary directory

Run this command to create the data directory:

```bash
mkdir -p ~/.secretary
```

## Step 2: Choose Gmail integration mode

The secretary needs to read and act on the user's Gmail inbox. There are two ways:

### Option A — Gmail MCP connector (simpler setup, limited capabilities)

The user enables the built-in Gmail connector in Claude (claude.ai or Claude Code → Settings → Connectors → Gmail). OAuth happens through Anthropic's flow — no Google Cloud project, no client secrets, just click "Connect".

**Limitations to disclose to the user before they choose this:**
- **Cannot send email autonomously.** The MCP exposes only `create_draft`, not `send_message`. Every reply the secretary writes will sit in Drafts until the user manually clicks Send. (Today the secretary auto-sends replies to known senders — this stops working.)
- **No bulk operations.** There is no `batchModify`. Trashing 50 newsletters means 50 separate tool calls — slower and noisier.
- **No direct attachment download.** PDF invoices can be listed but not pulled as raw bytes for parsing.
- **Cannot move to Trash directly.** Only label/unlabel operations are exposed; "delete spam" becomes "label as TRASH" if at all possible.
- **Per-session OAuth.** The user may need to reauthorize occasionally.
- **Vendor-controlled tool surface.** If Anthropic adds/removes MCP tools, the secretary's behaviour shifts without warning.

If the user picks MCP, save to `~/.secretary/config.json`:
```json
{ "gmailMode": "mcp" }
```

…and warn them: the `/secretary` skill currently codes against the direct Gmail API (Phase 2 uses `batchModify` for bulk archive/trash, autonomous `send_message` for replies, and direct attachment fetches). Switching to MCP mode requires reduced behaviour — the skill must fall back to drafts-only replies and per-message labeling. This adaptation has not yet been implemented; warn the user that picking MCP today means they must accept manual review of every action until the skill is updated.

### Option B — Direct Gmail API via OAuth (more setup, full power)

The user creates a Google Cloud project, enables the Gmail API, generates OAuth credentials, and runs an authorization flow once to obtain a refresh token. Detailed walkthrough is in `SETUP.md`. Required scopes: `gmail.modify` (read + label + trash + send) or the broader `gmail` scope.

**Capabilities:**
- Autonomous send of replies on the user's behalf (signed as Donna Claude)
- `batchModify` for bulk trash/archive (up to 1000 messages per call)
- Attachment download for PDF invoice parsing
- Refresh tokens — secretary handles expiry transparently
- No vendor-controlled tool surface

**End state of Option B:** `~/.gmail-mcp/credentials.json` containing `client_id`, `client_secret` (or via `~/.gmail-mcp/gcp-oauth.keys.json`), `access_token`, and `refresh_token`. The directory name `~/.gmail-mcp/` is historical — these credentials are used by direct API calls, not by an MCP server.

If the user picks API, save to `~/.secretary/config.json`:
```json
{ "gmailMode": "api" }
```

…and walk them through `SETUP.md` if `~/.gmail-mcp/credentials.json` is missing or has no `refresh_token`.

### Default recommendation

Recommend **Option B (API)** unless the user explicitly says they don't want to set up a Google Cloud project. The convenience of MCP is real, but the loss of autonomous send + bulk operations changes what the secretary can do for them — most of the value of this plugin comes from those two abilities.

## Step 3: Fio API tokens

The user needs to generate API tokens in Fio internetbanking:
1. Log into https://ib.fio.cz
2. Go to Nastaveni -> API
3. Click "Pridat novy token"
4. Select "Sledovani uctu a zadavani platebnich prikazu"
5. Authorize via SMS
6. Wait 5 minutes, then copy the token

Ask the user for:
- **Business account token** and account number (format: 123456/2010)
- **Personal account token** and account number

Save to `~/.secretary/fio-tokens.json`:
```json
{
  "business": {
    "token": "<user-provided-token>",
    "accountNumber": "<user-provided-account-number>"
  },
  "personal": {
    "token": "<user-provided-token>",
    "accountNumber": "<user-provided-account-number>"
  }
}
```

## Step 4: Other configuration

Merge into `~/.secretary/config.json` (preserving the `gmailMode` from Step 2):
```json
{
  "gmailMode": "<mcp or api, set in Step 2>",
  "autoPayLimit": 5000,
  "defaultAccount": "business",
  "dsAutoCheck": "monday"
}
```

Ask the user about each:
- `autoPayLimit` — CZK ceiling for trusted-vendor auto-pay (default 5000)
- `defaultAccount` — which Fio account is used for new vendors by default (default `business`)
- `dsAutoCheck` — when should `/secretary` check Datové schránky, archive new messages to Google Drive, and surface them in the summary. Three values:
  - `false` — never check DS automatically (manual invocation only)
  - `"monday"` — check only on Mondays (recommended for personal use; matches the typical weekly cadence of official correspondence)
  - `"always"` — check on every `/secretary` run (use only if you receive frequent ISDS messages)
  Default `"monday"`. Set to `false` if Step 9 is skipped or the user doesn't want auto-checking.

## Step 5: Initialize empty data files

```bash
echo '[]' > ~/.secretary/trusted_vendors.json
echo '[]' > ~/.secretary/pending_payments.json
```

## Step 6: Verify Gmail access

- **If `gmailMode` = `api`**: Test that `~/.gmail-mcp/credentials.json` works by calling `https://gmail.googleapis.com/gmail/v1/users/me/profile` with the access token. If the access token is expired, refresh it using the refresh token. If refresh fails, walk the user through `SETUP.md` again.
- **If `gmailMode` = `mcp`**: Confirm with the user that the Gmail connector shows as "Connected" in Claude's settings. Try a `list_labels` call via the MCP tool to verify it responds.

## Step 7: Verify Fio API

Test both Fio tokens. Use the periods endpoint (it tolerates a missing cursor, unlike `/last/`):
```bash
curl -s "https://fioapi.fio.cz/v1/rest/periods/<token>/<7-days-ago>/<today>/transactions.json"
```

Respect the **30-second per-token rate limit** — wait between the business and personal checks.

If either fails, help the user troubleshoot (wrong token, token not yet active after SMS, mistyped account number, etc.)

## Step 8: Optional — Slack integration

Ask the user: *"Do you want the secretary to handle small recurring micropayments posted to a Slack channel (e.g. team lunch payments)?"* If they say no, skip to Step 9.

If yes, explain what's involved before continuing — Slack does **not** offer a normal API token for personal accounts. The integration uses the user's browser session (xoxc user token + xoxd cookie). This is functionally Martin's full Slack login captured as a string. **Disclose plainly:**
- The captured pair grants the same powers as Martin's logged-in browser (read every channel and DM, post messages, react). Anyone with the file can impersonate him on Slack until he logs out.
- Tokens **expire on logout** — if Martin signs out of Slack, the secretary will fail until he repeats the extraction.
- Tokens **must be extracted from a real browser** with developer tools — not from the Slack desktop client.

Walk through the extraction:
1. Open https://app.slack.com in a Chromium-based browser (Chrome / Edge / Brave). Make sure you're logged into the workspace the secretary should monitor.
2. Open DevTools (Cmd-Opt-I / F12) → **Network** tab → filter for `client.boot`.
3. Reload the page. Click the `client.boot` request.
4. **Headers** sub-tab → scroll to **Form Data** (or **Payload**) → copy the `token` field starting with `xoxc-…` — this is `xoxc_token`.
5. **Headers** sub-tab → **Request Headers** → find `Cookie:` → copy the value of the `d=…` segment (just the value, without `d=`) — this is `xoxd_cookie`.
6. Workspace + channel + user IDs: `workspace_id` from the URL prefix `T0…`, `channel_id` from the channel URL `C0…`, `user_id` from one's own profile (`U0…`).

Ask for the `payment_account` (where micropayments go — format `prefix-account/bank`) and `vendor_name` (the human label for `trusted_vendors.json`).

Save to `~/.secretary/slack-token.json` (chmod 600):
```json
{
  "xoxc_token": "xoxc-...",
  "xoxd_cookie": "...",
  "workspace_id": "T...",
  "workspace_domain": "<workspace>",
  "channel_id": "C...",
  "channel_name": "<channel>",
  "user_id": "U...",
  "payment_account": "prefix-account/bank",
  "vendor_name": "<recurring recipient>",
  "auth_method": "xoxc_user_token"
}
```

```bash
chmod 600 ~/.secretary/slack-token.json
```

Verify by calling `auth.test` (must succeed) and `conversations.history` for the configured channel:
```bash
curl -s "https://slack.com/api/auth.test" \
  -H "Authorization: Bearer $(jq -r .xoxc_token ~/.secretary/slack-token.json)" \
  -H "Cookie: d=$(jq -r .xoxd_cookie ~/.secretary/slack-token.json)" \
  -H "User-Agent: Mozilla/5.0"
```
If `auth.test` returns `{"ok": true}` the integration is live. Tell the user that any Slack logout invalidates it.

## Step 9: Optional — Datová schránka (ISDS) integration

Ask the user: *"Do you have one or more Datové schránky (Czech official mailboxes) you'd like the secretary to monitor and act on?"* If no, skip to Step 10.

If yes, explain capabilities and constraints:
- The secretary can list incoming messages, download zfo + attachments, and send official messages on the user's behalf.
- Authentication is **username + password** for each box. Production endpoint is `https://ws1.mojedatovaschranka.cz/`. There is also a separate operations endpoint `/DsManage/` for some calls.
- ISDS issues a **5-attempt lockout** on failed logins, then a forced password reset. Type carefully.
- Messages received via ISDS are legally delivered after **10 days regardless of whether the user reads them** — the secretary's monitoring role is genuinely useful.

Ask the user for each box they own (typically `personal` and/or `business`):
- `username` (login provided by Ministerstvo vnitra, typically a 6-character alphanumeric)
- `password`
- `label` — friendly name (e.g. `"Osobní DS"`, `"OSVČ DS"`)
- `dsId` — optional 7-character box ID (`xxxxxxx`), needed for sending; can be looked up later via API if not known yet

Save to `~/.secretary/isds-credentials.json` (chmod 600):
```json
{
  "personal": {
    "username": "...",
    "password": "...",
    "label": "Osobní DS",
    "dsId": "..."
  },
  "business": {
    "username": "...",
    "password": "...",
    "label": "OSVČ DS",
    "dsId": "..."
  },
  "endpoint": "https://ws1.mojedatovaschranka.cz/",
  "operations_endpoint": "https://ws1.mojedatovaschranka.cz/DsManage/"
}
```

```bash
chmod 600 ~/.secretary/isds-credentials.json
```

Verify by calling `GetListOfReceivedMessages` (or equivalent SOAP method) for each configured box. If a call returns 401, the credentials are wrong; help the user check before they hit the lockout.

After verification succeeds, confirm `dsAutoCheck` setting in `~/.secretary/config.json` (set in Step 4):
- `"monday"` — recommended; secretary checks DS only on Mondays, archives novelties to Google Drive, surfaces them in the summary
- `"always"` — every run
- `false` — never automatically; user invokes DS checks manually

The `/secretary` skill's Phase 2.7 reads this flag and the current weekday to decide whether to run.

If `dsAutoCheck` is enabled, also ensure Google Drive access works (the secretary needs to write archived files there). The Google Drive MCP connector should show as Connected; if not, help the user reconnect.

## Step 10: Summary

Print a summary of what was configured:
- Gmail mode (api/mcp), validated yes/no
- Both Fio accounts validated, balances current
- Trusted vendors and pending payments — empty arrays
- Slack integration — configured / skipped
- ISDS integration — configured (which boxes) / skipped, with `dsAutoCheck` on/off

Suggest running `/secretary` to do a first inbox pass.

If `gmailMode` is `mcp`, remind the user one more time that replies will land in Drafts (not auto-sent) and bulk triage will be slower until the `/secretary` skill is updated to handle MCP mode natively.
