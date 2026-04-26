---
name: mo-feed
description: >
  Unified tech intelligence pipeline for developers. Fetches from Twitter, Instagram,
  LinkedIn, YouTube (via headless-* Chrome CDP skills), Hacker News, GitHub, arXiv,
  HuggingFace, Lobsters, and Reddit in parallel. Classifies content by customizable
  P0-P3 priority tiers. Renders TUI cards in terminal. Ingests into markdown knowledge base
  (~/.secondmem/knowledge/). Tracks read/unread status. Use when user says "catch me up",
  "what's interesting", "daily digest", "feed me", "om feed", "tech scan", "what's new",
  "fetch and ingest", or invokes /mo-feed. Also triggers on "what's trending", "scan twitter",
  "scan instagram", "scan linkedin", "scan youtube", "scan HN", "scan github", or any
  combination of fetch + store intent.
---

# mo-feed — Unified Tech Intelligence Pipeline

One skill. Fetch from everywhere — social, dev forums, papers, models. Classify.
Display cards. Ingest to memory. Done.

```
Twitter ────┐
Instagram ──┤   (headless-* Chrome CDP intercept, read-only)
LinkedIn ───┤
YouTube ────┤
HN ─────────┤──→ Classify (P0-P3) ──→ TUI Cards ──→ secondmem ingest
GitHub ─────┤
arXiv ──────┤
HF ─────────┤
Lobsters ───┤
Reddit ─────┘
```

## When to Use

- "catch me up" / "what's interesting" / "daily digest" / "feed me"
- "what's new in [topic]" / "scan [source]" / "tech scan"
- "fetch and store" / "fetch and ingest" / "om feed"
- Any request that implies: get content from internet + show it + remember it

## Pipeline — Execute in Order

### Phase 1: FETCH

Fetch from multiple sources in parallel. Default: Twitter timeline + top cross-reference sources.
If user specifies a topic, add targeted searches.

