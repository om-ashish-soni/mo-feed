# mo-feed

Tech intelligence pipeline for Claude Code. Fetch from 7+ sources. Classify by priority. Display TUI cards. Ingest into a markdown knowledge base.

```
Twitter ──┐
HN ───────┤
GitHub ───┤──→ Classify (P0-P3) ──→ TUI Cards ──→ Knowledge Base
arXiv ────┤
HF ───────┤
Lobsters ─┘
```

## What It Does

**mo-feed** is a Claude Code skill that turns your terminal into a tech intelligence dashboard.

| Phase | What happens |
|-------|-------------|
| **Fetch** | Pulls from Twitter, Hacker News, GitHub trending, arXiv, HuggingFace, Lobsters, Reddit — in parallel |
| **Classify** | Sorts every item into P0-P3 priority tiers using customizable keyword matching |
| **Display** | Renders TUI cards in your terminal, grouped by tier, sorted by engagement |
| **Track** | Maintains a reading list with unread/read status |
| **Ingest** | Writes high-signal items into a persistent markdown knowledge base |
| **Index** | Updates a chronological timeline for date-based recall |

## Install

### From skills.sh
```bash
claude skill add om-ashish-soni/mo-feed
```

### Manual
```bash
# Clone into your Claude Code skills directory
git clone https://github.com/om-ashish-soni/mo-feed.git ~/.claude/skills/mo-feed
```

### Prerequisites
```bash
# Required: headless-twitter for Twitter fetching (Chrome CDP, zero API keys)
npm install -g headless-twitter

# Required: jq for JSON processing
sudo apt install jq   # or: brew install jq
```

## Usage

Just talk to Claude Code:

```
> catch me up                          # Full scan: all sources, all tiers
> what's new in databases              # Topic scan: database-focused
> scan twitter                         # Source scan: Twitter only
> quick feed                           # Headlines only, no ingest
> what do I know about transformers    # Query your knowledge base
> what's pending                       # Show unread items
> mark #3 read                         # Track what you've read
```

Or invoke directly:
```
> /mo-feed
```

## Output Preview

```
━━━ P0: AGENTIC AI ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #1  @AnthropicAI                                   [P0] Agentic AI    │
│       ♥ 2.1k   ↺ 412   ◎ 89            Apr 12, 2026 · 3h ago          │
│                                                                          │
│  Introducing Claude Code skills — reusable agent capabilities           │
│  that extend what Claude can do in your terminal...                      │
│                                                                          │
│  → https://x.com/i/web/status/1234567890                                        │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #2  source: GitHub                                 [P0] Agentic AI    │
│       ⭐ 2.1k stars   🍴 228 forks   lang: Python   Apr 10, 2026 · 2d  │
│                                                                          │
│  alchaincyf/hermes-agent-orange-book — Comprehensive guide              │
│  for Nous Research's agent framework                                     │
│                                                                          │
│  → https://github.com/alchaincyf/hermes-agent-orange-book                       │
└──────────────────────────────────────────────────────────────────────────┘

━━━ P1: DATABASES ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #3  source: Hacker News                            [P1] Databases     │
│       ▲ 203 points   💬 54 comments     Apr 11, 2026 · 1d ago          │
│                                                                          │
│  Show HN: Postgres extension for BM25 full-text search                  │
│                                                                          │
│  → https://github.com/timescale/pg_textsearch                                   │
└──────────────────────────────────────────────────────────────────────────┘

┌─── ACTION ITEMS ──────────────────────────────────────────────────────┐
│  ⭐ hermes-agent-orange-book — Agent framework guide                   │
│  🔧 pg_textsearch — BM25 search in Postgres                           │
└──────────────────────────────────────────────────────────────────────┘

━━━ 42 items | P0: 10 · P1: 8 · P2: 12 · P3: 12 | 5 action items ━━━━━
```

## Knowledge Base

mo-feed writes to `~/.secondmem/knowledge/` — a markdown-native knowledge base with no database, no embeddings, no vector store. Just files.

```
~/.secondmem/
├── knowledge/
│   ├── hierarchy.md              # Root index
│   ├── ai-ml/
│   │   ├── hierarchy.md
│   │   ├── agentic-ai-2026-apr.md
│   │   └── ml-models-2026-apr.md
│   ├── engineering/
│   │   ├── hierarchy.md
│   │   ├── database-internals-2026-apr.md
│   │   └── gpu-hardware-landscape.md
│   └── ...
├── reading-list.md               # Unread/read tracker
├── timeline.md                   # Chronological index
└── logs/
    └── ingest.log
```

**Topic-first, month-scoped.** Files organized by domain, named by month for rolling content. A timeline index gives you chronological access across all topics.

Every ingested item carries:
```
> **Posted:** 2026-04-10 · **Ingested:** 2026-04-12
```

## Customization

### Interest Tiers
Edit the **Priority Tiers** table in `SKILL.md` to match your interests. The defaults cover:
- P0: Agentic AI, Foundational ML
- P1: GPU/Hardware, System Design, Infra
- P2: Startups, Science, OSS
- P3: Everything else

### Key Accounts
Replace the **Key Accounts** section with Twitter handles you want always surfaced.

### New Sources
Add entries to the source-specific fetch table. The classifier works on any text — just feed it JSON.

### Disable Ingest
Say "quick feed" instead of "catch me up" for read-only mode (no KB writes).

## How It Works

1. **headless-twitter** connects to your running Chrome via CDP (Chrome DevTools Protocol) — intercepts Twitter's GraphQL responses directly. No API keys. No browser downloads. Read-only by design (3-layer write protection).

2. **Cross-reference APIs** (HN Algolia, GitHub Search, HuggingFace, Lobsters JSON, arXiv, Reddit) are hit in parallel via curl. No auth needed for any of them.

3. **Classification** is keyword-based regex matching against text + author. Fast, transparent, no ML model needed. You can read the tier table and know exactly why something was classified P0.

4. **Knowledge base** is pure markdown on your filesystem. `hierarchy.md` files act as indexes. Claude navigates them like a developer navigates a codebase. No embedding, no vector DB, no sync pipeline.

## Modes

| Mode | Trigger | Sources | Ingest |
|------|---------|---------|--------|
| Full scan | "catch me up", "mo feed" | All | Yes |
| Topic scan | "what's new in [topic]" | Targeted | Yes |
| Source scan | "scan twitter" | Single | P0/P1 only |
| Quick peek | "quick feed" | Twitter | No |
| Query KB | "what do I know about X" | None (reads KB) | No |
| Ingest only | "remember this" | None | Yes |
| Show unread | "what's pending" | None (reads list) | No |
| Mark read | "read #N" | None | Updates list |

## Requirements

- [Claude Code](https://claude.ai/code) (CLI, desktop, or IDE extension)
- [headless-twitter](https://github.com/om-ashish-soni/headless-twitter) (`npm i -g headless-twitter`)
- [jq](https://jqlang.github.io/jq/) (`apt install jq` / `brew install jq`)
- Chrome/Chromium (for Twitter fetching — uses your existing logged-in session)

## License

MIT

## Author

[Om Ashish Soni](https://github.com/om-ashish-soni)
