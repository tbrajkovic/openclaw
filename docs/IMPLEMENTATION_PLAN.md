# OpenClaw Multi-Project Automation — Implementation Plan

## Context

OpenClaw running in Docker at `http://127.0.0.1:18789/` (code at `C:\Users\tom\openclaw`, config at `C:\Users\tom\.openclaw`). 37 extensions (Telegram, WhatsApp, Gmail, etc.) and 50+ skills available — only default "main" agent configured so far.

Goal: Automate tasks across 10 projects, starting with Fixatia's two-sided lead generation. Daily summaries + Telegram orchestration.

**Decisions:** Telegram for messaging, new dedicated Gmail for outreach, build on existing OpenClaw install.

---

## Architecture: Single Instance, 6 Agents

Agent names are **project-scoped** so more project agents can be added later.

| Agent | Name | Role |
|-------|------|------|
| **Orchestrator** | `orchestrator` | Telegram command interface, routes requests, delivers summaries |
| **Fixatia Investor Hunter** | `fixatia-investor-hunter` | Find & contact potential property investors worldwide |
| **Fixatia Partner Hunter** | `fixatia-partner-hunter` | Find & recruit regional partners in 6 countries |
| **Project Monitor** | `project-monitor` | Health checks, GA4 analytics, anomaly detection |
| **Scraper Watcher** | `fixatia-scraper-watcher` | Fixatia scraper status monitoring + alerts |
| **Content Ops** | `content-ops` | Directory submissions, doc sync, pitch generation |

```
You (Telegram)
  → orchestrator
      → fixatia-investor-hunter   (investors from ANY country/language)
      → fixatia-partner-hunter    (partners in 6 EU countries, local languages)
      → project-monitor           (all 10 projects)
      → fixatia-scraper-watcher   (scraper status)
      → content-ops               (submissions + docs)
```

Future: `domebay-lead-hunter`, `homevisto-outreach`, `aiscriba-demo-outreach`, etc.

---

## Fixatia Lead Generation: Two Pipelines

### Pipeline 1: `fixatia-investor-hunter` — Property Investors

**Goal:** Find international investors and invite them to browse Fixatia's 12,500+ distressed properties with green ROI analysis.

**Investors can be from ANY country, ANY language.** Not limited to the 6 countries Fixatia operates in — a Japanese fund, a Saudi family office, or a Canadian retiree could all be prospects.

**Target profiles:**
- Property investment funds focusing on Europe
- Individual investors seeking EU renovation/flip opportunities
- Digital nomads/remote workers relocating to Southern Europe
- Diaspora investors (Chinese, Arabic, Russian speakers interested in EU property)
- Retirement investors seeking affordable warm-climate property
- Real estate influencers and content creators covering EU markets

**Research approach:**
- Web search in multiple languages for investor communities, forums, funds
- LinkedIn company profiles (via browser, not API)
- PropTech conference attendee lists, real estate meetup organizers
- Reddit, BiggerPockets, Facebook groups focused on EU property investment
- Industry publications and their readership

---

### Pipeline 2: `fixatia-partner-hunter` — Regional Service Partners

**Goal:** Find professional service providers in Fixatia's 6 countries and invite them to register. Fixatia sends them qualified investor leads.

**6 countries = 6 languages** (expandable):
- Spain (ES), Portugal (PT), Italy (IT), France (FR), Germany (DE), Croatia (HR)

