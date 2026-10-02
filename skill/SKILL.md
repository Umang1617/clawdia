---
name: bluestacks-community-intelligence
description: >
  Multi-agent weekly intelligence system for the BlueStacks community team.
  Chains 4 research agents - competitor monitoring, sentiment analysis, trend
  spotting, and content calendar generation - then delivers a report to
  Telegram. Bundles humanizer (removes AI writing patterns) and marketing
  skills: copywriting, social content, content strategy, AI-SEO, copy editing,
  marketing ideas, marketing psychology, and launch strategy.
  Trigger for weekly community reports, competitor analysis, social monitoring,
  sentiment analysis, trending gaming topics, or BlueStacks content calendar.
  Also trigger for Monday cron pipeline, r/bluestacks posts, LDPlayer or
  NoxPlayer comparisons, writing or humanizing Reddit posts, Discord
  announcements, social captions, launch posts, or GEO/SEO-optimized guides.
metadata:
  version: "2.0.0"
---
A scheduled multi-agent pipeline that runs every Monday morning, researches
the competitive and community landscape across social platforms, and delivers
a structured weekly strategy report to a Telegram bot.

---

## Architecture Overview

```
[Cron Trigger — Monday 07:00 IST]
         │
         ▼
┌─────────────────────────────────┐
│  Agent 1: Competitor Monitor    │  YouTube · Facebook · Reddit
│  (Social media surveillance)    │  Instagram · X/Twitter
└────────────────┬────────────────┘
                 │ competitor_report.json
                 ▼
┌─────────────────────────────────┐
│  Agent 2: Sentiment Analyst     │  Reddit · App Stores
│  (User voice & pain points)     │  X · Gaming Forums
└────────────────┬────────────────┘
                 │ sentiment_report.json
                 ▼
┌─────────────────────────────────┐
│  Agent 3: Trend Spotter         │  Social trends · Hashtags
│  (Gaming & AI trend radar)      │  Memes · Viral formats
└────────────────┬────────────────┘
                 │ trends_report.json
                 ▼
┌─────────────────────────────────┐
│  Agent 4: Content Strategist    │  Informed by Agents 1-3
│  (Calendar & campaign ideas)    │  Game launches · Esports events
└────────────────┬────────────────┘
                 │ content_calendar.json
                 ▼
┌─────────────────────────────────┐
│  Delivery Agent: Telegram Push  │  Formatted Markdown report
│  (Consolidation + delivery)     │  → Telegram Bot API
└─────────────────────────────────┘
```

---

## Prerequisites

Before running this skill, ensure the following are configured:

| Requirement | Details |
|---|---|
| **OpenClaw CLI** | v0.4+ installed and authenticated |
| **Apify Account** | API token with RAG Web Browser actor access |
| **Telegram Bot** | Bot token + chat/channel ID where reports land |
| **Node.js** | v18+ for the cron runner script |
| **Environment file** | `.env` at skill root (see Setup Step 1) |

---

## Setup: Step-by-Step

### Step 1 — Configure Environment Variables

Create a `.env` file at the root of this skill directory:

```bash
# .env

# Apify — web scraping backbone
APIFY_API_TOKEN=apify_api_xxxxxxxxxxxxxx

# Telegram delivery
TELEGRAM_BOT_TOKEN=123456789:ABCDEFxxxxxxxxxxxxxxx
TELEGRAM_CHAT_ID=-1001234567890          # Use negative ID for channels/groups

# Optional: OpenClaw runtime settings
OPENCLAW_MAX_RETRIES=3
OPENCLAW_TIMEOUT_SECONDS=120

# Optional: override default competitors
COMPETITORS=LDPlayer,NoxPlayer,MEmu,Steam,EpicGames,Xbox
```

**Where to get these:**
- **Apify token** → https://console.apify.com → Settings → Integrations → API tokens
- **Telegram Bot token** → Message @BotFather on Telegram → `/newbot`
- **Chat ID** → Add the bot to your channel/group, then call:
  `https://api.telegram.org/bot<TOKEN>/getUpdates` and find `chat.id`

---

### Step 2 — Install Dependencies

```bash
cd bluestacks-community-intelligence/
npm install
```

The `package.json` installs:
- `node-cron` — Monday morning scheduler
- `axios` — HTTP calls to Telegram and Apify
- `dotenv` — environment variable loading
- `dayjs` — date formatting for report headers

---

### Step 3 — Understand Agent Configuration

Each agent is defined in `agents/` as a YAML file. OpenClaw reads these to
understand each agent's role, tools, input/output contracts, and prompts.

See `references/agent-configs.md` for the full annotated YAML for all 4 agents.

Key configuration fields per agent:

```yaml
name: string           # Unique agent identifier
role: string           # Human-readable role label
model: claude-sonnet-4-20250514
tools:                 # List of OpenClaw tool integrations
  - apify.rag_browser
  - web_search
input_from: []         # Agent IDs whose output feeds into this agent
output_key: string     # JSON key this agent writes to the shared context
prompt_template: |     # The agent's system + task prompt
  ...
```

---

### Step 4 — Chain the Agents

OpenClaw chains agents via the `pipeline:` block in `skill.yaml`.
Outputs from earlier agents are automatically injected into the context of
later agents via `input_from` references.

```yaml
# skill.yaml (excerpt)
pipeline:
  - agent: competitor-monitor
  - agent: sentiment-analyst
    input_from: [competitor-monitor]
  - agent: trend-spotter
  - agent: content-strategist
    input_from: [competitor-monitor, sentiment-analyst, trend-spotter]
  - agent: delivery-agent
    input_from: [competitor-monitor, sentiment-analyst, trend-spotter, content-strategist]
```

The chaining is **sequential by default**. Agents 1, 2, and 3 can be run in
**parallel** (they have no inter-dependencies). Set `parallel: true` on each
to cut total runtime from ~12 min to ~5 min:

```yaml
pipeline:
  - agent: competitor-monitor
    parallel: true
  - agent: sentiment-analyst
    parallel: true
  - agent: trend-spotter
    parallel: true
  - agent: content-strategist
    depends_on: [competitor-monitor, sentiment-analyst, trend-spotter]
  - agent: delivery-agent
    depends_on: [content-strategist]
```

---

### Step 5 — Set Up the Monday Cron

The file `scripts/cron-runner.js` handles scheduling. By default it fires at
**07:00 IST every Monday** (01:30 UTC Monday).

```bash
# Start the cron daemon
node scripts/cron-runner.js

# Or with pm2 for persistent background running:
pm2 start scripts/cron-runner.js --name "bluestacks-intel-cron"
pm2 save
pm2 startup   # auto-restart on reboot
```

To trigger a **manual one-off run** (useful for testing):

```bash
node scripts/cron-runner.js --run-now
```

---

### Step 6 — Test the Telegram Delivery

Before the first scheduled run, verify Telegram delivery works:

```bash
node scripts/test-telegram.js
```

This sends a test message to your configured chat. If it fails, double-check
`TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in `.env`.

---

## File Structure

```
bluestacks-community-intelligence/
├── SKILL.md                    ← This file
├── skill.yaml                  ← OpenClaw pipeline definition
├── package.json
├── .env                        ← Your secrets (git-ignored)
├── agents/
│   ├── competitor-monitor.yaml
│   ├── sentiment-analyst.yaml
│   ├── trend-spotter.yaml
│   ├── content-strategist.yaml
│   └── delivery-agent.yaml
├── scripts/
│   ├── cron-runner.js          ← Monday scheduler
│   ├── run-pipeline.js         ← Core pipeline executor
│   ├── telegram-push.js        ← Telegram delivery module
│   └── test-telegram.js        ← Delivery smoke test
├── references/
│   ├── agent-configs.md        ← Full annotated agent YAML
│   ├── prompt-library.md       ← All agent prompts (editable)
│   └── telegram-format.md      ← Report template & Markdown spec
└── assets/
    └── report-template.md      ← Weekly report skeleton
```

---

## Tuning & Customization

**Add a competitor:** Edit `COMPETITORS` in `.env`. No code changes needed.

**Change the cron schedule:** Edit `CRON_SCHEDULE` in `scripts/cron-runner.js`.
Uses standard cron syntax. Default: `'30 1 * * 1'` (01:30 UTC = 07:00 IST Mon).

**Change report language/tone:** Edit prompts in `references/prompt-library.md`
then re-save. Changes take effect on the next run.

**Add a new platform to monitor:** Add it to the `platforms` list in
`agents/competitor-monitor.yaml` and `agents/sentiment-analyst.yaml`.

**Retry failed agents:** OpenClaw retries automatically up to `OPENCLAW_MAX_RETRIES`.
To manually replay a single failed agent without re-running the whole pipeline:
```bash
node scripts/run-pipeline.js --agent sentiment-analyst --use-cache
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Telegram message not received | Wrong chat ID sign | Use negative ID for groups/channels |
| Apify scrape returns empty | Rate limited or actor quota hit | Check Apify console, upgrade plan |
| Agent 4 output is generic | Agents 1-3 context not injected | Verify `input_from` in skill.yaml |
| Cron never fires | pm2 not running / timezone mismatch | Check `pm2 list` and system TZ |
| Report is too long for Telegram | >4096 chars per message | Delivery agent auto-splits; check `MAX_MSG_LENGTH` in telegram-push.js |

