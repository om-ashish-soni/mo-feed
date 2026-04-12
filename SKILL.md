---
name: mo-feed
description: >
  Tech intelligence pipeline for developers. Fetches from Twitter, Hacker News, GitHub,
  arXiv, HuggingFace, Lobsters, and Reddit in parallel. Classifies content by customizable
  P0-P3 priority tiers. Renders TUI cards in terminal. Ingests into markdown knowledge base
  (~/.secondmem/knowledge/). Tracks read/unread status. Use when user says "catch me up",
  "what's interesting", "daily digest", "feed me", "mo feed", "tech scan", "what's new",
  "fetch and ingest", or invokes /mo-feed.
---

# mo-feed — Tech Intelligence Pipeline for Developers

One skill. Fetch from everywhere. Classify. Display cards. Ingest to memory. Done.

```
Twitter ──┐
HN ───────┤
GitHub ───┤──→ Classify (P0-P3) ──→ TUI Cards ──→ Knowledge Base Ingest
arXiv ────┤
HF ───────┤
Lobsters ─┘
```

## What It Does

1. **FETCH** — pulls from 7+ sources in parallel (Twitter timeline, HN, GitHub trending, arXiv, HuggingFace, Lobsters, Reddit)
2. **CLASSIFY** — sorts every item into P0-P3 priority tiers using keyword matching
3. **DISPLAY** — renders beautiful TUI cards in terminal, grouped by tier, sorted by engagement
4. **TRACK** — maintains a reading list with unread/read status (`~/.secondmem/reading-list.md`)
5. **INGEST** — writes P0 + high-signal P1 items into a markdown knowledge base (`~/.secondmem/knowledge/`)
6. **INDEX** — updates a chronological timeline for date-based recall (`~/.secondmem/timeline.md`)

## When to Use

- "catch me up" / "what's interesting" / "daily digest" / "feed me"
- "what's new in [topic]" / "scan [source]" / "tech scan"
- "fetch and store" / "fetch and ingest" / "mo feed"
- Any request that implies: get content from internet + show it + remember it

## Prerequisites

```bash
# Required: headless-twitter for Twitter fetching (Chrome CDP, no API keys)
which headless-twitter || npm install -g headless-twitter

# Required: jq for JSON processing
which jq || sudo apt install jq

# Knowledge base directory (auto-created on first run if missing)
mkdir -p ~/.secondmem/knowledge ~/.secondmem/logs
```

## Pipeline — Execute in Order

### Phase 1: FETCH

Fetch from multiple sources in parallel. Default: Twitter timeline + top cross-reference sources.
If user specifies a topic, add targeted searches.

#### Default fetch (no topic specified):
```bash
# Twitter timeline (primary) — requires headless-twitter + logged-in Chrome
headless-twitter twitter timeline '' 50 --lang en --json

# Cross-reference sources (parallel curl — no auth needed)
curl -s "https://hn.algolia.com/api/v1/search?tags=show_hn&numericFilters=points>30" | jq '.hits[0:10]'
curl -s "https://huggingface.co/api/models?sort=trending&limit=10" | jq '.[] | {modelId, likes, pipeline_tag, lastModified}'
curl -s "https://lobste.rs/hottest.json" | jq '.[0:10]'
curl -s "https://api.github.com/search/repositories?q=stars:>50+created:>$(date -d '3 days ago' +%Y-%m-%d)&sort=stars&per_page=10" | jq '.items[] | {full_name, stargazers_count, language, description}'
```

#### Topic-specific fetch (user says "what's new in [topic]"):

| Topic | Twitter Search | Extra Sources |
|-------|---------------|---------------|
| Agentic AI | `"agent framework" OR "agent SDK" OR "MCP" OR "tool-use"` | arXiv cs.AI, HuggingFace, r/LocalLLaMA |
| ML/AI | `"open source model" OR "fine-tuning" OR "inference" OR "transformer"` | arXiv cs.LG, HuggingFace, r/machinelearning |
| GPU/Hardware | `"GPU" OR "CUDA" OR "RISC-V" OR "FPGA" OR "NVIDIA"` | GitHub C/C++/CUDA trending, Lobsters hardware |
| Databases | `"database" OR "B-tree" OR "storage engine" OR "PostgreSQL" OR "SQLite"` | Lobsters databases, HN, r/programming |
| System Design | `"distributed" OR "latency" OR "consensus" OR "architecture"` | Lobsters, HN top, r/experienceddevs |
| Infra | `"microVM" OR "eBPF" OR "kubernetes" OR "Docker" OR "CLI tool"` | GitHub Go/Rust trending, Show HN |
| Startups | `"just shipped" OR "just launched" OR "YC" OR "users"` | Product Hunt, Show HN |
| Frontend | `"React" OR "Next.js" OR "Svelte" OR "CSS" OR "web components"` | GitHub trending JS/TS, r/webdev |
| Security | `"CVE" OR "zero-day" OR "vulnerability" OR "exploit" OR "pentest"` | Lobsters security, r/netsec |
| DevOps | `"terraform" OR "ansible" OR "CI/CD" OR "deployment" OR "SRE"` | HN, Lobsters devops |

