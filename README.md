# Personal Assistant Plugin for Claude Code

Transform Claude Code into your intelligent personal assistant with seamless Gmail and Google Calendar integration. This plugin helps you manage emails, coordinate meetings, process payments, and stay organized throughout your day with virtual secretary capabilities.

## Features

### Email Management
- **Smart Email Summaries**: Get prioritized summaries of your inbox
- **Context-Aware Responses**: Draft responses based on email thread context
- **Advanced Search**: Find emails using natural language queries
- **Priority Detection**: Automatically identifies urgent and important messages

### Calendar Management
- **Schedule Overview**: View your daily, weekly schedule at a glance
- **Smart Scheduling**: Find optimal time slots for meetings and focus work
- **Meeting Preparation**: Get context and prep materials for upcoming meetings
- **Conflict Detection**: Automatically identifies scheduling conflicts

### Virtual Secretary
- **Inbox Triage**: Automatically classifies emails - deletes spam/commercial, archives informational
- **Payment Processing**: Extracts invoice data, verifies vendors, submits payments via Fio API
- **Vendor Trust**: Builds trust database from payment history - auto-pays trusted vendors under limit
- **Smart Duplicate Detection**: Checks payment history and frequency to prevent double payments
- **Pending Payments**: Queues payments when insufficient funds, retries on next run
- **Dual Account Support**: Manages both business and personal Fio bank accounts

### Personal Assistant Commands

#### `/morning-brief`
Start your day right with a comprehensive morning briefing:
- Today's calendar events and schedule
- Important email summaries
- Action items requiring attention
- Recommendations for optimal time management

#### `/schedule [action] [details]`
Manage your calendar effortlessly:
- View schedules by day, week, or date range
- Create new events with natural language
- Update or delete existing events
- Check for conflicts

#### `/email-summary [timeframe] [filter]`
Triage your inbox intelligently:
- Prioritized email summaries (HIGH/MEDIUM/LOW)
- Filter by sender, subject, or keywords
- Actionable recommendations
- Suggested response approaches

#### `/meeting-prep [meeting]`
Never go into a meeting unprepared:
- Meeting details and attendees
- Relevant email context
- Recent communication history
- Talking points and recommendations

#### `/free-time [duration] [timeframe] [preferences]`
Find the perfect time for any task:
- Discover available time slots
- Smart recommendations for focus time
- Respects your scheduling preferences
- Avoids fragmentation

#### `/secretary`
Process your inbox like a virtual secretary:
- Triage unread emails (delete spam, archive processed)
- Extract and verify payment requests
- Auto-pay trusted vendors under 5,000 CZK limit
- Queue payments when funds are insufficient
- Report summary with items needing your decision

#### `/secretary-setup`
First-time setup wizard:
- Configure Fio bank API tokens
- Set auto-pay limits
- Initialize data files

## Quick Start

1. **Clone the plugin**:
   ```bash
   git clone https://github.com/claryaldringen/claude-code-virtual-secretary.git
   ```

2. **Copy commands to your Claude Code config**:
   ```bash
   cp claude-code-virtual-secretary/commands/* ~/.claude/commands/
   ```

3. **Set up Google Cloud credentials** (see [SETUP.md](SETUP.md))

4. **Run the setup wizard**:
   ```
   /secretary-setup
   ```

5. **Start using**:
   ```
   /morning-brief
   /secretary
   ```

## Setup

See [SETUP.md](SETUP.md) for detailed installation and configuration instructions.

## Requirements

- Claude Code
- Google account with Gmail API access (OAuth2 credentials at `~/.gmail-mcp/`)
- Google Cloud Platform account (free tier)
- Fio bank account(s) with API tokens

### Optional

- **SweptMind** (GTD task manager) — if configured as an MCP server, `/morning-brief` will include task overview. Install via `claude mcp add --scope user sweptmind npx sweptmind-cli serve`
- **Google Calendar MCP** — for calendar integration in `/morning-brief`

## Customizing the Persona

By default, the secretary uses a neutral, professional tone. To customize:

Create `~/.secretary/persona.md` with your preferred personality:

```markdown
You are **Donna Claude**, virtual secretary of John Smith.
Your personality is inspired by **Donna Paulsen from Suits**.
You communicate in Czech, in the female gender.
Sign emails formally as "Donna Claude, assistant to Mr. Smith".
```

The secretary will read this file on startup and adopt the persona described there.

## How It Works

The plugin uses direct API calls (Gmail REST API, Fio API) rather than MCP servers for core functionality. This means fewer dependencies and simpler setup. MCP servers are optional for extended features like task management (SweptMind) and calendar (Google Calendar).

## Privacy & Security

- OAuth 2.0 authentication with token storage at `~/.gmail-mcp/`
- No data is stored or transmitted outside of Google, Fio, and Claude
- Minimal required permissions (read/modify emails, manage calendar)
- All operations require explicit user consent
- Sensitive files (tokens, credentials) are excluded via `.gitignore`

## Example Usage

**Morning routine**:
```
/morning-brief
```

**During the day**:
```
What emails do I need to respond to?
/meeting-prep next
Show me my afternoon schedule
```

**Scheduling**:
```
/free-time 2hours tomorrow afternoon
Schedule a meeting with John next week for 1 hour
```

**End of day**:
```
/email-summary today
What do I have tomorrow morning?
```

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

MIT License - See LICENSE file for details

## Support

For issues and questions:
- Check [SETUP.md](SETUP.md) for troubleshooting
- Review [Claude Code documentation](https://docs.claude.com/en/docs/claude-code)
- File an issue in this repository