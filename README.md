# Slack2Clickup

A Slack bot that uses Claude to triage support requests and turn them into ClickUp tickets. It reads every message in a monitored channel, decides whether it can answer directly, needs more information, or needs a human — then replies in-thread or opens an assigned ClickUp task. No slash command required; users just describe their problem in plain language.

## How it works

1. A user posts a message in the monitored Slack channel — plain text, or with screenshots/files attached.
2. The bot sends the message to Claude along with a system prompt describing your team, routing rules, and known self-serve answers.
3. Claude classifies the message as one of:

   | Type | Behavior | Reaction |
   |---|---|---|
   | `self-serve` | Answers directly in the thread — no ticket created | ✅ |
   | `needs_info` | Asks one specific clarifying question, then resolves on the reply | — |
   | `ticket` | Creates an assigned ClickUp task, links back to the Slack thread | 📋 |
   | `ignore` | Not a support request (social chat, bot output) — no action | — |

4. Any attached files are downloaded from Slack and uploaded to the ClickUp task as real attachments.
5. Subsequent replies in a ticket's Slack thread are synced to the ClickUp task as comments, so the conversation stays in one place.

## Features

- **Zero-command triage** — no `/support` syntax to learn, just describe the issue
- **Configurable routing** — map issue categories to different assignees on your team
- **Clarifying questions** — the bot asks for the one missing detail instead of filing a vague ticket
- **Self-serve answers** — common how-to questions are answered directly, never becoming tickets
- **Multi-language** — replies in the requester's language; ticket text is normalized to English for the team
- **File attachments** — screenshots and documents are uploaded to the ClickUp task, not just referenced
- **Thread → comment sync** — follow-up discussion in Slack lands on the ClickUp task
- **Crash-resilient** — structured logging, API timeouts, top-level exception handlers, and ticket-thread state persisted across restarts
- **No public endpoint** — runs over Slack Socket Mode, so no inbound webhook or reverse proxy needed

## Prerequisites

- Node.js 18+
- A Slack workspace where you can install apps
- A [ClickUp](https://clickup.com/) account and API token
- An [Anthropic API key](https://console.anthropic.com/)

## Setup

### 1. Create the Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From an app manifest**, and paste in [`slack-app-manifest.json`](./slack-app-manifest.json) (adjust the name and description to taste).
2. **Socket Mode** → generate an app-level token with the `connections:write` scope. This is your `SLACK_APP_TOKEN`.
3. **OAuth & Permissions** → install the app to your workspace, then copy the Bot User OAuth Token (`xoxb-...`). This is your `SLACK_BOT_TOKEN`.
4. Invite the bot to the channel you want it to monitor: `/invite @YourBotName`.

### 2. Install and configure

```bash
git clone https://github.com/patrick112358/Slack2Clickup.git
cd Slack2Clickup
npm install
cp .env.example .env
```

Fill in `.env` with your Slack tokens, Anthropic API key, and ClickUp API token. All four are validated at startup — the process exits immediately if any are missing.

Then edit the constants near the top of [`index.js`](./index.js) for your workspace:

| Constant | What it is |
|---|---|
| `CHANNEL_ID` | The Slack channel the bot monitors |
| `CLICKUP_LIST_ID` | The ClickUp list new tickets are created in |
| `WORKSPACE_URL` | Your Slack workspace URL, used to build thread links |
| `CLICKUP_ASSIGNEES` | Maps team member names to their ClickUp user IDs |
| `SLACK_USERS` | Maps the same names to their Slack member IDs |

The keys in `CLICKUP_ASSIGNEES` and `SLACK_USERS` must match the assignee names used in the system prompt.

### 3. Customize the triage logic

[`system-prompt.js`](./system-prompt.js) is the bot's brain. Edit it to match your team:

- **Routing rules** — who handles which kind of issue
- **Categories and priorities** — what counts as urgent
- **Self-serviceable list** — questions the bot should answer instead of ticketing
- **Clarifying questions** — when to ask for one more detail before filing
- **Security boundaries** — things the bot must never answer directly (credentials, server access, privilege escalation), routed to a human instead

### 4. Run it

```bash
npm start
```

For a persistent Linux deployment, [`it-support-bot.service`](./it-support-bot.service) is a sample systemd unit with restart limits and timeouts already set. Update the `User` and path values, then:

```bash
sudo cp it-support-bot.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now it-support-bot
```

Logs go to the journal:

```bash
journalctl -u it-support-bot -f
```

## Operational notes

- Ticket-thread mappings are cached in `.ticket-threads.json` next to `index.js` so thread replies still sync to ClickUp after a restart. Entries expire after 7 days; the file is gitignored.
- Claude API calls have a 30-second hard timeout. On timeout or error the bot posts a fallback message in-thread pinging a human rather than failing silently.
- If ClickUp task creation fails, the bot pings the intended assignee in Slack so the request isn't lost.

## Updating

Behavior changes usually only need an edit to `system-prompt.js`. Logic changes go in `index.js`. Either way, restart the process:

```bash
git pull
sudo systemctl restart it-support-bot
```

## Tech stack

- [Bolt for JavaScript](https://slack.dev/bolt-js/) — Slack integration over Socket Mode
- [Anthropic SDK](https://github.com/anthropics/anthropic-sdk-typescript) — Claude classification
- [ClickUp API v2](https://developer.clickup.com/) — tickets, attachments, comments

## License

[MIT](./LICENSE) — free to use, modify, and distribute, including commercially. Keep the copyright notice; there is no warranty.

## Acknowledgements

Developed with the support of [Claude Code](https://claude.com/claude-code).
