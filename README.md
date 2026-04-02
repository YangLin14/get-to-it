# Get To It ✅

**An AI-powered personal secretary for people with too many ideas, too little focus — but the real drive to move forward.**

Every morning, it tells you the three most important things to do today. It tracks your momentum. When a new idea tries to pull you off track, it holds you accountable. When you've been stuck too long, it speaks frankly.

> "Other tools help already-disciplined people work more efficiently. Get To It helps motivated-but-easily-distracted people find direction and actually stick with it."

🌏 [中文版說明 README.zh.md](README.zh.md)

---

## Two Versions

|                   | Claude Cowork Skill       | OpenClaw / ClawHub Skill           |
| ----------------- | ------------------------- | ---------------------------------- |
| **Best for**      | Claude Pro subscribers    | Anyone — bring your own AI model   |
| **Platform**      | Claude Desktop App        | OpenClaw (20+ messaging platforms) |
| **AI Model**      | Claude Sonnet             | Any model configured in OpenClaw   |
| **Install**       | Download `.skill` file    | `clawhub install get-to-it`        |
| **Cost**          | Requires Claude Pro       | Free                               |
| **Telegram push** | Manual Bot setup required | Native OpenClaw feature            |

---

## Features

- **🌅 Morning Brief** — Daily Top 3 with reasoning and time estimates, auto-linked to your calendar for free-time calculation
- **⚖️ Momentum-Aware Sorting** — Dynamic priority scoring across six factors: deadline, goal priority, momentum, skip history, time fit, and proximity
- **💡 Idea Capture** — Drop in new ideas anytime; AI scores their relevance to your goals so good ideas aren't lost and distractions don't derail you
- **🧠 Dual-Layer Memory** — Short-term memory (14 days) + permanent semantic long-term memory (TF-IDF vector search)
- **⏱️ Time Tracking & Calibration** — Logs actual time spent; EWMA auto-corrects time estimates per task category
- **🤖 Human-AI Task Collaboration** — Tasks can be delegated to AI agents; auto-detects timeout (30 min) and prompts recovery
- **🎓 ML Priority Learning** — Observes which recommended tasks you actually complete; uses logistic regression to auto-adjust scoring weights
- **🤵‍♀️ Secretary Persona** — Dynamically shifts tone based on stagnation days: warm support → gentle reminder → firm accountability
- **📱 Telegram Output** — `morning --format telegram` outputs MarkdownV2-formatted push notifications to your phone

---

## Installation

### Claude Cowork Skill (Flagship)

Requires the Claude Desktop App with Cowork mode enabled.

1. Go to [Releases](../../releases) and download `get-to-it.skill`
2. Open Claude Desktop App and click the file → **Save Skill**
3. Start a new conversation and say "morning brief" to get started

### OpenClaw / ClawHub Skill (Open)

Requires [OpenClaw](https://github.com/openclaw/openclaw) and Python 3.

```bash
# Install from ClawHub
clawhub install get-to-it

# Or install directly from this repo
openclaw skills install github:YangLin14/get-to-it --path clawhub-skill/get-to-it
```

Once installed, say "morning brief" or "what should I work on today?" in OpenClaw to get started.

**Supported models**: GPT-4o, Gemini 1.5 Pro, Claude Sonnet, and any model you configure in OpenClaw.
GPT-4o / Gemini 1.5 Pro tier works best; mini/flash tier covers core features with slightly reduced persona nuance.

---

## Daily Workflow

```
Morning:   "morning brief"       → Secretary gives you today's Top 3
Working:   "I have an idea..."   → Evaluated + stored; you stay on track
Done:      "finished it"         → Logged, progress updated, next step revealed
Stuck:     "skip, because..."    → Stored in memory, sorting adjusted
Wrapping:  "that's a wrap"       → Today reviewed, mood logged, tomorrow previewed
```

---

## Technical Details

- **Backend**: `gti.py` — 2,400+ line single Python script, pure CLI, JSON output
- **Database**: SQLite WAL mode, 19 tables, data stored locally at `~/.get-to-it.db`
- **Vector Search**: Custom TF-IDF (character n-gram + numpy cosine similarity) — no model downloads, supports CJK
- **Time Calibration**: EWMA (α=0.3) per-category rolling ratio
- **ML Sorting**: Pure numpy logistic regression trained from your completion history
- **Dependencies**: Python 3, numpy (for vector features), icalendar (for calendar integration, optional)
- **Privacy**: All data stored locally on your machine — nothing uploaded to any server

---

## Telegram Push Notifications

```bash
# Output Telegram MarkdownV2 format
python3 gti.py morning --format telegram

# Pipe to curl (requires a Telegram Bot token)
MSG=$(python3 gti.py morning --format telegram)
curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
  -d chat_id="$CHAT_ID" \
  -d parse_mode="MarkdownV2" \
  -d text="$MSG"

# Add to crontab for automatic 8:30 AM delivery on weekdays
30 8 * * 1-5 cd /path/to/gti && bash send_morning.sh
```

OpenClaw users get Telegram push as a native feature — no extra setup needed.

---

## Repo Structure

```
get-to-it/
├── cowork-skill/          # Claude Cowork version
│   └── get-to-it/
│       ├── SKILL.md       # Claude behavior rules
│       ├── scripts/gti.py # Core CLI backend
│       └── references/persona.md
│
├── clawhub-skill/         # OpenClaw / ClawHub version
│   └── get-to-it/
│       ├── SKILL.md       # Model-agnostic behavior rules
│       ├── scripts/gti.py # Core CLI backend (same engine)
│       └── references/persona.md
│
└── docs/
    ├── 使用手冊.html      # Full user guide (Chinese)
    ├── system-diagram.html # System architecture diagram
    └── 技術報告.html      # V2 technical report (Chinese)
```

---

## License

This repo is licensed under the [MIT License](LICENSE).
The ClawHub-published version follows MIT-0 per ClawHub policy.

---

_Built for people with too many ideas, too little focus — but the real drive to move forward._
