<img src="https://github.com/chatwoot/chatwoot/raw/develop/.github/screenshots/header.png#gh-light-mode-only" width="100%" alt="Header light mode"/>
<img src="https://github.com/chatwoot/chatwoot/raw/develop/.github/screenshots/header-dark.png#gh-dark-mode-only" width="100%" alt="Header dark mode"/>

___

# Chatwoot — Unlocked Edition

The modern customer support platform, an open-source alternative to Intercom, Zendesk, Salesforce Service Cloud etc. — with every paid-plan feature gate removed for self-hosted use.

<p>
  <a href="https://github.com/engacs/chatwoot/tree/master-unlocked">
    <img src="https://img.shields.io/badge/branch-master--unlocked-blue" alt="Branch: master-unlocked"/>
  </a>
  <a href="https://hub.docker.com/r/engacs/chatwoot/tags">
    <img src="https://img.shields.io/docker/pulls/engacs/chatwoot" alt="Docker Pulls"/>
  </a>
  <a href="https://hub.docker.com/r/engacs/chatwoot/tags">
    <img src="https://img.shields.io/docker/v/engacs/chatwoot?label=docker%20image&color=green" alt="Docker Image"/>
  </a>
  <img src="https://img.shields.io/badge/version-4.17.0--unlocked-brightgreen" alt="Version"/>
  <img src="https://img.shields.io/github/commit-activity/m/engacs/chatwoot" alt="Commits-per-month">
  <a href="https://discord.gg/cJXdrwS"><img src="https://img.shields.io/discord/647412545203994635" alt="Discord"></a>
</p>

---