#### Source-specific fetch (user says "scan [source]"):

| Source | Command |
|--------|---------|
| Twitter | `headless-twitter twitter timeline '' 50 --lang en --json` |
| Twitter search | `headless-twitter twitter search "QUERY" 30 --lang en --json` |
| Twitter user | `headless-twitter twitter user "@handle" 20 --lang en --json` |
| HN top | `curl -s "https://hn.algolia.com/api/v1/search?tags=front_page" \| jq '.hits[0:15]'` |
| HN Show | `curl -s "https://hn.algolia.com/api/v1/search?tags=show_hn&numericFilters=points>30" \| jq '.hits[0:15]'` |
| GitHub trending | `curl -s "https://api.github.com/search/repositories?q=stars:>50+created:>$(date -d '3 days ago' +%Y-%m-%d)&sort=stars&per_page=15" \| jq '.items[]'` |
| arXiv AI | `curl -s "http://export.arxiv.org/api/query?search_query=cat:cs.AI+OR+cat:cs.LG&sortBy=submittedDate&sortOrder=descending&max_results=10"` |
| HuggingFace | `curl -s "https://huggingface.co/api/models?sort=trending&limit=10" \| jq '.[]'` |
| Lobsters | `curl -s "https://lobste.rs/hottest.json" \| jq '.[0:15]'` |
| Reddit | `curl -s "https://www.reddit.com/r/programming/hot.json?limit=15" -H "User-Agent: mo-feed/1.0" \| jq '.data.children[].data'` |
| r/LocalLLaMA | `curl -s "https://www.reddit.com/r/LocalLLaMA/hot.json?limit=15" -H "User-Agent: mo-feed/1.0" \| jq '.data.children[].data'` |

### Phase 2: CLASSIFY

Classify ALL fetched content into priority tiers using keyword matching.

#### Default Priority Tiers

> **Customization:** Edit the tiers below to match YOUR interests. These defaults are tuned for
> a full-stack developer interested in AI, systems, and infrastructure. Swap keywords, rename
> tiers, add new domains — the classifier just does keyword matching against text + author.

| Tier | Domain | Keywords / Signals |
|------|--------|--------------------|
| **P0** | Agentic AI | agent, MCP, tool-use, memory system, harness, skills, RAG, Claude, LangChain, CrewAI, AutoGen, agentic, deepagent |
| **P0** | Foundational AI/ML | transformer, attention, training, distill, inference, RLHF, DPO, open-source model, vLLM, TRL, GGUF, quantiz, fine-tun, benchmark, LLM, GPT, param, token |
| **P1** | GPU / Hardware | GPU, CUDA, NPU, RISC-V, FPGA, chip, silicon, NVIDIA, AMD, robot, hardware, spacecraft |
| **P1** | System Design | database, B-tree, LSM, consensus, distributed, latency, throughput, architecture, CAP, CRDT, system design, query optim |
| **P1** | Infra Engineering | microVM, sandbox, Docker, Kubernetes, eBPF, observability, CI/CD, CLI tool, build system, Vercel, container |
| **P2** | Startups & Builders | YC, shipped, launched, users, milestone, open source, founder, builder, directory |
| **P2** | Science & Space | SpaceX, rocket, Starship, physics, quantum, space |
| **P2** | OSS & PKM | Obsidian, second brain, knowledge, awesome-list, PKM, open-source |
| **P3** | Everything else | Sort last, still show |

#### Classification Rules
1. Scan text + author for keyword matches
2. Assign highest matching tier (P0 wins over P1)
3. Within each tier, sort by engagement (likes + retweets + points + stars)
4. Remove exact duplicates and RTs of already-shown items
5. Extract links from P0 items as action items

#### Key Accounts (always surface when they appear)

> **Customization:** Replace with accounts YOU follow that are high-signal for your interests.

