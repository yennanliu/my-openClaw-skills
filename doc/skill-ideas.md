# OpenClaw Skills & Plugin Ideas

**Platform**: [OpenClaw](https://openclaw.ai/) — open-source personal AI assistant that runs locally,
accessible via WhatsApp, Telegram, Discord, Slack, Signal, and iMessage. Supports Claude, GPT, and local models.

---

## What Makes a Good OpenClaw Skill

A great OpenClaw skill should:
- Solve a real, recurring pain point (not a one-off task)
- Leverage OpenClaw's unique strengths: local execution, persistent memory, chat access, browser/file/shell control
- Work autonomously (trigger on a schedule, event, or message) — not just on demand
- Produce output that lands somewhere useful: a message, a file, a calendar event, a commit

---

## Skill Ideas by Category

---

### 1. Developer Productivity

**`git-standup`**
Every morning, scan recent commits across local repos and compose a concise standup summary ("Yesterday I…"). Send it via Slack or iMessage on a cron schedule.

**`pr-watchdog`**
Monitor open GitHub PRs for review requests, stale state (no activity in N days), or CI failures. Ping the user proactively over chat.

**`log-triage`**
Watch a log file or a directory for ERROR/WARN patterns. When a new exception appears, deduplicate it, look up the stack trace, and send a one-line summary with a suggested fix.

**`doc-writer`**
On demand: read a file or function, generate a docstring or README section, and write it back. Triggered by a chat message like "document src/utils.py".

**`deploy-narrator`**
After a `git push`, poll the CI/CD pipeline (GitHub Actions, CircleCI) and narrate status updates in chat until the build passes or fails.

---

### 2. Email & Communication

**`inbox-brief`**
Each morning, summarize unread emails by sender and urgency. Highlight anything needing a reply today. Draft suggested responses for the top 3.

**`newsletter-digest`**
Detect newsletter emails (Substack, Morning Brew, etc.), extract key points, and compile a weekly 5-bullet digest saved to a markdown file or sent over chat.

**`tone-checker`**
Before sending a drafted email or Slack message (passed via chat), analyze tone and flag if it reads as passive-aggressive, unclear, or overly terse.

**`follow-up-radar`**
Scan sent emails for threads with no reply after N days. Remind the user and optionally draft a follow-up.

---

### 3. Finance & Investment

**`market-open-brief`**
At 9:25 AM on trading days, pull prices for a watchlist, check overnight futures, and send a 5-bullet market brief via iMessage or Telegram.

**`portfolio-pulse`**
Weekly: calculate unrealized P&L on tracked positions, compare to benchmarks (SPY, QQQ), and flag any position that has drifted beyond a set allocation threshold.

**`expense-categorizer`**
Parse bank statement CSV exports, categorize transactions using LLM inference, and output a monthly spending breakdown by category.

**`bill-sentinel`**
Scan emails for invoices and bill-due notices, extract amounts and due dates, and add them to a calendar or send a reminder 3 days before.

---

### 4. Knowledge & Research

**`research-saver`**
On demand: given a topic or URL in chat, fetch content, summarize it, and save a structured note to an Obsidian vault (or a local markdown folder).

**`podcast-recap`**
Given a podcast RSS feed or YouTube link, download the transcript (via Whisper or auto-captions), summarize key takeaways, and save to notes.

**`reading-list-tracker`**
Monitor a Pocket, Instapaper, or plain-text reading list. Weekly: summarize unread articles and suggest the top 3 to read based on topics the user has engaged with most.

**`concept-explainer`**
Triggered by chat: "explain [X] like I work in [Y field]". Pulls context from memory (user's background) to tailor the explanation and optionally saves it to a personal wiki.

---

### 5. Health & Daily Routines

**`morning-brief`**
Aggregate: today's calendar, weather, top 3 emails, a motivational or interesting fact, and pending tasks. Delivered at a set time via preferred chat app.

**`focus-block`**
Start a focus session: set a timer, close distracting apps (via shell commands), and send a "session started" message. At the end, ask for a one-line progress note and log it.

**`habit-logger`**
Accept simple habit check-ins via chat ("done: workout, meditation"). Maintain a streak log. Weekly: send a habit adherence summary.

**`sleep-journal`**
Each morning: ask "how did you sleep?" via chat, log the response with a timestamp, and after 2 weeks, surface patterns (e.g., "Your best sleep follows days without late coffee").

---

### 6. Content & Social

**`post-drafter`**
Given a rough idea via chat, draft a LinkedIn post, tweet thread, or blog intro. Optionally save drafts to a file and schedule posting via the platform's API.

**`content-repurposer`**
Take a blog post or long-form note and produce: a Twitter thread, a LinkedIn summary, and a TL;DR for a newsletter. All saved to an output folder.

**`youtube-notes`**
Given a YouTube URL, fetch the transcript, extract key points and timestamps, and save structured notes locally.

---

### 7. Home & Personal Automation

**`smart-home-voice`**
Bridge chat messages to Home Assistant API: "turn off living room lights", "set thermostat to 70". Translates natural language to HA service calls.

**`grocery-list-builder`**
Parse a meal plan (from a note or chat), extract ingredients, deduplicate against a pantry list, and output a sorted grocery list (optionally sent via Telegram).

**`travel-prep`**
Given a trip destination and dates, compile: weather forecast, packing checklist, flight check-in reminders, and local time zone offset — all in one formatted message.

---

## Prioritization Matrix

| Skill | Effort | Daily Value | Uniqueness to OpenClaw |
|---|---|---|---|
| `morning-brief` | Low | High | Medium |
| `git-standup` | Low | High | High |
| `inbox-brief` | Medium | High | Medium |
| `pr-watchdog` | Medium | High | High |
| `market-open-brief` | Low | High | Medium |
| `log-triage` | Medium | Medium | High |
| `research-saver` | Low | Medium | High |
| `focus-block` | Low | Medium | High |
| `post-drafter` | Low | Medium | Low |
| `smart-home-voice` | High | Medium | High |

---

## Recommended Starting Points

1. **`morning-brief`** — High impact, low complexity, good showcase of OpenClaw's cron + multi-source aggregation
2. **`git-standup`** — Developers use this daily; demonstrates file/shell access + scheduled delivery
3. **`inbox-brief`** — Uses the existing Gmail MCP integration; immediately useful
4. **`market-open-brief`** — Builds on `us-stock-analysis` skills already in this repo; strong synergy
5. **`research-saver`** — Showcases memory + file write + web fetch in one simple skill

---

## Notes on Skill Architecture

Each skill typically consists of:
- A **trigger** (cron expression, chat keyword, or event hook)
- A **data-fetch** step (shell, browser, MCP tool, or API call)
- A **reasoning** step (Claude prompt to summarize, categorize, or decide)
- An **output** step (chat message, file write, calendar event, or API call)

OpenClaw's persistent memory means skills can get smarter over time — remembering preferences, learning patterns, and reducing noise as they accumulate context about the user.