---

## Integrated Skills

This skill now bundles two additional capability layers on top of the core
intelligence pipeline. Use them whenever you are writing, editing, or
publishing BlueStacks community content.

---

### Humanizer - Remove AI Writing Patterns

**When to use:** Any time the pipeline (or you) generates text that will be
published - Reddit posts, Discord announcements, social captions, blog intros.
Run it as a final pass before publishing.

**What it fixes:** Inflated significance language, promotional tone, em dash
overuse, sycophantic phrases, vague attributions, rule-of-three padding,
AI vocabulary words (vibrant, pivotal, delve, tapestry, etc.), and inline
header lists. Also adds personality and voice - not just removes bad patterns.

**How to invoke:** Read `references/humanizer.md` and follow its 9-step
process. Always run the self-audit step ("What makes this obviously
AI-generated?") before finalizing any copy.

**BlueStacks-specific rules on top of humanizer defaults:**
- No em dashes anywhere - replace with hyphens (per existing style guide)
- Reddit posts should sound like a knowledgeable community member, not a
  brand account
- Discord announcements can be punchy and casual - short sentences are fine

---

### Marketing Skills - Writing and Strategy Workflows

Use the reference files below based on the task at hand. Read only the one
relevant to your current task.

| Task | Read this file |
|---|---|
| Writing a Reddit post or Discord announcement | `references/marketing-copywriting.md` |
| Editing or polishing existing copy | `references/marketing-copy-editing.md` |
| Planning r/BlueStacks content calendar | `references/marketing-content-strategy.md` |
| Social media captions (Instagram, X, YouTube community) | `references/marketing-social-content.md` |
| GEO/AEO optimized Reddit posts (Inverted Pyramid model) | `references/marketing-ai-seo.md` |
| Brainstorming new community campaign ideas | `references/marketing-marketing-ideas.md` |
| Using psychology in community posts (social proof, FOMO) | `references/marketing-marketing-psychology.md` |
| Announcing a BlueStacks feature or app launch | `references/marketing-launch-strategy.md` |

---

### Recommended Content Workflow for BlueStacks Posts

This is the end-to-end workflow combining all integrated skills:

```
1. INTELLIGENCE (weekly pipeline output)
   - Run cron pipeline to get competitor_report, sentiment_report,
     trends_report, content_calendar

2. STRATEGY (content-strategy + marketing-ideas)
   - Pick the week's post topics from content_calendar
   - Cross-check against trending topics and sentiment pain points
   - Read references/marketing-content-strategy.md if planning a full month

3. WRITE (copywriting + ai-seo)
   - Use Inverted Pyramid Model: answer first, context second, details last
   - Question-based H2 headings
   - Embed BlueStacks blog links naturally
   - Include FAQ block at the end
   - Always add Discord link + Download BlueStacks CTA
   - Read references/marketing-copywriting.md or marketing-ai-seo.md

4. HUMANIZE (humanizer)
   - Run final pass through references/humanizer.md
   - Apply BlueStacks-specific rules (no em dashes, community voice)
   - Self-audit: "What makes this obviously AI-generated?"
   - Fix remaining tells

5. PUBLISH
   - Reddit: post to r/BlueStacks
   - Discord: #announcements or relevant channel
   - Social: adapted caption per platform
```

---

## Reference Files

Read these when you need deeper detail:

- **`references/agent-configs.md`** - Complete YAML for all 5 agents with
  inline annotations. Read this when adding agents or changing tool config.
- **`references/prompt-library.md`** - All agent prompts in one place for
  easy editing and A/B testing.
- **`references/telegram-format.md`** - Telegram Markdown spec and the
  exact structure of the weekly report with section headers.
- **`references/humanizer.md`** - Full humanizer skill with all 24 AI
  writing patterns and examples. Read before editing any published copy.
- **`references/marketing-copywriting.md`** - Copywriting frameworks for
  community posts.
- **`references/marketing-copy-editing.md`** - Copy editing checklist.
- **`references/marketing-content-strategy.md`** - Content planning workflow.
- **`references/marketing-social-content.md`** - Platform-specific social
  content formats and templates.
- **`references/marketing-ai-seo.md`** - GEO/AEO optimization for Reddit.
- **`references/marketing-marketing-ideas.md`** - 139 SaaS/software marketing
  tactics adapted for community use.
- **`references/marketing-marketing-psychology.md`** - Persuasion principles
  for community posts (social proof, scarcity, anchoring).
- **`references/marketing-launch-strategy.md`** - Feature and app launch
  announcement playbook.