#### Default fetch (no topic specified):
```bash
# Social — headless-* Chrome CDP skills (read-only, zero API keys)
# All four use positional args: SOURCE MODE QUERY LIMIT [--lang en] [--json]
# Pass '' for QUERY when the mode doesn't need one. Tech filter is ON by default
# for Instagram and YouTube; pass --no-filter to disable.
headless-twitter   twitter   timeline '' 50 --lang en --json
headless-instagram instagram explore  '' 20 --json
headless-linkedin  linkedin  feed     '' 20 --lang en --json
headless-youtube   youtube   trending '' 15 --json

# Dev forums + APIs (parallel curl)
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

#### Source-specific fetch (user says "scan [source]"):

| Source | Command |
|--------|---------|
| Twitter | `headless-twitter twitter timeline '' 50 --lang en --json` |
| Twitter search | `headless-twitter twitter search "QUERY" 30 --lang en --json` |
| Twitter user | `headless-twitter twitter user "@handle" 20 --lang en --json` |
| Instagram explore | `headless-instagram instagram explore '' 25 --json` |
| Instagram reels | `headless-instagram instagram reels '' 20 --json` |
| Instagram hashtag | `headless-instagram instagram hashtag "TAG" 25 --json` |
| Instagram account | `headless-instagram instagram account "HANDLE" 15 --json` |
| LinkedIn feed | `headless-linkedin linkedin feed '' 25 --lang en --json` |
| LinkedIn search | `headless-linkedin linkedin search "QUERY" 25 --json` |
| LinkedIn profile | `headless-linkedin linkedin profile "PROFILE_SLUG" 15 --json` |
| LinkedIn company | `headless-linkedin linkedin company "COMPANY_SLUG" 15 --json` |
| LinkedIn hashtag | `headless-linkedin linkedin hashtag "TAG" 20 --json` |
| YouTube trending | `headless-youtube youtube trending '' 20 --json` |
| YouTube shorts | `headless-youtube youtube shorts '' 20 --json` |
| YouTube search | `headless-youtube youtube search "QUERY" 20 --json` |
| YouTube channel | `headless-youtube youtube channel "@handle" 15 --json` |
| HN top | `curl -s "https://hn.algolia.com/api/v1/search?tags=front_page" \| jq '.hits[0:15]'` |
| HN Show | `curl -s "https://hn.algolia.com/api/v1/search?tags=show_hn&numericFilters=points>30" \| jq '.hits[0:15]'` |
| GitHub trending | `curl -s "https://api.github.com/search/repositories?q=stars:>50+created:>$(date -d '3 days ago' +%Y-%m-%d)&sort=stars&per_page=15" \| jq '.items[]'` |
| arXiv AI | `curl -s "http://export.arxiv.org/api/query?search_query=cat:cs.AI+OR+cat:cs.LG&sortBy=submittedDate&sortOrder=descending&max_results=10"` |
| HuggingFace | `curl -s "https://huggingface.co/api/models?sort=trending&limit=10" \| jq '.[]'` |
| Lobsters | `curl -s "https://lobste.rs/hottest.json" \| jq '.[0:15]'` |
| Reddit | `curl -s "https://www.reddit.com/r/programming/hot.json?limit=15" -H "User-Agent: td/1.0" \| jq '.data.children[].data'` |
| r/LocalLLaMA | `curl -s "https://www.reddit.com/r/LocalLLaMA/hot.json?limit=15" -H "User-Agent: td/1.0" \| jq '.data.children[].data'` |

#### Headless-* notes (Chrome CDP)

All four `headless-*` skills share architecture: Chrome on port 9222 (reuse existing logged-in session), GraphQL/JSON intercept, 3-layer read-only enforcement (zero mutations).

- **CLI shape**: every headless-* skill takes positional args `SOURCE MODE QUERY LIMIT [OPTIONS]` — pass `''` for QUERY when the mode doesn't need one. There is no `--max` flag.
- **headless-youtube** has anti-doom-scroll caps (default 15 results, hard max 50, default 4 scrolls, hard max 8, force-muted video). Tech filter is on by default — pass `--no-filter` only if the user explicitly wants generic videos. `--max-scrolls N` is the only knob; never raise it past 8.
- **headless-instagram** has tech-intro detection on by default — pass `--no-filter` only if the user explicitly wants generic reels.
- **headless-linkedin** supports `--reuse-tab` to attach to an already-open LinkedIn tab (best anti-bot path). Use it when the user reports LinkedIn challenge-walling automation.
- If Chrome isn't running on port 9222, skip the headless source and continue with the rest. Never block the pipeline waiting for a browser.

### Phase 2: CLASSIFY

Classify ALL fetched content into priority tiers.

#### Priority Tiers

| Tier | Domain | Keywords / Signals |
|------|--------|--------------------|
| **P0** | Agentic AI | agent, MCP, tool-use, memory system, harness, skills, RAG, Claude, LangChain, CrewAI, AutoGen, agentic, deepagent |
| **P0** | Foundational AI/ML | transformer, attention, training, distill, inference, RLHF, DPO, open-source model, vLLM, TRL, GGUF, quantiz, fine-tun, benchmark, LLM, GPT, param, token |
| **P1** | GPU / Hardware | GPU, CUDA, NPU, RISC-V, FPGA, chip, silicon, NVIDIA, AMD, robot, hardware, spacecraft |
| **P1** | System Design | database, B-tree, LSM, consensus, distributed, latency, throughput, architecture, CAP, CRDT, system design, query optim |
| **P1** | Infra Engineering | microVM, sandbox, Docker, Kubernetes, eBPF, observability, CI/CD, CLI tool, build system, Vercel, container |
| **P2** | Startups & Builders | YC, shipped, launched, users, milestone, open source, founder, Indian startup, builder, directory |
| **P2** | Science & Space | SpaceX, rocket, Starship, physics, quantum, space, ferrofluid |
| **P2** | OSS & PKM | Obsidian, second brain, knowledge, awesome-list, PKM, open-source |
| **P3** | Everything else | Sort last, still show |

#### Classification Rules
1. Scan text + author for keyword matches
2. Assign highest matching tier (P0 wins over P1)
3. Within each tier, sort by engagement (likes + retweets + points + stars)
4. Remove exact duplicates and RTs of already-shown items
5. Extract links from P0 items as action items

#### Key Accounts (always surface when they appear)
```
@hwchase17 @AnthropicAI @karpathy @swyx @jxnlco @rauchg @cramforce
@mitchellh @kelseyhightower @ThePrimeagen @antirez @_lewtun @vllm_project
@huggingface @elonmusk @kepano @DanielleFong @simonw @levelsio @tom_doerr
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
│  Truncate with … if longer                                               │
│                                                                          │
│  → https://x.com/i/web/status/ID                                        │
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
│  → https://github.com/timescale/pg_textsearch                            │
└──────────────────────────────────────────────────────────────────────────┘
```

#### GitHub Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: GitHub                                 [P1] Infra         │
│       ⭐ 1.2k stars   🍴 89 forks   lang: Go        Apr 10, 2026 · 2d ago │
│                                                                          │
│  owner/repo — Description text here                                      │
│                                                                          │
│  → https://github.com/owner/repo                                         │
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
│  → https://arxiv.org/abs/2604.12345                                      │
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
│  → https://huggingface.co/org/model-name                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Instagram Reel Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: Instagram                              [P0] Agentic AI    │
│       ❤ 4.2k   💬 132   ▶ 89k views     Apr 12, 2026 · 6h ago          │
│                                                                          │
│  @author — caption (first ~60 chars), tech-intro detected ✓            │
│                                                                          │
│  → https://www.instagram.com/reel/REEL_ID/                               │
└──────────────────────────────────────────────────────────────────────────┘
```