```
# AI / LLM Engineering
@hwchase17 @AnthropicAI @karpathy @swyx @jxnlco @simonw @tom_doerr

# Systems / Infra / Tooling
@rauchg @cramforce @mitchellh @kelseyhightower @ThePrimeagen @antirez

# AI Research / Models
@_lewtun @vllm_project @huggingface @GoogleDeepMind

# Hardware / Science
@elonmusk @dfrobotcn @LensScientific

# Builders
@kepano @DanielleFong @levelsio @zenorocha
```

### Phase 3: DISPLAY — TUI Cards

ALL content rendered as cards. Never plain tables or bullets for content items.

#### Tweet Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  @author                                        [P0] Agentic AI    │
│       ♥ 10.7k   ↺ 723   ◎ 184          Apr 12, 2026 · 3h ago          │
│                                                                          │
│  Tweet text wrapped at ~70 chars, max 4 lines                            │
│  Truncate with ... if longer                                             │
│                                                                          │
│  -> x.com/i/web/status/ID                                                │
└──────────────────────────────────────────────────────────────────────────┘
```

#### HN Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: Hacker News                            [P1] Databases     │
│       ▲ 203 points   💬 54 comments     Apr 11, 2026 · 1d ago          │
│                                                                          │
│  Show HN: Postgres extension for BM25 full-text search                  │
│                                                                          │
│  -> github.com/timescale/pg_textsearch                                   │
└──────────────────────────────────────────────────────────────────────────┘
```

#### GitHub Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: GitHub                                 [P1] Infra         │
│       ⭐ 1.2k stars   🍴 89 forks   lang: Go       Apr 10, 2026 · 2d   │
│                                                                          │
│  owner/repo — Description text here                                      │
│                                                                          │
│  -> github.com/owner/repo                                                │
└──────────────────────────────────────────────────────────────────────────┘
```

#### arXiv Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: arXiv                                  [P0] ML Research   │
│       📄 cs.AI                                  published: Apr 11, 2026  │
│                                                                          │
│  Paper Title Here                                                        │
│  Authors: First, Second, Third                                           │
│                                                                          │
│  -> arxiv.org/abs/2604.12345                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

#### HuggingFace Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: HuggingFace                            [P0] Models        │
│       ❤️ 542 likes   ⬇ 12.4k downloads   task: text-generation         │
│                                                                          │
│  org/model-name — 7B param, Apache 2.0                                  │
│                                                                          │
│  -> huggingface.co/org/model-name                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Lobsters / Reddit Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: Lobste.rs                              [P1] Systems       │
│       ▲ 45 score   🏷 rust, cli             Apr 12, 2026 · 5h ago      │
│                                                                          │
│  Title of the post                                                       │
│                                                                          │
│  -> lobste.rs/s/abc123                                                   │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Card Rules
1. Box width: 76 inner (78 with border). Fixed.
2. Header: read-status icon + source/author left, `[tier] topic` right-aligned
3. **Date line**: engagement stats LEFT, date + relative time RIGHT (e.g., `Apr 12, 2026 · 3h ago`)
4. Engagement: K/M suffixes for >999
5. Text: wrap ~70 chars, max 4 lines, truncate with ...
6. Link: always `-> URL` at bottom
7. Tier section headers: `━━━ P0: AGENTIC AI ━━━━━━━━━━━━━━━━━━━━━━━━━━━`

#### Date Display Rules
- Twitter: use the `time` field from JSON, convert to `MMM DD, YYYY · Nh ago` or `Nd ago`
- HN: use `created_at` field
- GitHub: use `created_at` field from repo
- Lobsters: use `created_at` field
- HuggingFace: use `lastModified` field
- arXiv: use `published` field
- Relative time: `Xm ago` if <1h, `Xh ago` if <24h, `Xd ago` if <30d, just date if older

#### Read Status Icons
- `○` = unread (not yet read/actioned)
- `●` = read (seen/actioned)

Default: all items start as `○` (unread). When user says "mark read", "done", "read #N", or discusses a specific item in detail, flip it to `●` in the reading list.

#### Action Items Box (after all cards)
```
┌─── ACTION ITEMS ──────────────────────────────────────────────────────┐
│  📄 @author — description -> URL                                      │
│  ⭐ repo — description -> URL                                         │
│  🔧 tool — description -> URL                                         │
└──────────────────────────────────────────────────────────────────────┘
```

#### Summary Footer
```
━━━ 42 items | P0: 10 · P1: 6 · P2: 8 · P3: 18 | ○ 38 unread · ● 4 read | 3 action items ━━━
```

### Phase 3.5: READING LIST — Track Read Status

After displaying cards, save/update a reading list at `~/.secondmem/reading-list.md`.

#### Reading List File Format
```markdown
# Reading List

