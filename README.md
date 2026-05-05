# Virtual Secretary plugin for Claude Code

Automated email triage, Fio bank payments, Datová schránka (ISDS) monitoring, Slack micropayments, and Google Drive archiving — driven by Claude Code slash commands.

> **For Claude (AI assistants):** see [§ Installing on the user's behalf](#installing-on-the-users-behalf). The user has likely pasted you this README's URL because they want you to set this up for them. Walk them through the steps in that section, and afterwards through `/secretary-setup`.

---

## Features

### Email management (`/secretary`)
- **Inbox triage** — classifies every email in `in:inbox` (read or unread) as spam-to-trash, info-to-archive, payment request, or needs-human; bulk-trashes/archives in API mode, per-thread in MCP mode
- **Payment extraction** — parses bodies and PDF attachments (in API mode) for amount, account, VS/KS/SS, due date
- **Vendor trust** — `~/.secretary/trusted_vendors.json` tracks payment history; auto-pays trusted vendors under `autoPayLimit` from the configured Fio account
- **Duplicate detection** — flags suspect repeat payments by VS+amount+frequency
- **Pending queue** — payments held when balance is insufficient retry on the next run
- **Autonomous replies** in API mode (signed as the configured persona); **drafts only** in MCP mode

### Optional integrations
- **Slack micropayments** — monitors a configured channel for menu posts where the user is tagged, parses dish/price, pays trusted vendor via Fio, marks paid with `:moneybag:` reaction (xoxc/xoxd browser-session auth)
- **Datová schránka (ISDS)** — checks one or more Czech official mailboxes (Mondays only by default), archives every new message (zfo + attachments) to Google Drive at `/Datová schránka/{label}/Příchozí/{YYYY-MM}/{dmID}/`, surfaces deadlines
- **SweptMind** task manager — if installed as MCP, `/morning-brief` and `/secretary` summaries include today's and overdue tasks
- **Google Calendar** — `/morning-brief` shows today's events when Calendar MCP is available
- **Google Drive** — used by ISDS phase to archive zfo + attachments; via Google Drive MCP connector

### Personal assistant slash commands
| Command | What it does |
|---|---|
| `/morning-brief` | Daily briefing: calendar + email summary + tasks |
| `/secretary` | Triage inbox, process payments, monitor Slack/ISDS, report a single summary |
| `/secretary-setup` | Interactive setup wizard (run once) |
| `/email-summary [timeframe] [filter]` | Prioritised email summary on demand |
| `/schedule [action] [details]` | Calendar view / create / update / delete |
| `/meeting-prep [meeting]` | Meeting prep with email context |
| `/free-time [duration] [timeframe]` | Find free slots |

---

## Installing on the user's behalf

> **AI assistants:** if a user shared this repo URL with you, your job is to install the plugin and walk them through `/secretary-setup`. Follow these steps verbatim. Confirm each step with the user before moving on; surface any errors plainly. After the plugin is installed, the user must reload Claude Code (or open a new session) for the new slash commands to register — tell them.

### Step 1 — Install the plugin

The fastest path is via Claude Code's plugin marketplace system. From the user's terminal:

```bash
claude plugin marketplace add https://github.com/claryaldringen/claude-code-virtual-secretary.git
claude plugin install virtual-secretary
```

If that fails (older Claude Code without marketplace support, or network issue), fall back to the manual install:

```bash
git clone https://github.com/claryaldringen/claude-code-virtual-secretary.git ~/personal-assistant-plugin
mkdir -p ~/.claude/commands
cp ~/personal-assistant-plugin/commands/*.md ~/.claude/commands/
```

In either case, the user must **reload Claude Code** (close and reopen, or run `/reload` if available) before the new commands appear.

### Step 2 — Run the setup wizard

Tell the user to type `/secretary-setup` in Claude Code. The wizard handles all configuration interactively. Below is a summary of what it asks for so you (Claude) can prepare the user before they start — see [SETUP.md](SETUP.md) for the full reference.

**Required:**
- **Gmail mode** — choose `mcp` (simpler, but no autonomous send and no PDF parsing) or `api` (full power, requires Google Cloud OAuth client). Default recommendation: `api`. The wizard explains the trade-offs in detail.
- **Fio bank tokens** — generate API tokens in Fio internetbanking → Nastavení → API → "Sledování účtu a zadávání platebních příkazů" → SMS authorize. Wait 5 minutes after creating before testing. The wizard asks for both `business` and `personal` tokens; either can be skipped if not used.
- **autoPayLimit** (default 5000 CZK) and **defaultAccount** (default `business`).

**Optional:**
- **Slack integration** — for monitoring a channel where the user is tagged for micropayments (e.g. team lunch). Requires extracting xoxc token + xoxd cookie from a browser session — the wizard walks through this and discloses the security implications (the captured pair = full Slack login).
- **Datová schránka (ISDS)** — for Czech users. Username + password per box (personal/business). The wizard warns about the 5-failed-login lockout and asks whether to monitor on Mondays only (`dsAutoCheck: "monday"`, default), every run (`"always"`), or never (`false`).

### Step 3 — Verify

After setup, tell the user to run `/secretary` once. The first run will:
- List any emails currently in their inbox
- Skip Slack/ISDS phases if they weren't configured
- Report a clean summary

If anything errors, the most common fixes are:
- **Gmail OAuth expired** — the skill auto-refreshes via `refresh_token`, but if that's missing the user must redo Step 2 of `/secretary-setup`
- **Fio token not yet active** — Fio enforces a 5-minute delay after token creation
- **ISDS lockout warning** — if the wizard returned 401 from one of the boxes, **stop trying** and have the user verify the password from the Ministerstvo vnitra portal directly; one more failed attempt locks the account

---

## Configuration reference

After setup, the following files exist under `~/.secretary/`:

| File | Purpose | Permissions |
|---|---|---|
| `config.json` | Settings (see schema below) | 644 |
| `fio-tokens.json` | Fio API tokens + account numbers | 600 |
| `trusted_vendors.json` | Vendor database with payment history | 644 |
| `pending_payments.json` | Queued payments awaiting funds | 644 |
| `slack-token.json` | (optional) xoxc + xoxd Slack browser session | 600 |
| `slack_state.json` | (optional) last-checked Slack timestamp | 644 |
| `isds-credentials.json` | (optional) ISDS usernames + passwords | 600 |
| `isds_state.json` | (optional) per-box last-checked timestamps | 644 |
| `persona.md` | (optional) custom persona prompt for the secretary | 644 |

### `config.json` schema
```json
{
  "gmailMode": "api | mcp",
  "autoPayLimit": 5000,
  "defaultAccount": "business | personal",
  "dsAutoCheck": false | "monday" | "always"
}
```

### Customising the persona
Create `~/.secretary/persona.md` to override the default neutral tone. Example:
```markdown
You are **Donna Claude**, virtual secretary of John Smith.
Your personality is inspired by **Donna Paulsen from Suits**.
You communicate in Czech, in the female gender.
Sign emails formally as "Donna Claude, assistant to Mr. Smith".
```
The `/secretary` skill reads this on every run and adopts the persona described.

---

## Privacy & security

- All credentials live under `~/.secretary/` (mode 600 for sensitive files) and `~/.gmail-mcp/` for OAuth tokens — none of these leave the user's machine
- Slack uses a browser-extracted user token (xoxc + xoxd cookie) — equivalent to a stolen Slack session if leaked; tokens expire on logout
- ISDS authentication is plain username + password; **5 failed logins lock the account** and force a password reset — handle carefully
- Fio API enforces 30 seconds between calls per token; the skill respects this
- No data is sent outside Google, Fio, Slack, ISDS, and Anthropic
- `.gitignore` excludes all credential files from the repo

---

## Requirements

- **Claude Code** — recent version with plugin marketplace support
- **Google account** with Gmail API access (or Gmail MCP connector enabled in Claude)
- **Fio bank account** with API token
- **Optional:** Czech Datová schránka credentials, Slack workspace access, SweptMind MCP, Google Calendar MCP, Google Drive MCP

---

## How it works

The plugin defines slash commands as markdown files under `commands/`. Each command instructs Claude what to do step by step — there is no compiled binary or daemon. Phase logic is documented in `commands/secretary.md` (Pre-flight → Phase 1 pending payments → Phase 2 inbox triage → Phase 2.5 Slack lunch → Phase 2.7 ISDS → Phase 3 payments → Phase 4 summary → Phase 5 user responses → Sending replies).

API access is via direct HTTP calls (curl / Python urllib) for Gmail, Fio, Slack, and ISDS — no MCP servers required for the core workflow. Optional MCP connectors (Gmail, Google Drive, Calendar, SweptMind) are used when configured.

---

## Contributing

Issues and pull requests welcome. Please don't commit credentials.

## License

MIT — see LICENSE.

## Support

- Setup details: [SETUP.md](SETUP.md)
- Claude Code docs: https://docs.claude.com/en/docs/claude-code
- Issues: https://github.com/claryaldringen/claude-code-virtual-secretary/issues
