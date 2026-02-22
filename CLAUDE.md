# OpenClaw Automation Setup

## Project Overview

This is a local OpenClaw instance running in Docker at `http://127.0.0.1:18789/` that automates tasks across 10 projects managed from the ProjectSummaries dashboard.

**Primary automation:** Fixatia (European real estate investment platform) two-sided lead generation + Gmail outreach.

## Architecture

Single OpenClaw instance, 3 active agents + standalone outreach scripts:

| Component | Type | Role |
|-----------|------|------|
| Orchestrator | Agent (Groq/Llama) | Telegram command interface, daily summaries |
| Investor Hunter | Agent (OpenAI GPT-4o-mini) | Research: find property investors via web search |
| Partner Hunter | Agent (OpenAI GPT-4o-mini) | Research: recruit partners in 6 EU countries |
| send-outreach.js | Script | Send personalized outreach emails via Gmail SMTP |
| send-followups.js | Script | Send follow-up emails to non-responders |
| check-inbox.js | Script | Monitor inbox for replies and opt-outs |

**Why agents for research, scripts for email:** OpenClaw wraps hook/cron messages with a security notice that prevents GPT-4o-mini from calling `exec`. Research tools (web_search, web_fetch, read, write) work fine through agents; email sending via curl SMTP runs as standalone Node.js scripts.

Future agents: `project-monitor`, `fixatia-scraper-watcher`, `content-ops`, etc.

## Scheduling

### OpenClaw Cron (agents)
| Job | Schedule | Agent |
|-----|----------|-------|
| Daily Summary | 09:00 daily | orchestrator |
| Investor Research | 10:00 Mon-Fri CET | investor-hunter |
| Partner Research | 10:30 Mon-Fri CET | partner-hunter |

### Windows Task Scheduler (scripts)
| Task | Schedule | Script |
|------|----------|--------|
| FixatiaFollowups | 11:00 Mon-Fri | outreach-followups.bat (inbox check + follow-ups) |
| FixatiaInvestorOutreach | 14:00 Mon-Fri | outreach-investors.bat |
| FixatiaPartnerOutreach | 15:00 Mon-Fri | outreach-partners.bat |

## Key Directories

- **OpenClaw code:** `C:\Users\tom\openclaw` (this repo)
- **OpenClaw config:** `C:\Users\tom\.openclaw`
- **Orchestrator workspace:** `C:\Users\tom\.openclaw\workspace-orchestrator`
- **Investor Hunter workspace:** `C:\Users\tom\.openclaw\workspace-investor-hunter`
- **Partner Hunter workspace:** `C:\Users\tom\.openclaw\workspace-partner-hunter`
- **Outreach scripts:** `C:\Users\tom\.openclaw\scripts`
- **Cron jobs:** `C:\Users\tom\.openclaw\cron\jobs.json`
- **ProjectSummaries dashboard:** `I:\Users\tom\Documents\Projects\ProjectSummaries`
- **Fixatia project:** `I:\Users\tom\Documents\Projects\Fixatia`

## Fixatia Lead Generation

### Research (via agents)
- **Investor Hunter:** Searches globally in multiple languages for small/mid property investors. Uses Brave Search + web_fetch. Logs to `investors.csv`.
- **Partner Hunter:** Searches in local languages across 6 EU countries for legal experts, notaries, brokers, etc. Logs to `partners.csv`.

### Outreach (via scripts)
- **send-outreach.js:** Reads CSV for "researched" leads, calls OpenAI API to generate personalized email, sends via curl SMTP, updates CSV status.
- **send-followups.js:** Finds leads emailed 3+ days ago, generates follow-up with different angle, CAN include links.
- **check-inbox.js:** Reads Gmail via IMAP, detects replies and opt-outs, updates CSVs.

### Email Rules
- Every email is unique, personalized via AI
- First email: NO links (links come in follow-ups only)
- Written in prospect's local language
- Signed as "Equipo Fixatia" / "L'equipe Fixatia" etc.
- Max 20 emails/day, 2-min spacing, weekdays only, GDPR compliant
- Gmail: fixatiacom@gmail.com with app password

## Manual Commands

```bash
# Research (via hooks API)
curl -X POST http://127.0.0.1:18789/hooks/agent -H "Authorization: Bearer <hook-token>" -H "Content-Type: application/json" -d '{"agentId":"fixatia-investor-hunter","sessionKey":"hook:research","message":"Run a research cycle..."}'

# Outreach (via docker exec)
docker exec openclaw-openclaw-gateway-1 bash -c "node /home/node/.openclaw/scripts/send-outreach.js --type investors --max 5"
docker exec openclaw-openclaw-gateway-1 bash -c "node /home/node/.openclaw/scripts/send-outreach.js --type partners --max 5"
docker exec openclaw-openclaw-gateway-1 bash -c "node /home/node/.openclaw/scripts/send-outreach.js --type investors --max 1 --test-to fixatiacom@gmail.com"
docker exec openclaw-openclaw-gateway-1 bash -c "node /home/node/.openclaw/scripts/send-outreach.js --type investors --dry-run"

# Follow-ups
docker exec openclaw-openclaw-gateway-1 bash -c "node /home/node/.openclaw/scripts/send-followups.js --type investors --max 5"

# Check inbox
docker exec openclaw-openclaw-gateway-1 bash -c "node /home/node/.openclaw/scripts/check-inbox.js --update-csv --verbose"

# Read Gmail inbox
docker exec openclaw-openclaw-gateway-1 bash -c 'curl -s --ssl-reqd --url "imaps://imap.gmail.com:993/INBOX" --user "fixatiacom@gmail.com:iebo egzh lvxc nlma" --request "SEARCH ALL"'
```

## Decisions Made

- **Messaging:** Telegram (via BotFather, bot @TomProjectBot)
- **Outreach email:** fixatiacom@gmail.com with app password, via curl SMTP
- **AI for research:** OpenAI GPT-4o-mini (Gemini free tier had 0 quota)
- **AI for email generation:** OpenAI GPT-4o-mini via direct API call in scripts
- **Orchestrator:** Groq/Llama 4 Scout (free tier)
- **Web search:** Brave Search API
- **Lead tracking:** CSV initially, migrate to Fixatia DB later
- **No static templates:** `email-guidelines.md` + `context/` files guide AI generation

## Existing Config Reference

- Main config: `C:\Users\tom\.openclaw\openclaw.json`
- Docker: `C:\Users\tom\openclaw\docker-compose.yml` (gateway port 18789, bridge port 18790)
- Override: `C:\Users\tom\openclaw\docker-compose.override.yml` (env vars, volume mounts)
- Env: `C:\Users\tom\openclaw\.env` (all API keys, Gmail credentials)
- Hooks token: separate from gateway auth token (required by OpenClaw)

## ProjectSummaries Reference

The dashboard at `I:\Users\tom\Documents\Projects\ProjectSummaries` has:
- `dashboard/projects.config.json` — all 10 project URLs, API endpoints, GA4 property IDs
- `dashboard/src/lib/connectors/fixatia.ts` — Fixatia DB queries, scraper status
- `dashboard/src/lib/connectors/base.ts` — health check patterns
- `dashboard/src/lib/connectors/google-analytics.ts` — GA4 integration