> **⚠️ Notice about this fork**
>
> This is a community fork of [Chatwoot](https://github.com/chatwoot/chatwoot) based on version **4.17.0**.
>
> The original Chatwoot codebase restricts certain features (Captain AI, SLA, Custom Roles, Audit Logs, Advanced Assignment, etc.) behind paid Enterprise/Cloud plans. We understand and respect the original team's licensing model — they have worked hard to build an amazing product.
>
> However, for our self-hosted use case we needed all features available without plan restrictions. This fork removes those gates so everything works out of the box on a self-hosted installation.
>
> **Docker image:** [`engacs/chatwoot:latest`](https://hub.docker.com/r/engacs/chatwoot/tags) (rolling, tracks the newest version tag) · [`engacs/chatwoot:v4.17.0-unlocked`](https://hub.docker.com/r/engacs/chatwoot/tags) (pinned) · [`engacs/chatwoot:unlocked`](https://hub.docker.com/r/engacs/chatwoot/tags) (legacy rolling tag, kept for existing installs)
>
> **This repository hosts documentation and setup instructions only.** The `enterprise/` portion of Chatwoot is licensed separately from the rest of the codebase (see [`enterprise/LICENSE`](https://github.com/chatwoot/chatwoot/blob/develop/enterprise/LICENSE) in the upstream repo), and that license does not permit redistributing modified copies of it. To respect that, the modified source code for this fork is not published here — only the Docker image and usage docs are. Sorry for any inconvenience this causes if you were expecting a browsable source tree.
>
> _Last updated: 2026-08-22 — merged upstream `master` (v4.17.0, 411 commits) into `master-unlocked`._

---

## Table of contents

- [What's unlocked](#-whats-unlocked)
- [Quick install (Docker)](#-quick-install-docker)
- [Full docker-compose example](#-full-docker-compose-example)
- [Updating an existing install](#-updating-an-existing-install)
- [Local development setup](#️-local-development-setup-macos)
- [WhatsApp integration](#-whatsapp-integration)
- [Usage examples](#-usage-examples)
- [Core Chatwoot features](#core-chatwoot-features)
- [Documentation & community](#documentation--community)

---

## ✨ What's unlocked

| Feature | Stock Chatwoot (self-hosted) | This fork |
|---|---|---|
| Agent seats | Limited by plan | Unlimited |
| Captain AI (documents, responses, tools) | Enterprise plan required | Unlimited, enabled |
| SLA policies | Enterprise plan required | Enabled |
| Custom Roles | Enterprise plan required | Enabled |
| Audit Logs | Enterprise plan required | Enabled |
| SAML SSO | Enterprise plan required | Enabled |
| Upgrade prompts / paywall modals | Shown | Removed |
| Add Agent form | Email + role only | Adds password & confirm-password fields |
| Agent list | Basic | Shows verified/pending badge, joined date, confirmed date |
| Agent confirmation | Email only | Admin can manually confirm or resend confirmation email |
| AI Provider settings | Not available | Per-account page: configure OpenAI, OpenRouter, or any custom OpenAI-compatible endpoint |
| Inbox avatar in sidebar | Generic channel icon | Custom inbox avatar image |
| WhatsApp voice calling | Feature-flag gated | Enabled by default (upstream 4.17.0 feature) |
| Version string | Standard | `4.17.0-unlocked` |

---

## 🚀 Quick install (Docker)

### 1. Download the docker-compose file

```bash
wget -O docker-compose.yml https://raw.githubusercontent.com/engacs/chatwoot/master-unlocked/docker-compose.production.yaml
```

### 2. Create the environment file

```bash
wget -O .env https://raw.githubusercontent.com/engacs/chatwoot/master-unlocked/.env.example
```

Edit `.env` and set at minimum:
```env
SECRET_KEY_BASE=your_secret_key        # generate with: openssl rand -hex 64
POSTGRES_PASSWORD=your_db_password
REDIS_PASSWORD=your_redis_password
FRONTEND_URL=https://your-domain.com
```

### 3. Point docker-compose at this image

In `docker-compose.yml`, the `image:` line should read:
```yaml
image: engacs/chatwoot:latest
```

Or pin to a specific release: `engacs/chatwoot:v4.17.0-unlocked`.

### 4. Run

```bash
# Create the database
docker compose run --rm rails bundle exec rails db:chatwoot_prepare

# Start all services
docker compose up -d
```

Open `https://your-domain.com` — done. ✅

---

## 📄 Full docker-compose example

A complete, working example (Postgres with `pgvector`, Redis, Rails, Sidekiq):

```yaml
version: '3'
services:
  base: &base
    image: engacs/chatwoot:latest
    env_file: .env
    volumes:
      - storage_data:/app/storage

  rails:
    <<: *base
    depends_on:
      - postgres
      - redis
    ports:
      - '127.0.0.1:3000:3000'
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    entrypoint: docker/entrypoints/rails.sh
    command: ['bundle', 'exec', 'rails', 's', '-p', '3000', '-b', '0.0.0.0']
    restart: always

  sidekiq:
    <<: *base
    depends_on:
      - postgres
      - redis
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    command: ['bundle', 'exec', 'sidekiq', '-C', 'config/sidekiq.yml']
    restart: always

  postgres:
    image: pgvector/pgvector:pg16
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=chatwoot
      - POSTGRES_USER=chatwoot
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

  redis:
    image: redis:alpine
    restart: always
    command: ["sh", "-c", "redis-server --requirepass \"$REDIS_PASSWORD\""]
    env_file: .env
    volumes:
      - redis_data:/data

volumes:
  storage_data:
  postgres_data:
  redis_data:
```

Put a matching `POSTGRES_PASSWORD` in your `.env` file rather than hardcoding it in the compose file.

`rails` serves the app behind a reverse proxy (nginx/Caddy/Traefik) on `127.0.0.1:3000` — point your reverse proxy's `FRONTEND_URL` domain at that port with TLS termination.

---

## 🔄 Updating an existing install

```bash
cd /path/to/your/chatwoot-install

# Back up the database first — always, before any version jump
docker compose exec postgres pg_dump -U chatwoot chatwoot > backup_$(date +%Y%m%d).sql

# Pull the newest image
docker compose pull

# Recreate containers with the new image (volumes/data untouched)
docker compose up -d

# Run pending migrations
docker compose exec rails bundle exec rails db:migrate

# Restart to pick up the new code cleanly
docker compose restart rails sidekiq
```

---

## 🛠️ Local development setup (macOS)

```bash
# 1. Install dependencies
brew install rbenv postgresql@16 redis overmind
rbenv install $(cat .ruby-version)
eval "$(rbenv init -)"

# 2. Start services
brew services start postgresql@16
brew services start redis

# 3. Install packages
bundle install
pnpm install

# 4. Set up environment
cp .env.example .env
# Edit .env: set SECRET_KEY_BASE, POSTGRES_HOST=localhost, POSTGRES_USERNAME=<your mac username>

# 5. Set up database
bundle exec rails db:create db:schema:load db:seed

# 6. Run the app
pnpm dev
```

Open **http://localhost:3000** — default admin credentials are printed after `db:seed`.

---

## 📱 WhatsApp integration

Connect WhatsApp to this Chatwoot instance using the **[WhatsApp–Chatwoot Bridge](https://github.com/engacs/chatwoot-bridge)** — a companion server that links WhatsApp (via QR-code linked device) to any Chatwoot API inbox.

**Features:** multi-account, two-way messaging, media support, webhook debug logs, per-account log settings.

```bash
git clone https://github.com/engacs/chatwoot-bridge
cd chatwoot-bridge
# See README for full setup
```

For native WhatsApp Cloud API voice calling (built into this fork, upstream 4.17.0 feature), enable it per-inbox from **Inbox Settings → WhatsApp → Enable voice calling** — no extra configuration needed beyond a WhatsApp Cloud inbox already subscribed to Meta's calling API.

---

## 💡 Usage examples

### Add an agent with a custom password (instead of a random temp password)

1. **Settings → Agents → Add Agent**
2. Fill in name, email, role
3. This fork adds **Password** and **Confirm Password** fields to the form — set one directly instead of relying on the emailed temporary password
4. Save — the agent can log in immediately with the password you set

### Manually confirm an agent (skip email confirmation)

If an invited agent hasn't clicked their confirmation email:

1. **Settings → Agents**
2. Find the agent — a **Pending** badge shows next to unconfirmed agents, along with their joined date
3. Click **Confirm** to activate the account immediately, or **Resend confirmation email** to re-send the link

### Configure a custom AI provider (OpenAI, OpenRouter, or any OpenAI-compatible endpoint)

1. **Settings → Integrations → AI Provider**
2. Choose a provider:
   - **OpenAI** — paste your API key
   - **OpenRouter** — paste your OpenRouter API key
   - **Custom** — set a base URL (e.g. a self-hosted vLLM/Ollama-compatible endpoint) and API key
3. Pick the model to use for Captain AI responses
4. Save — Captain AI now routes through your configured provider instead of requiring Chatwoot's own AI subscription

### Set a custom inbox avatar

1. **Settings → Inboxes → (select an inbox) → Settings**
2. Upload an avatar image
3. It now shows in the conversation sidebar instead of the generic channel-type icon

---

## Core Chatwoot features

Chatwoot is the modern, open-source, and self-hosted customer support platform designed to help businesses deliver exceptional customer support experience. Built for scale and flexibility, Chatwoot gives you full control over your customer data while providing powerful tools to manage conversations across channels.

### ✨ Captain – AI Agent for Support

Supercharge your support with Captain, Chatwoot's AI agent. Captain helps automate responses, handle common queries, and reduce agent workload—ensuring customers get instant, accurate answers. With Captain, your team can focus on complex conversations while routine questions are resolved automatically. Read more about Captain [here](https://chwt.app/captain-docs).

### 💬 Omnichannel Support Desk

Chatwoot centralizes all customer conversations into one powerful inbox, no matter where your customers reach out from. It supports live chat on your website, email, Facebook, Instagram, Twitter, WhatsApp, Telegram, Line, SMS etc.

### 📚 Help center portal

Publish help articles, FAQs, and guides through the built-in Help Center Portal. Enable customers to find answers on their own, reduce repetitive queries, and keep your support team focused on more complex issues.

### 🗂️ Other features

#### Collaboration & Productivity

- Private Notes and @mentions for internal team discussions.
- Labels to organize and categorize conversations.
- Keyboard Shortcuts and a Command Bar for quick navigation.
- Canned Responses to reply faster to frequently asked questions.
- Auto-Assignment to route conversations based on agent availability.
- Multi-lingual Support to serve customers in multiple languages.
- Custom Views and Filters for better inbox organization.
- Business Hours and Auto-Responders to manage response expectations.
- Teams and Automation tools for scaling support workflows.
- Agent Capacity Management to balance workload across the team.

#### Customer Data & Segmentation
- Contact Management with profiles and interaction history.
- Contact Segments and Notes for targeted communication.
- Campaigns to proactively engage customers.
- Custom Attributes for storing additional customer data.
- Pre-Chat Forms to collect user information before starting conversations.

#### Integrations
- Slack Integration to manage conversations directly from Slack.
- Dialogflow Integration for chatbot automation.
- Dashboard Apps to embed internal tools within Chatwoot.
- Shopify Integration to view and manage customer orders right within Chatwoot.
- Use Google Translate to translate messages from your customers in realtime.
- Create and manage Linear tickets within Chatwoot.

#### Reports & Insights
- Live View of ongoing conversations for real-time monitoring.
- Conversation, Agent, Inbox, Label, and Team Reports for operational visibility.
- CSAT Reports to measure customer satisfaction.
- Downloadable Reports for offline analysis and reporting.

## Documentation & community

Detailed documentation is available at [chatwoot.com/help-center](https://www.chatwoot.com/help-center).

### Translation process

The translation process for Chatwoot web and mobile app is managed at [https://translate.chatwoot.com](https://translate.chatwoot.com) using Crowdin. Please read the [translation guide](https://www.chatwoot.com/docs/contributing/translating-chatwoot-to-your-language) for contributing to Chatwoot.

### Security

Looking to report a vulnerability? Please refer our [SECURITY.md](./SECURITY.md) file.

### Community

If you need help or just want to hang out, come, say hi on our [Discord](https://discord.gg/cJXdrwS) server.

### Contributors

Thanks goes to all these [wonderful people](https://www.chatwoot.com/docs/contributors):

<a href="https://github.com/chatwoot/chatwoot/graphs/contributors"><img src="https://opencollective.com/chatwoot/contributors.svg?width=890&button=false" /></a>


*Chatwoot* &copy; 2017-2026, Chatwoot Inc - Released under the MIT License. Portions under `enterprise/` are licensed separately — see [`LICENSE`](./LICENSE) and [`enterprise/LICENSE`](./enterprise/LICENSE).