#### LinkedIn Post Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: LinkedIn                               [P1] Infra         │
│       👍 312 reactions   💬 28 comments   ↺ 14    Apr 11, 2026 · 1d ago │
│                                                                          │
│  Author Name — first ~140 chars of post text                            │
│                                                                          │
│  → https://www.linkedin.com/feed/update/urn:li:activity:ID/             │
└──────────────────────────────────────────────────────────────────────────┘
```

#### YouTube Video / Short Card
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #N  source: YouTube                                [P0] ML Research   │
│       ▶ 142k views   👍 6.1k   ⏱ 12:04           Apr 10, 2026 · 2d ago │
│                                                                          │
│  Channel — Video title here (truncated to ~70 chars)                    │
│                                                                          │
│  → https://www.youtube.com/watch?v=VIDEO_ID                              │
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
│  → https://lobste.rs/s/abc123                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Card Rules
1. Box width: 76 inner (78 with border). Fixed.
2. Header: read-status icon + source/author left, `[tier] topic` right-aligned
3. **Date line**: engagement stats LEFT, date + relative time RIGHT (e.g., `Apr 12, 2026 · 3h ago`)
4. Engagement: K/M suffixes for >999
5. Text: wrap ~70 chars, max 4 lines, truncate with …
6. Link: always `→ https://FULL_URL` at bottom (MUST include `https://` prefix so terminals render clickable links)
7. Tier section headers: `━━━ P0: AGENTIC AI ━━━━━━━━━━━━━━━━━━━━━━━━━━━`

#### Date Display Rules
- Twitter: use the `time` field from JSON, convert to `MMM DD, YYYY · Nh ago` or `Nd ago`
- Instagram: use `taken_at` (epoch sec) → format
- LinkedIn: use `postedAt` / `created` (epoch ms) → format
- YouTube: use `publishedTimeText` (already relative) or `publishDate` (absolute)
- HN: use `created_at` field
- GitHub: use `created_at` field from repo
- Lobsters: use `created_at` field
- HuggingFace: use `lastModified` field
- arXiv: use `published` field
- Relative time: `Xm ago` if <1h, `Xh ago` if <24h, `Xd ago` if <30d, just date if older

#### Read Status Icons
- `○` = unread (not yet read/actioned by Om)
- `●` = read (Om has seen/actioned this item)

Default: all items start as `○` (unread). When Om says "mark read", "done", "read #N", or discusses a specific item in detail, flip it to `●` in the reading list.

#### Action Items Box (after all cards)
```
┌─── ACTION ITEMS ──────────────────────────────────────────────────────┐
│  📄 @author — description → https://FULL_URL                            │
│  ⭐ repo — description → https://FULL_URL                               │
│  🔧 tool — description → https://FULL_URL                               │
└──────────────────────────────────────────────────────────────────────┘
```

#### Summary Footer
```
━━━ 42 items | P0: 10 · P1: 6 · P2: 8 · P3: 18 | 3 action items ━━━━━
```

### Phase 3.5: READING LIST — Track Read Status

After displaying cards, save a reading list to `~/.secondmem/reading-list.md`. This file is the persistent read/unread tracker.

#### Reading List File Format (`~/.secondmem/reading-list.md`)

```markdown
# Om's Reading List

## Fetched: 2026-04-12 22:30

### Unread (○)
| # | Status | Tier | Source | Author/Repo | Summary | URL | Date |
|---|--------|------|--------|-------------|---------|-----|------|
| 1 | ○ | P0 | GitHub | hermes-agent-orange-book | Nous Research agent guide | github.com/... | Apr 10 |
| 2 | ○ | P0 | Twitter | @sharbel | AI agents hijacked by websites | x.com/... | Apr 12 |
| 3 | ○ | P1 | Lobsters | nockawa | DB engine in C# | nockawa.github.io/... | Apr 12 |

### Read (●)
| # | Status | Tier | Source | Author/Repo | Summary | URL | Date | Read On |
|---|--------|------|--------|-------------|---------|-----|------|---------|
| 4 | ● | P0 | Twitter | @tom_doerr | 1000+ agent skills | x.com/... | Apr 12 | Apr 12 |
```

