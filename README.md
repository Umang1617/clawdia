# 🦞 Clawdia

**Community Intelligence System.** A five-agent OpenClaw skill that studies competitors, listens to the community, spots trends, and delivers a ready-to-use content calendar to Telegram every Monday at 7:00 AM IST.

![OpenClaw](https://img.shields.io/badge/OpenClaw-E4572E?style=flat-square)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![Claude](https://img.shields.io/badge/Claude_Sonnet-D97757?style=flat-square&logo=anthropic&logoColor=white)
![Apify](https://img.shields.io/badge/Apify-97D700?style=flat-square&logo=apify&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=flat-square&logo=telegram&logoColor=white)

🏆 **2nd place, OpenClaw Hackathon 2026** (Track 1: Build & Create). It started life as a project called BlueClaw.

<!-- TODO: add a screenshot of a sample Telegram report here -->

---

## The problem

Community managers lose hours every week to four problems:

- **Competitor blindness.** Tracking rival platforms across Reddit, YouTube, Instagram and X by hand never gives full coverage.
- **Sentiment vacuum.** User complaints on Reddit, app store reviews and forums go unread until they blow up.
- **Content paralysis.** Without fresh intelligence, weekly content turns reactive and inconsistent.
- **Missed opportunities.** Game launches, memes, esports events and AI news pass by before anyone can connect the brand to them.

Clawdia handles all four in one automated weekly run.

## What you get every Monday

| Section | What's inside |
|---|---|
| 📊 Header | Week number, date, one-line mood signal |
| ⚡ TL;DR | The 3-4 most critical signals across all agents |
| 🏁 Competitor Watch | Top 3 competitor moves, labeled threat or opportunity |
| 💬 Community Voice | Top complaints with real quotes, praise themes, #1 feature request |
| 🔥 Trending Now | Top 3 trends with a relevance score and content angle |
| 📅 Content Calendar | 5-7 post ideas with hook, CTA, priority and best post time |
| 🗓️ Event Hooks | Upcoming dates rated by urgency |

Long reports split automatically at section boundaries to fit Telegram's 4,096 character limit, and fall back to plain text if MarkdownV2 formatting is rejected.

## The five agents

```
Monday 07:00 IST (cron)
   ├─ Agent 1: Competitor Monitor ─┐
   ├─ Agent 2: Sentiment Analyst  ─┼─► Agent 4: Content Strategist ─► Agent 5: Delivery ─► Telegram
   └─ Agent 3: Trend Spotter      ─┘
```

| # | Agent | Reads | Produces |
|---|---|---|---|
| 1 | **Competitor Monitor** | YouTube, Facebook, Reddit, Instagram, X | `competitor_report.json`: top content, cadence, hashtags, threat signals |
| 2 | **Sentiment Analyst** | Reddit, Play Store reviews, X, gaming forums | `sentiment_report.json`: complaints, praise, feature requests |
| 3 | **Trend Spotter** | Reddit, YouTube Trending, X, Google Trends, gaming news | `trends_report.json`: trending topics, game launches, esports events |
| 4 | **Content Strategist** | Reports 1-3 | `content_calendar.json`: 5-7 post ideas with hooks, CTAs, timings |
| 5 | **Delivery Agent** | All four outputs | Formatted MarkdownV2 report sent to Telegram |

Agents 1-3 run in parallel, so a full run takes about 5 minutes.

## What it monitors

**Competitors** (configurable in `.env`): LDPlayer, NoxPlayer, MEmu, Steam, Epic Games, Xbox

**Subreddits:** r/BlueStacks, r/AndroidEmulators, r/AndroidGaming, r/gachagaming, r/gaming, r/pcgaming, r/MobileGaming, r/Genshin_Impact

**Other sources:** YouTube Trending, X / Twitter, Google Trends, Google Play Store reviews (via Apify), XDA Developers forums, game launch calendars

## Why Apify

Reddit threads, Play Store reviews and forum boards render with JavaScript, so a normal web search only returns a link and a short snippet. Apify's RAG Web Browser loads the real page, runs the JavaScript, and returns the full text, so the agents can read what users actually wrote in the comments.

| | Standard web search | Apify RAG Web Browser |
|---|---|---|
| Reddit threads | Post title only | Full comments and replies |
| Play Store | App listing only | Individual reviews with ratings and dates |
| Forums | Thread title only | Full thread body and posts |

At weekly cadence a full run costs roughly $0.10-0.25 in Apify credit, which fits inside the free tier.

## Bundled writing skills

The pipeline ships with nine reference skills for turning the weekly intelligence into published posts:

| Skill | Use it for |
|---|---|
| Humanizer | Removing AI writing patterns before anything goes public |
| Copywriting | Reddit posts, Discord announcements, captions |
| Copy Editing | Multi-pass editing of drafts |
| Content Strategy | Monthly planning, pillars, editorial calendar |
| Social Content | Platform-specific posts for X, Instagram, Facebook, YouTube |
| AI SEO / GEO | Writing posts that surface in AI search answers |
| Marketing Ideas | A library of 139 tactics for brainstorming |
| Marketing Psychology | Writing content that feels authentic, not promotional |
| Launch Strategy | Announcing releases across channels |

## Repo contents

```
clawdia/
├── README.md
├── .env.example
├── .gitignore
├── skill/
│   ├── SKILL.md              # skill definition, architecture, setup, troubleshooting
│   └── references/           # the nine bundled writing skills
└── docs/
    └── Clawdia-Hackathon-Submission.docx
```

<!-- TODO: add skill.yaml, agents/*.yaml and scripts/*.js once they are in the repo -->

## Setup

1. Install OpenClaw.
2. Install the skill: `openclaw skills install bluestacks-community-intelligence.skill`
3. Copy `.env.example` to `.env` and fill in `APIFY_API_TOKEN`, `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`.
4. Test Telegram delivery: `node scripts/test-telegram.js`
5. Test the full pipeline: `node scripts/cron-runner.js --run-now`
6. Turn on the weekly schedule: `pm2 start scripts/cron-runner.js --name clawdia-cron && pm2 save`

The default schedule is every Monday at 07:00 IST (`30 1 * * 1` in UTC).

## Configuration

| Setting | Where | Default |
|---|---|---|
| Competitor list | `COMPETITORS` in `.env` | LDPlayer, NoxPlayer, MEmu, Steam, EpicGames, Xbox |
| Telegram destination | `TELEGRAM_CHAT_ID` in `.env` | none |
| Agent retries | `OPENCLAW_MAX_RETRIES` | 3 |
| Agent timeout | `OPENCLAW_TIMEOUT_SECONDS` | 120 |

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No Telegram message | Wrong chat ID sign | Use a negative ID for groups and channels |
| Apify scrape is empty | Monthly quota used up | Check the Apify console, upgrade or wait for reset |
| Agent 4 output is generic | Agents 1-3 context not passed in | Check the `input_from` references in `skill.yaml` |
| Cron never fires | PM2 not running or timezone mismatch | Run `pm2 list`, check the UTC cron expression |

## Roadmap

- [ ] Discord activity analyzer (Agent 6)
- [ ] App store rating tracker with drop alerts
- [ ] Natural language queries over the report archive
- [ ] Deeper X / Twitter and YouTube monitoring
- [ ] Week-over-week trend analysis
- [ ] Slack delivery alongside Telegram
- [ ] Auto-posting agent with human approval

## Author

Built by [Umang Srivastava](https://www.linkedin.com/in/umang1617/) for the OpenClaw Hackathon, March 2026.
