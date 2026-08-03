# Slack IT Support Bot

A Slack bot that uses Claude to triage support requests in a channel. It reads every message, decides whether it can answer directly or needs a human, and either replies in-thread or opens a ticket in ClickUp — automatically, no slash command required.

## How it works

1. A user posts a message in a monitored Slack channel (plain text, or with an image/file attached).
2. The bot sends the message to Claude along with a system prompt describing your team, routing rules, and known self-serve answers.
3. Claude classifies the message as one of:
   - **self-serve** — answered directly in the thread (reacts ✅)
   - **ticket** — a ClickUp task is created and assigned, with a link back to the Slack thread (reacts 📋)
   - **ignore** — not a support request (social chat, replies, bot messages), no action taken
4. Messages in languages other than English are handled natively; ticket titles/descriptions are normalized to English for the team, while direct replies stay in the requester's language.

## Features

- Zero-command triage — just describe the issue in the channel
- Configurable routing to different assignees based on issue category
- Configurable self-serve knowledge base for common how-to questions
- Multi-language support out of the box
- Image/file attachments are noted in the ticket with a link back to the original Slack thread
- Runs over Slack Socket Mode — no public webhook endpoint needed

## Prerequisites

- Node.js 18+
- A Slack workspace where you can install apps
- A [ClickUp](https://clickup.com/) account and API token
- An [Anthropic API key](https://console.anthropic.com/)

## Setup

### 1. Create the Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From an app manifest**, and paste in [`slack-app-manifest.json`](./slack-app-manifest.json) (adjust the name/description if you like).
2. **Socket Mode** → generate an app-level token with the `connections:write` scope. This is your `SLACK_APP_TOKEN`.
3. **OAuth & Permissions** → install the app to your workspace, then copy the Bot User OAuth Token (`xoxb-...`). This is your `SLACK_BOT_TOKEN`.
4. Invite the bot to the channel you want it to monitor: `/invite @YourBotName`.

### 2. Configure the bot

Clone the repo and install dependencies:

```bash
git clone https://github.com/patrick112358/CCslackSupport.git
cd CCslackSupport
npm install
cp .env.example .env
```

Fill in `.env` with your Slack tokens, Anthropic API key, and ClickUp API token.

Then edit the constants at the top of [`index.js`](./index.js) for your workspace:

- `CHANNEL_ID` — the Slack channel to monitor
- `CLICKUP_LIST_ID` — the ClickUp list tickets get created in
- `WORKSPACE_URL` — your Slack workspace URL, used to build thread links
- `CLICKUP_ASSIGNEES` / `SLACK_USERS` — map your team's names to their ClickUp user IDs and Slack member IDs

### 3. Customize the triage logic

[`system-prompt.js`](./system-prompt.js) is the bot's brain — edit it to match your team:

- Routing rules (who gets what category of issue)
- Categories and priority rules
- What counts as self-serviceable vs. ticket-worthy
- Anything the bot should refuse to answer directly (credentials, admin access, etc.)

### 4. Run it

```bash
npm start
```

For a persistent deployment on Linux, [`it-support-bot.service`](./it-support-bot.service) is a sample systemd unit — copy it to `/etc/systemd/system/`, update the paths/user, then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now it-support-bot
```

## Updating

Edit `system-prompt.js` to change behavior, or `index.js` for logic changes, then restart the process (`systemctl restart it-support-bot` if running as a service).

## Tech stack

- [Bolt for JavaScript](https://slack.dev/bolt-js/) (Socket Mode)
- [Anthropic SDK](https://github.com/anthropics/anthropic-sdk-typescript) (Claude)
- [ClickUp API](https://developer.clickup.com/) for ticketing