#### Reading List Rules

1. **On every fetch**: Append new items to `Unread (○)` section. Don't duplicate items already in the list (match by URL).
2. **Mark as read**: When Om says "mark #N read", "done with #N", "read #N", or discusses an item in depth → move from `Unread` to `Read` section, add `Read On` date.
3. **Mark batch read**: "mark all P0 read", "done with agentic AI" → move matching items.
4. **Show unread**: When Om says "what's pending", "unread items", "what haven't I read" → read `~/.secondmem/reading-list.md` and display only `○` items as cards.
5. **Show read**: When Om says "what did I read", "reading history" → show `●` items.
6. **Auto-mark read**: When Om explicitly discusses a specific item ("tell me more about #3", "open that hermes repo"), mark it `●`.
7. **Cleanup**: Items older than 30 days in `Read` section can be archived to `~/.secondmem/reading-list-archive.md`.

#### Reading List Integration with Cards

When displaying cards from the reading list (not a fresh fetch), show read status:
```
┌──────────────────────────────────────────────────────────────────────────┐
│ ○ #1  @sharbel                                      [P0] Agentic AI    │
│       ♥ 986   ↺ 432   ◎ 68             Apr 12, 2026 · 3h ago          │
│                                                                          │
│  AI agents browsing the web can be secretly hijacked                     │
│                                                                          │
│  → https://x.com/i/web/status/2043009361321087291                        │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│ ● #4  @tom_doerr                                    [P0] Agentic AI    │
│       ♥ 35   ↺ 2   ◎ 2                 Apr 12, 2026 · 5h ago          │
│                                                                          │
│  Collection of 1000+ official agent skills                               │
│                                                                          │
│  → https://x.com/i/web/status/2043337768021934091                        │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Summary Footer with Read Stats
```
━━━ 77 items | P0: 15 · P1: 8 · P2: 15 · P3: 39 | ○ 73 unread · ● 4 read | 15 action items ━━━
```

### Phase 4: INGEST into secondmem

After displaying cards, automatically ingest P0 and P1 items into `~/.secondmem/knowledge/`.

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
| Video tutorials (YouTube) | `/learning/` | Create or upsert `video-notes-YYYY-mmm.md`, link to channel |
| Reels / short-form (Instagram) | `/learning/` | Append to `short-form-YYYY-mmm.md`, dedup by reel ID |
| LinkedIn long-form posts | `/people-insights/` | Group by author, otherwise topic dir if it's a launch/ship |

#### Ingest Rules
1. Read target directory's `hierarchy.md` first
2. Check if relevant file exists — upsert if yes, create if no
3. Use month-scoped filenames for rolling content (`database-internals-2026-apr.md`)
4. Write content in secondmem format (Topic, Source, Ingested, Tags, structured sections)
5. Tweet format: `> "text" — @author, date` + "Why this matters" + extracted links
6. Repo format: name, stars, language, description, why it matters
7. Paper format: title, authors, key claims, link
8. **Every item MUST carry per-item dates** in the source blockquote:
   ```
   > **Posted:** YYYY-MM-DD · **Ingested:** YYYY-MM-DD
   ```
   - **Posted** = original publish/tweet date (from API `time`, `created_at`, or page date)
   - **Ingested** = today's date when we write it to secondmem
   - Place this line immediately after the engagement line in the blockquote
9. Update directory `hierarchy.md` after writes
10. Update root `hierarchy.md` if new files created
11. Update `~/.secondmem/timeline.md` — append new items to the current month section
12. Log to `~/.secondmem/logs/ingest.log`
13. Max 1116 lines per file — split if exceeded
14. Cross-reference new files with related existing files (3-8 refs)

#### Date Organization Strategy
- **Primary axis: topic-first** — files live in topic directories (`/ai-ml/`, `/engineering/`)
- **Secondary axis: month-scoped filenames** — rolling content uses `topic-YYYY-mmm.md` (e.g., `agentic-ai-2026-apr.md`)
- **Chronological index: `~/.secondmem/timeline.md`** — a flat reverse-chronological index for date-based recall
- When user asks "what did I learn last week" or "recent ingests" → read `timeline.md`
- When user asks "what do I know about databases" → read topic directory
- Both paths lead to the same content files — timeline is an index, not a copy

#### Timeline Index Format (`~/.secondmem/timeline.md`)
```markdown
## 2026-04 (April)

