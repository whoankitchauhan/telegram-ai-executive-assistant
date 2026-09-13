# Telegram AI Executive Assistant

An autonomous AI assistant that lives inside Telegram and manages your Google Calendar through natural conversation — built with [n8n](https://n8n.io), [Groq](https://groq.com), and the Google Calendar API.

You talk to it like a real assistant: *"schedule a call with Priya tomorrow at 3pm"*, *"what's on my calendar this week?"*, *"move my 5pm to tomorrow morning"* — and it reasons about your calendar, checks for conflicts, remembers your preferences, and only ever confirms an action after it's actually been done.

This is a personal learning project built step-by-step to understand agentic AI workflows, tool-calling LLMs, and event-driven automation — not a polished commercial product. The README below documents it in full, including the mistakes and fixes along the way, because that's most of what I actually learned.

---

## Demo

*(Add a short screen-recording GIF or video link here once you have one — see `/demo`)*

---

## Table of Contents

- [What it does](#what-it-does)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Features](#features)
- [How it works: step by step](#how-it-works-step-by-step)
- [Setup guide](#setup-guide)
- [System prompt](#system-prompt)
- [Known limitations](#known-limitations)
- [Roadmap / build timeline](#roadmap--build-timeline)
- [Lessons learned](#lessons-learned)

---

## What it does

Send the bot a message (or eventually, a voice note) on Telegram, and it can:

- **Read your calendar** — "What's on my calendar this week?", "Am I free tomorrow at 4?"
- **Create events** — checks for conflicts and duplicates *before* booking, asks for missing details instead of guessing
- **Update events** — finds the right event first, checks the new slot for conflicts, then reschedules
- **Delete events** — confirms intent before cancelling anything
- **Remember your preferences** — e.g. "I prefer morning meetings" gets saved and used automatically in future conversations
- **Hold a real conversation** — keeps context across multiple messages, so you don't have to repeat yourself
- **Fail honestly** — if it can't do something (or a tool call fails), it says so plainly instead of pretending

---

## Architecture

```
Telegram (user message)
        │
        ▼
 Telegram Trigger (n8n)
        │
        ▼
 ┌─────────────────────────────────────────────┐
 │                 AI AGENT                     │
 │        (Groq — openai/gpt-oss-20b)           │
 │                                               │
 │   ┌───────────────┐    ┌──────────────────┐ │
 │   │ Simple Memory │    │   Tools:          │ │
 │   │ (short-term,  │    │  - Get many events│ │
 │   │  per-chat)    │    │  - Create event   │ │
 │   └───────────────┘    │  - Update event   │ │
 │                         │  - Delete event   │ │
 │                         │  - Get saved facts│ │
 │                         │  - Save new fact  │ │
 │                         └──────────────────┘ │
 └─────────────────────────────────────────────┘
        │
        ▼
 Send message back to Telegram
```

Everything runs in a **single n8n workflow** — one AI Agent node with six tools attached (two Calendar read/write pairs, two Data Table read/write for long-term memory), a short-term conversation memory buffer, and a Telegram trigger/response pair on either end.

---

## Tech stack

| Component | Tool |
|---|---|
| Automation / orchestration | [n8n](https://n8n.io) (self-hosted via Docker) |
| LLM | [Groq](https://groq.com) — `openai/gpt-oss-20b` |
| Calendar | Google Calendar API (OAuth2) |
| Messaging | Telegram Bot API |
| Long-term memory | n8n Data Tables |
| Short-term memory | n8n Simple Memory (windowed buffer) |
| Local tunnel (dev) | [ngrok](https://ngrok.com) |

---

## Features

- ✅ Natural-language calendar reading, with clean human-readable summaries
- ✅ Conflict detection before every booking — proposes alternative times instead of failing silently
- ✅ Duplicate-event protection
- ✅ Full CRUD on calendar events (create / read / update / delete)
- ✅ Persistent long-term memory of user preferences (e.g. preferred meeting times)
- ✅ Short-term conversational memory (holds context across a multi-turn conversation)
- ✅ Timezone-correct scheduling (explicit ISO 8601 + offset, anchored to real current date/time)
- ✅ Honest capability boundaries — never claims an action succeeded unless a tool actually confirms it
- ⏳ Voice note support (planned — see [Roadmap](#roadmap--build-timeline))
- ⏳ Interactive Telegram buttons for conflict resolution (planned)

---

## How it works: step by step

1. **Telegram Trigger** receives an incoming message.
2. The message text is passed to an **AI Agent** node (LLM: Groq `gpt-oss-20b`).
3. The Agent's **system prompt** (see below) defines its rules: check memory, check calendar for conflicts before writing, never guess dates, never fabricate a success.
4. Based on the message, the Agent decides which tool(s) to call:
   - Reads saved facts about the user (for personalization)
   - Searches the calendar (for read/update/delete requests)
   - Creates, updates, or deletes an event (only after checking availability)
   - Saves a new fact if the user shared a durable preference
5. The Agent's final natural-language reply is sent back to the user via Telegram.

---

## Setup guide

If you want to build this yourself, here's the order that worked:

1. **Create a Telegram bot** via [@BotFather](https://t.me/BotFather), get the API token.
2. **Run n8n locally via Docker**:
   ```bash
   docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
   ```
3. **Expose it publicly with ngrok** (Telegram requires an HTTPS webhook):
   ```bash
   ngrok http 5678
   ```
   Then restart n8n with the ngrok URL set:
   ```bash
   docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n -e N8N_WEBHOOK_URL=https://YOUR-NGROK-URL.ngrok-free.dev/ docker.n8n.io/n8nio/n8n
   ```
4. **Build the base flow**: Telegram Trigger → Send Message, confirm the pipe works end-to-end first.
5. **Add the AI Agent** with a Groq Chat Model, replace the static reply with AI-generated text.
6. **Connect Google Calendar** via OAuth2 (Google Cloud Console → enable Calendar API → create OAuth client → set redirect URI to `<your-n8n-url>/rest/oauth2-credential/callback`).
7. **Add Calendar tools** to the Agent: Get Many Events, Create Event, Update Event, Delete Event — each with a clear tool description and `$fromAI(...)` expressions for AI-controlled parameters.
8. **Add memory**: a Simple Memory node (short-term) + an n8n Data Table with `chat_id` / `fact` columns, exposed as Insert/Get tools (long-term).
9. **Write and iterate on the system prompt** (see below) — this is most of the actual "engineering" in a project like this.
10. **Publish/activate the workflow** so it runs continuously instead of only in test mode.

Full command-by-command notes are in [`/docs/setup-log.md`](./docs/setup-log.md).

---

## System prompt

The full system prompt driving the agent's behavior is in [`/prompts/system-prompt.md`](./prompts/system-prompt.md). It covers:

- Authoritative current date/time and timezone anchoring
- Memory rules (when to read/write saved facts)
- Conflict + duplicate checking before any calendar write
- Exact tool requirements (e.g. Update/Delete need a real `event_id`, always found via search first — never invented)
- Honest failure/capability-boundary behavior
- Reply formatting rules (short, natural, no raw JSON/timestamps)

---

## Known limitations

- No recurring-event management (single events only)
- No voice note support yet
- Conflict alternatives are presented as text, not tappable buttons, yet
- Runs locally via Docker + ngrok — not yet deployed to a persistent server, so it's only reachable while my machine, n8n, and ngrok are all running
- Google OAuth is in "Testing" mode, so the Calendar connection token expires roughly weekly and needs a quick manual reconnect

---

## Roadmap / build timeline

- [x] **v1** — Core pipeline: Telegram ↔ n8n ↔ Groq AI, basic conversational replies
- [x] **v2** — Google Calendar read access (AI can answer schedule questions)
- [x] **v3** — Calendar write access: create events with conflict checking
- [x] **v4** — Long-term memory (saved user facts) + short-term conversation memory
- [x] **v5** — Full CRUD: update and delete events, with search-before-modify safety
- [ ] **v6** — Voice note support (Telegram audio → Groq Whisper → same pipeline)
- [ ] **v7** — Interactive conflict-resolution buttons (tap to pick an alternative time)
- [ ] **v8** — Deploy to a persistent cloud server (remove local-machine dependency)

---

## Lessons learned

A few real things that came up building this, worth noting for anyone doing similar work:

- **Conflict checking has to be a hard rule in the prompt, not a suggestion** — the AI will happily skip it unless told explicitly and repeatedly.
- **Timezone bugs are sneaky** — a model can compute the right local time but still send the wrong `timeZone` field to an API if not told to use an explicit UTC offset.
- **LLMs will confidently claim success even when a tool call did something unexpected** — always verify against the actual source of truth (in this case, Google Calendar itself) rather than trusting the AI's own confirmation message.
- **Token/rate limits matter on free-tier LLM APIs** — a long system prompt plus growing conversation memory can add up fast; trimming the prompt and switching to a smaller model fixed repeated rate-limit errors.
- **Network environment can silently break everything** — a restrictive network (in my case, a college hostel network) can block Docker's outbound HTTPS traffic in a way that looks identical to a misconfiguration, wasting a lot of debugging time before the real cause (the network itself) was identified.

---

## Author

Ankit Chauhan