**Partner types (from Fixatia's business model):**

| Type | What Fixatia Offers | Search Focus |
|------|---------------------|--------------|
| Legal Expert | Qualified buyer leads needing purchase legal advice | RE law firms, foreign buyer specialists |
| Notary | Transaction volume from cross-border purchases | Notaries near auction courts |
| Financial Institution | Green mortgage leads with calculated renovation costs | Banks with non-resident mortgage products |
| Mortgage Broker | Pre-qualified leads with ROI analysis | Foreign buyer mortgage specialists |
| Green Energy Contractor | Renovation projects post-purchase | Solar installers, heat pump companies, retrofit firms |
| Real Estate Agent | Investors needing local guidance | Agents near auction property clusters, English-speaking |
| Real Estate Broker | Volume deal flow from platform | Auction/distressed property specialists |

**Research approach (in local language of each country):**
- Country-specific business directories
- Professional association member lists (bar associations, notary chambers, RE agent federations)
- Google Maps/local search for services near property clusters
- Industry events and trade shows in each country

---

### Email Strategy: AI-Personalized, Not Generic Templates

**Key principle: Every email is unique and highly specialized for the individual recipient.**

No fill-in-the-blanks templates. Instead, the agent:

1. **Researches the prospect** — visits their website, reads their about page, understands their specialization, recent activity, portfolio, etc.
2. **Builds a prospect profile** — what they do, where they operate, what matters to them, what language they communicate in
3. **Generates a personalized email** using AI — referencing specific details about their business and explaining exactly how Fixatia is relevant to *them specifically*

**Example for a partner (Italian lawyer):**
> Subject: Opportunità di leads qualificati per il suo studio — investitori immobiliari all'asta in Toscana
>
> Gentile Avv. Rossi, ho notato che il suo studio a Firenze è specializzato in diritto immobiliare per acquirenti stranieri. Fixatia è una piattaforma che aggrega aste immobiliari giudiziarie dal PVP Giustizia e collega investitori internazionali con professionisti locali...

**Example for an investor (German fund):**
> Subject: 12.500+ Zwangsversteigerungen in 6 EU-Ländern — mit Green-ROI-Analyse
>
> Sehr geehrte Frau Schmidt, ich habe gesehen, dass Ihre Firma auf Immobilieninvestments in Südeuropa spezialisiert ist...

### Smart AI Provider Usage

| Task | Recommended Provider | Reasoning |
|------|---------------------|-----------|
| **Email generation** | Gemini 2.0 Flash (free tier) | Google AI Studio free tier: generous limits, excellent multilingual, fast |
| **Prospect research summary** | Gemini 2.0 Flash (free tier) | Summarizing web pages is a straightforward task |
| **Lead qualification scoring** | GPT-4o-mini (existing key) | Already have OpenAI key from Fixatia, cheap ($0.15/1M input) |
| **Complex reasoning** (agent orchestration) | Claude via OpenClaw | OpenClaw already runs on an AI provider for agent logic |
| **Fallback** | Groq (free tier, Llama 3) | Free, fast, good for simple tasks when other limits hit |

---

### Shared Lead Rules

- Max 20 emails/day total (split between pipelines as needed)
- 2-minute spacing between sends
- Weekdays only (Mon-Fri)
- Every email: Fixatia identification, business reason, unsubscribe link
- GDPR: only public business contacts, document sources, maintain opt-out list
- Language auto-detected from prospect's website/location

### Lead Tracking

```
workspace/fixatia/leads/
  investors.csv      # date, company, contact, email, language, country, score, status, email_sent, response, notes
  partners.csv       # date, company, type, region, country, language, email, score, status, email_sent, response, notes
  opt-out.csv        # email, date_added
  search-history.csv # date, query, language, results_count, pipeline
```

---

## Configuration Changes

### 1. Add to `.env` (`C:\Users\tom\openclaw\.env`)

```env
# AI Provider (for OpenClaw agent reasoning)
ANTHROPIC_API_KEY=sk-ant-...

# Telegram
TELEGRAM_BOT_TOKEN=<from BotFather>

# Gmail OAuth2 (dedicated outreach Gmail)
GMAIL_CLIENT_ID=<from Google Cloud Console>
GMAIL_CLIENT_SECRET=<from Google Cloud Console>
GMAIL_REFRESH_TOKEN=<from OAuth consent flow>

# For email generation (free tier)
GOOGLE_AI_API_KEY=<from Google AI Studio — free>

# For lead scoring (existing key from Fixatia)
OPENAI_API_KEY=<existing>

# Fixatia DB (read-only, for monitoring)
FIXATIA_DB_URL=<from dashboard config>

# Google Analytics (reuse existing)
GOOGLE_APPLICATION_CREDENTIALS_JSON=<existing SA>
```

### 2. Agent Configs in `openclaw.json`

Add to `C:\Users\tom\.openclaw\openclaw.json` under `agents.list`:

```json5
{
  agents: {
    list: [
      // ... existing main agent ...
      { id: "orchestrator", name: "Orchestrator", workspace: "~/.openclaw/workspace" },
      { id: "fixatia-investor-hunter", name: "Fixatia Investor Hunter", workspace: "~/.openclaw/workspace" },
      { id: "fixatia-partner-hunter", name: "Fixatia Partner Hunter", workspace: "~/.openclaw/workspace" },
      { id: "project-monitor", name: "Project Monitor", workspace: "~/.openclaw/workspace" },
      { id: "fixatia-scraper-watcher", name: "Fixatia Scraper Watcher", workspace: "~/.openclaw/workspace" },
      { id: "content-ops", name: "Content Ops", workspace: "~/.openclaw/workspace" }
    ]
  },
  channels: {
    telegram: { accounts: { default: { botToken: "${TELEGRAM_BOT_TOKEN}" } } }
  },
  bindings: [
    { agentId: "orchestrator", match: { channel: "telegram" } }
  ]
}
```

### 3. Cron Jobs — `C:\Users\tom\.openclaw\cron\jobs.json`

```json
{
  "version": 1,
  "jobs": [
    { "id": "daily-summary", "trigger": { "cron": "0 9 * * *" }, "message": { "text": "Generate daily summary", "agentSessionKey": "agent:orchestrator:main" } },
    { "id": "investor-research", "trigger": { "cron": "0 10 * * 1-5" }, "message": { "text": "Research new investor leads", "agentSessionKey": "agent:fixatia-investor-hunter:main" } },
    { "id": "investor-outreach", "trigger": { "cron": "0 14 * * 1-5" }, "message": { "text": "Send investor outreach emails", "agentSessionKey": "agent:fixatia-investor-hunter:main" } },
    { "id": "partner-research", "trigger": { "cron": "0 10 * * 1-5" }, "message": { "text": "Research new partner leads", "agentSessionKey": "agent:fixatia-partner-hunter:main" } },
    { "id": "partner-outreach", "trigger": { "cron": "0 15 * * 1-5" }, "message": { "text": "Send partner outreach emails", "agentSessionKey": "agent:fixatia-partner-hunter:main" } },
    { "id": "investor-followup", "trigger": { "cron": "0 11 * * 1-5" }, "message": { "text": "Send follow-up emails to investors", "agentSessionKey": "agent:fixatia-investor-hunter:main" } },
    { "id": "partner-followup", "trigger": { "cron": "30 11 * * 1-5" }, "message": { "text": "Send follow-up emails to partners", "agentSessionKey": "agent:fixatia-partner-hunter:main" } },
    { "id": "health-check", "trigger": { "cron": "*/30 * * * *" }, "message": { "text": "Run health checks", "agentSessionKey": "agent:project-monitor:main" } },
    { "id": "scraper-check", "trigger": { "cron": "0 */4 * * *" }, "message": { "text": "Check scraper status", "agentSessionKey": "agent:fixatia-scraper-watcher:main" } }
  ]
}
```

### 4. Volume Mount — `docker-compose.yml`

```yaml
# Under openclaw-gateway volumes:
- /i/Users/tom/Documents/Projects/ProjectSummaries:/data/project-summaries:ro
```

### 5. Workspace Structure

```
~/.openclaw/workspace/
  fixatia/
    context/
      fixatia-overview.md       # Condensed Fixatia pitch for agent context
      partner-types.md          # Partner type descriptions + value props
      investor-personas.md      # Investor persona descriptions
      countries.md              # 6 countries + regions + languages
    email-guidelines.md         # Tone, compliance rules, personalization instructions
    leads/
      investors.csv
      partners.csv
      opt-out.csv
      search-history.csv
  shared/
    projects-config.json        # symlink from ProjectSummaries
```

---

## Daily Summary (9:00 AM via Telegram)

```
--- Daily Report ---
HEALTH: 8/10 projects healthy
  [!] Fixatia: 423ms (slow)

ANALYTICS (24h):
  Fixatia: 45 users (+12%), 3 signups

SCRAPER: SUCCESS, +14 properties

INVESTOR LEADS:
  Researched: 8 | Qualified: 5 | Emails: 4 | Responses: 1
  Pipeline total: 23 contacted, 3 responses

PARTNER LEADS:
  Researched: 10 | Qualified: 7 | Emails: 6 | Responses: 2
  New: lawyer (Algarve PT), mortgage broker (Barcelona ES)
  Pipeline total: 31 contacted, 5 responses

TODAY: 2 investor follow-ups, 3 partner follow-ups
```

---

## Telegram Commands

| Command | Agent | Action |
|---------|-------|--------|
| `status` | project-monitor | Health summary all projects |
| `leads` | both hunters | Today's lead activity |
| `find investors` | fixatia-investor-hunter | Manual research cycle |
| `find partners in [country]` | fixatia-partner-hunter | Research for specific country |
| `scraper` | fixatia-scraper-watcher | Latest scraper status |
| `summary` | orchestrator | Full daily report on demand |
| `pause leads` | both hunters | Stop automated outreach |
| `resume leads` | both hunters | Resume automated outreach |

---

## Implementation Phases

### Phase 1: Foundation
- [ ] Add AI provider keys to `.env` (Anthropic, Google AI Studio free, existing OpenAI)
- [ ] Create Telegram bot via BotFather, add token
- [ ] Configure `orchestrator` agent in `openclaw.json`
- [ ] Add Telegram channel config + binding
- [ ] Add ProjectSummaries volume mount, restart containers
- [ ] **Test:** "hello" via Telegram → response

### Phase 2: Monitoring
- [ ] Configure `project-monitor` agent
- [ ] Configure `fixatia-scraper-watcher` agent
- [ ] Implement health checks + GA4 access
- [ ] Implement daily summary + cron schedule
- [ ] **Test:** Daily summary via Telegram

### Phase 3: Investor Lead Generation
- [ ] Create dedicated Gmail, set up OAuth2
- [ ] Write Fixatia context files (`fixatia-overview.md`, `investor-personas.md`, `email-guidelines.md`)
- [ ] Configure `fixatia-investor-hunter` with browser + Gmail + Gemini email gen
- [ ] Run first research cycle via Telegram
- [ ] **Test:** AI-personalized email → send to yourself, review quality

### Phase 4: Partner Lead Generation
- [ ] Write partner context files (`partner-types.md`, `countries.md`)
- [ ] Configure `fixatia-partner-hunter` with browser + Gmail + Gemini email gen
- [ ] Start with Portugal (strongest Fixatia data), then expand
- [ ] **Test:** Personalized partner email in Portuguese → review quality

### Phase 5: Content & Polish
- [ ] Configure `content-ops` agent
- [ ] Enable all cron schedules
- [ ] **Test:** Full week of automated operation

### Phase 6: Scale (ongoing)
- [ ] Add agents for other projects (`domebay-lead-hunter`, etc.)
- [ ] Refine email quality based on response rates
- [ ] Migrate leads from CSV to Fixatia DB
- [ ] Expand investor outreach to more languages/regions

---

## Key Existing Files

| File | Read By |
|------|---------|
| `I:\...\ProjectSummaries\dashboard\projects.config.json` | project-monitor |
| `I:\...\ProjectSummaries\dashboard\src\lib\connectors\fixatia.ts` | fixatia-scraper-watcher |
| `I:\...\ProjectSummaries\dashboard\src\lib\connectors\base.ts` | project-monitor |
| `I:\...\ProjectSummaries\dashboard\src\lib\connectors\google-analytics.ts` | project-monitor |
| `C:\Users\tom\openclaw\docker-compose.yml` | Modify — add volume mount |
| `C:\Users\tom\openclaw\.env` | Modify — add API keys |
| `C:\Users\tom\.openclaw\openclaw.json` | Modify — add agents, channels, bindings |
| `C:\Users\tom\.openclaw\cron\jobs.json` | Modify — add schedules |

## Verification

1. `docker compose restart` — containers pick up new config
2. `http://127.0.0.1:18789/` — all 6 agents visible
3. Telegram "status" → health summary
4. Telegram "find investors" → research cycle, reports findings
5. Telegram "find partners in portugal" → finds lawyers/banks/contractors in PT
6. Review AI-generated emails — verify they are truly personalized, not generic
7. Send test emails to yourself in multiple languages
8. Next morning: daily summary at 9 AM with both pipelines