| Date | Topic | Title | File | Source |
|------|-------|-------|------|--------|
| Apr 12 | Agentic AI | Agent web hijacking | ai-ml/agentic-ai-2026-apr.md | @sharbel |
| Apr 12 | Databases | B-Tree depth analysis | engineering/database-internals-2026-apr.md | @BenjDicken |
| Apr 11 | GPU/HW | Vercel Sandbox fastest microVM | ai-ml/agentic-ai-2026-apr.md | @rauchg |
```
- Grouped by month, reverse chronological within each month
- One row per ingested item (not per file)
- Keep last 6 months; archive older months to `timeline-archive-YYYY.md`

#### What gets ingested vs skipped
- **Always ingest**: P0 items, P1 items with significant engagement (>50 points/likes)
- **Selectively ingest**: P2 items that are repos, papers, or tool launches
- **Skip**: P3 items, engagement bait, generic motivation, duplicate RTs
- **Always extract**: Links from P0 tweets → separate entries for papers/repos

### Phase 5: CONFIRM

After all phases complete, show summary:

```
━━━ mo-feed complete ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Fetched:  Twitter (50) + HN (10) + GitHub (10) + Lobsters (10) = 80 items
Unique:   62 after dedup
Classified: P0: 12 · P1: 18 · P2: 14 · P3: 18
Ingested: 24 items into secondmem (12 P0 + 12 P1)
Files:    engineering/database-internals-2026-apr.md (updated)
          ai-ml/agentic-ai-2026-apr.md (created)
          engineering/gpu-hardware-landscape.md (updated)
Action:   5 links to check
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Full scan** | "catch me up", "daily digest", "om feed" | All sources, all tiers, full ingest |
| **Topic scan** | "what's new in [topic]" | Topic-specific Twitter search + matching cross-refs |
| **Source scan** | "scan twitter", "scan instagram", "scan linkedin", "scan youtube", "scan HN" | Single source, all tiers, ingest P0/P1 |
| **Social scan** | "scan socials", "social feed" | All four headless-* sources only (twitter+ig+li+yt), no curl APIs |
| **Quick peek** | "quick feed", "headlines" | Twitter timeline only, cards only, no ingest |
| **Query KB** | "what do I know about [topic]" | Read from secondmem, no fetch |
| **Ingest only** | "ingest this", "remember this" | No fetch, write provided content to secondmem |
| **Show unread** | "what's pending", "unread", "what haven't I read" | Show ○ items from reading-list.md as cards |
| **Show read** | "what did I read", "reading history" | Show ● items from reading-list.md |
| **Mark read** | "read #N", "done with #N", "mark #5 read" | Move item from ○ to ● in reading-list.md |
| **Mark batch** | "mark all P0 read", "done with agentic" | Move matching tier/topic items to ● |

## Agent Instructions

1. ALWAYS fetch with `--json` for Twitter (for classification)
2. Run cross-reference fetches in PARALLEL (independent curl calls)
3. Classify ALL items before displaying any cards
4. Display cards grouped by tier, sorted by engagement within tier
5. After displaying: ingest P0 + high-engagement P1 into secondmem
6. Every ingested item MUST have `> **Posted:** YYYY-MM-DD · **Ingested:** YYYY-MM-DD` in its blockquote
7. Update hierarchy.md files after ingest
8. Append new items to `~/.secondmem/timeline.md` under the current month section
9. Log all operations to `~/.secondmem/logs/ingest.log`
10. Save catch-up file to `~/Documents/twitter-catchup-YYYY-MM-DD.md` for >20 items
11. Default to `--lang en` for Twitter
12. If a fetch fails (timeout, no Chrome), skip that source and continue with others
13. Never block the whole pipeline on one source failure

## Prerequisites

```bash
# Required: headless-* Chrome CDP skills for social fetches
which headless-twitter   || npm install -g headless-twitter
which headless-instagram || npm install -g headless-instagram
which headless-linkedin  || npm install -g headless-linkedin
which headless-youtube   || npm install -g headless-youtube

# Required: jq for JSON processing
which jq || sudo apt install jq

# Chrome must be running with remote debugging on port 9222 with the user
# logged into Twitter/Instagram/LinkedIn/YouTube. Same Chrome instance is
# shared by all four headless-* skills (they intercept different domains).
# Start once: google-chrome --remote-debugging-port=9222 --user-data-dir=$HOME/.config/google-chrome

# Knowledge base must exist
ls ~/.secondmem/knowledge/hierarchy.md
```