## Fetched: 2026-04-12 22:30

### Unread (○)
| # | Status | Tier | Source | Author/Repo | Summary | URL | Date |
|---|--------|------|--------|-------------|---------|-----|------|
| 1 | ○ | P0 | GitHub | hermes-agent-orange-book | Agent framework guide | github.com/... | Apr 10 |
| 2 | ○ | P0 | Twitter | @sharbel | AI agents hijacked by websites | x.com/... | Apr 12 |

### Read (●)
| # | Status | Tier | Source | Author/Repo | Summary | URL | Date | Read On |
|---|--------|------|--------|-------------|---------|-----|------|---------|
| 4 | ● | P0 | Twitter | @tom_doerr | 1000+ agent skills | x.com/... | Apr 12 | Apr 12 |
```

#### Reading List Rules
1. **On every fetch**: Append new items to `Unread (○)` section. Don't duplicate (match by URL).
2. **Mark as read**: "mark #N read", "done with #N", "read #N" → move from Unread to Read, add `Read On` date.
3. **Mark batch**: "mark all P0 read", "done with [topic]" → move matching items.
4. **Show unread**: "what's pending", "unread items" → display ○ items as cards.
5. **Show read**: "what did I read", "reading history" → show ● items.
6. **Auto-mark**: When user discusses a specific item in depth, mark it ●.
7. **Cleanup**: Items older than 30 days in Read section → archive to `reading-list-archive.md`.

### Phase 4: INGEST into Knowledge Base

After displaying cards, automatically ingest P0 and P1 items into `~/.secondmem/knowledge/`.

> **Knowledge base structure:** Topic-first directories with month-scoped content files.
> Creates the directory structure automatically on first run.

#### Domain → Directory Mapping

| Tier/Domain | Target Directory | File Strategy |
|-------------|-----------------|---------------|
| Agentic AI | `/ai-ml/` | Create or upsert `agentic-ai-YYYY-MMM.md` |
| Foundational ML | `/ai-ml/` | Create or upsert `ml-models-YYYY-MMM.md` |
| GPU / Hardware | `/engineering/` | Create or upsert `gpu-hardware-landscape.md` |
| Databases / System Design | `/engineering/` | Create or upsert `database-internals-YYYY-MMM.md` |
| Infra | `/engineering/` | Create or upsert into relevant existing file |
| Startups | `/startups/` | Create or upsert based on sub-topic |
| Papers | `/research/` | One file per significant paper |
| People insights | `/people-insights/` | Group by person |
| Frontend | `/engineering/` | Create or upsert `frontend-YYYY-MMM.md` |
| Security | `/engineering/` | Create or upsert `security-YYYY-MMM.md` |

#### Ingest Rules
1. Read target directory's `hierarchy.md` first (create if missing)
2. Check if relevant file exists — upsert if yes, create if no
3. Use month-scoped filenames for rolling content (`database-internals-2026-apr.md`)
4. Write content in structured markdown format (Topic, Source, Ingested, Tags, sections)
5. Tweet format: `> "text" — @author, date` + "Why this matters" + extracted links
6. Repo format: name, stars, language, description, why it matters
7. Paper format: title, authors, key claims, link
8. **Every item MUST carry per-item dates** in the source blockquote:
   ```
   > **Posted:** YYYY-MM-DD · **Ingested:** YYYY-MM-DD
   ```
   - **Posted** = original publish/tweet date (from API `time`, `created_at`, or page date)
   - **Ingested** = today's date when written to knowledge base
   - Place immediately after the engagement line in the blockquote
9. Update directory `hierarchy.md` after writes
10. Update root `hierarchy.md` if new files created
11. Update `~/.secondmem/timeline.md` — append new items to current month section
12. Log to `~/.secondmem/logs/ingest.log`
13. Max 1116 lines per file — split if exceeded
14. Cross-reference new files with related existing files (3-8 refs)

#### Date Organization Strategy
- **Primary axis: topic-first** — files live in topic directories (`/ai-ml/`, `/engineering/`)
- **Secondary axis: month-scoped filenames** — rolling content uses `topic-YYYY-mmm.md`
- **Chronological index: `~/.secondmem/timeline.md`** — reverse-chronological index for date-based recall
- "What did I learn last week?" → read `timeline.md`
- "What do I know about databases?" → read topic directory
- Both paths lead to the same content files — timeline is an index, not a copy

#### Timeline Index Format (`~/.secondmem/timeline.md`)
```markdown
## 2026-04 (April)

| Date | Topic | Title | File | Source |
|------|-------|-------|------|--------|
| Apr 12 | Agentic AI | Agent web hijacking | ai-ml/agentic-ai-2026-apr.md | @sharbel |
| Apr 11 | Databases | Postgres BM25 extension | engineering/database-internals-2026-apr.md | timescale |
```
- Grouped by month, reverse chronological within each month
- One row per ingested item (not per file)
- Keep last 6 months; archive older to `timeline-archive-YYYY.md`

#### What gets ingested vs skipped
- **Always ingest**: P0 items, P1 items with significant engagement (>50 points/likes)
- **Selectively ingest**: P2 items that are repos, papers, or tool launches
- **Skip**: P3 items, engagement bait, generic motivation, duplicate RTs
- **Always extract**: Links from P0 tweets → separate entries for papers/repos

#### Knowledge Base Init (first run)
If `~/.secondmem/knowledge/hierarchy.md` doesn't exist, create the base structure:
```bash
mkdir -p ~/.secondmem/knowledge/{ai-ml,engineering,startups,research,people-insights}
mkdir -p ~/.secondmem/logs
# Create root hierarchy.md and per-directory hierarchy.md files
```

### Phase 5: CONFIRM

After all phases complete, show summary:

```
━━━ mo-feed complete ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Fetched:  Twitter (50) + HN (10) + GitHub (10) + Lobsters (10) = 80 items
Unique:   62 after dedup
Classified: P0: 12 · P1: 18 · P2: 14 · P3: 18
Ingested: 24 items into knowledge base (12 P0 + 12 P1)
Files:    engineering/database-internals-2026-apr.md (updated)
          ai-ml/agentic-ai-2026-apr.md (created)
Reading:  24 new items added to reading list (○ unread)
Action:   5 links to check
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Full scan** | "catch me up", "daily digest", "mo feed" | All sources, all tiers, full ingest |
| **Topic scan** | "what's new in [topic]" | Topic-specific Twitter search + matching cross-refs |
| **Source scan** | "scan twitter", "scan HN" | Single source, all tiers, ingest P0/P1 |
| **Quick peek** | "quick feed", "headlines" | Twitter timeline only, cards only, no ingest |
| **Query KB** | "what do I know about [topic]" | Read from knowledge base, no fetch |
| **Ingest only** | "ingest this", "remember this" | No fetch, write provided content to KB |
| **Show unread** | "what's pending", "unread" | Show ○ items from reading-list.md as cards |
| **Show read** | "what did I read", "reading history" | Show ● items from reading-list.md |
| **Mark read** | "read #N", "done with #N" | Move item from ○ to ● in reading-list.md |
| **Mark batch** | "mark all P0 read", "done with [topic]" | Move matching tier/topic items to ● |

## Agent Instructions

1. ALWAYS fetch with `--json` for Twitter (for classification)
2. Run cross-reference fetches in PARALLEL (independent curl calls)
3. Classify ALL items before displaying any cards
4. Display cards grouped by tier, sorted by engagement within tier
5. After displaying: ingest P0 + high-engagement P1 into knowledge base
6. Every ingested item MUST have `> **Posted:** YYYY-MM-DD · **Ingested:** YYYY-MM-DD`
7. Update hierarchy.md files after ingest
8. Append new items to `~/.secondmem/timeline.md` under the current month section
9. Log all operations to `~/.secondmem/logs/ingest.log`
10. Save catch-up file to `~/Documents/mo-feed-catchup-YYYY-MM-DD.md` for >20 items
11. Default to `--lang en` for Twitter
12. If a fetch fails (timeout, no Chrome), skip that source and continue with others
13. Never block the whole pipeline on one source failure
14. On first run, create `~/.secondmem/` directory structure if it doesn't exist

## Customization Guide

### Adding Your Own Interest Tiers
Edit the **Priority Tiers** table in Phase 2. Add keywords that matter to you:

```markdown
| **P0** | Your Domain | keyword1, keyword2, keyword3 |
```

### Adding Your Key Accounts
Replace the **Key Accounts** list with Twitter handles you want to always surface.

### Adding New Sources
Add a new entry to the **Source-specific fetch** table with the API/curl command.
The classifier works on any text content — just feed it JSON with a `text` or `title` field.

### Changing the Knowledge Base Path
The default knowledge base path is `~/.secondmem/knowledge/`. To change it,
update all references to `~/.secondmem/` in this file.

### Disabling Ingest
For a read-only feed (no knowledge base writes), use **Quick peek** mode:
say "quick feed" or "headlines" instead of "catch me up".
