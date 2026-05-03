# Filefox — design spec

**Status:** Draft v1
**Date:** 2026-05-02
**Author:** Brainstormed in collaboration with Cursor agent
**Source pattern:** [`llm-wiki.md`](../../llm-wiki.md) (Karpathy)

---

## 1. One-line summary

Filefox is a GitHub template repository that turns any `AGENTS.md`-aware coding agent into a disciplined personal-wiki maintainer, implementing the LLM-wiki pattern with a verification regime that makes hallucination cheap to detect.

## 2. Problem & motivation

The LLM-wiki pattern (in `llm-wiki.md`) is intentionally abstract — it describes *the idea*, not an implementation. Anyone who wants to actually try the pattern has to design directory structure, page conventions, schema rules, and operations from scratch. That friction is what Filefox removes.

Two further pain points the spec is intentionally trying to solve:

1. **Hallucination contamination.** Without explicit rules, an LLM-maintained wiki silently mixes sourced facts with training-data assertions. The wiki *looks* authoritative even when it isn't.
2. **Useful-but-weak content.** Users will sometimes want to learn from the LLM directly ("explain X to me"). That's legitimate, but the resulting content shouldn't masquerade as a verified source.

Filefox addresses both: a strict-by-default schema, plus a recognized "LLM explainer" source type whose weakness is visible everywhere it appears.

## 3. Project shape (and what it isn't)

**Filefox is a harness, not a tool.** It is a GitHub template repo containing:

- An `AGENTS.md` schema that disciplines the user's coding agent.
- A starter folder structure (`raw/`, `wiki/`).
- A small bash bootstrapper (`init.sh`) for users who want to scaffold a wiki into an existing folder.
- One worked example (`examples/ai-study/`) showing what a healthy wiki looks like.
- A README that gets a new user productive in under five minutes.

**Filefox is not (v1):**

- A CLI tool (no `filefox query "..."`). The query interface is the user's coding agent.
- A Python or Node package. No language runtime, no API keys to manage.
- A web UI, Electron app, or Obsidian plugin.
- A vector-search / embedding-based RAG system. The `index.md` file is enough at moderate scale.
- A source-file checksum / hash-anchoring system.
- Domain-specific presets (one generic schema only).

The motivating principle is YAGNI: if we keep Filefox to pure markdown + a tiny shell script, it is approximately zero-maintenance and extremely portable.

## 4. Repository layout

```text
filefox/
├── README.md
├── AGENTS.md                  # the schema (the brain)
├── init.sh                    # bootstrapper for existing folders
├── LICENSE                    # MIT
├── .gitignore
├── raw/                       # immutable source documents (user fills)
│   ├── llm-explainers/        # reserved subfolder for LLM-generated sources
│   │   └── .gitkeep
│   └── .gitkeep
├── wiki/                      # LLM-maintained markdown
│   ├── index.md               # catalog of pages (LLM-maintained)
│   ├── log.md                 # chronological event log (LLM-maintained, append-only)
│   ├── health.md              # auto-generated dashboard (LLM-maintained on lint)
│   ├── contradictions.md      # unresolved conflicts (LLM-maintained)
│   └── .gitkeep
└── examples/
    └── ai-study/              # the worked example
        ├── README.md
        ├── raw/
        │   ├── papers/
        │   ├── articles/
        │   ├── lecture-slides/
        │   └── llm-explainers/
        └── wiki/
            ├── index.md
            ├── log.md
            ├── health.md
            ├── contradictions.md
            └── (entity & concept pages)
```

Three top-level user-facing concepts: `raw/` (sources, immutable), `wiki/` (LLM output, fully owned by the agent), `AGENTS.md` (the contract). This mirrors the three layers in `llm-wiki.md` exactly.

## 5. The schema — `AGENTS.md`

`AGENTS.md` is the most important file in the project. Its sections, in order:

### 5.1 Role
"You are a wiki maintainer, not a chatbot." Sets stance: edit files, follow conventions, surface uncertainty.

### 5.2 Layers
Restates the three-layer architecture: `raw/` is immutable, `wiki/` is yours to write, this file is the shared contract.

### 5.3 Page conventions

- All wiki pages are markdown with YAML frontmatter.
- Required frontmatter on every wiki page: `type` (entity | concept | summary | synthesis), `tags`, `sources` (list of file paths cited), `updated` (ISO date), `summary` (one-line, ~120 chars; the agent's "do I need to read this body?" signal — same role as a skill's `description:` field).
- Cross-references use `[[wikilinks]]` (Obsidian-compatible).
- One entity or concept per file. Filenames are kebab-case slugs of the topic.
- **Lead-section convention.** Every wiki page opens with a `## Lead` section: a bullet list of the page's key claims (each with citation). Detail and examples come after. This lets the agent answer most queries from the lead alone, reading the full body only when the question needs depth. Mirrors how Wikipedia leads work.

### 5.4 Provenance tags (in page body)

A small vocabulary, used inline to mark sentence/claim provenance:

- `[claim]` — directly stated in a cited source
- `[synthesis]` — combines multiple cited claims
- `[inference]` — LLM reasoning beyond sources (must be flagged for human review)
- `[gap]` — unknown / not in any source (must NOT be asserted as fact)

Use sparingly — only where provenance might be ambiguous to a reader.

### 5.5 Source taxonomy

`raw/` files have frontmatter with `source_type`. Recognized values:

- `paper` — peer-reviewed or formal publication
- `article` — blog post, news, online article
- `lecture-slides` — class slides, presentations
- `book` or `book-chapter` — published books
- `transcript` — podcast or video transcript
- `note` — user-written note
- `llm-explainer` — LLM-generated explainer (see §5.6)

### 5.6 The `llm-explainer` source type (special)

Frontmatter required:

```yaml
---
source_type: llm-explainer
model: <model-id>          # e.g. claude-sonnet-4.7
date: <ISO date>            # generation date
topic: <short topic>
caveat: LLM-generated from training data; no primary source. Verify before serious use.
---
```

Stored in `raw/llm-explainers/`. Treated as a first-class source for citation purposes, but the weakest type. See §6 for visibility rules.

### 5.7 Operations

Four operations, each spelled out as an explicit recipe.

#### 5.7.1 `ingest`
Triggered when the user adds a file to `raw/` and asks to ingest, or pastes a URL.

Recipe — *plan all changes first, write only on approval; use frontmatter-first scanning to limit page reads:*

1. Read the source end-to-end.
2. Discuss key takeaways with the user (in chat). Confirm understanding.
3. *Pass 1.* Extract candidate entities and concepts from the source alone (no wiki context yet).
4. *Pass 2.* For each candidate, check `index.md` to see if a page exists. Read **only frontmatter** of existing pages first, then `## Lead` of those that look genuinely affected. Only read full bodies when the lead doesn't tell you whether/how to update.
5. Plan a summary page at `wiki/sources/<slug>.md` (≤ ~500 tokens, bullet-form key claims).
6. Plan updates to affected entity/concept pages (and new pages where needed).
7. Check for contradictions with existing wiki claims. Plan entries to `wiki/contradictions.md` and `wiki/log.md` if found.
8. Plan updates to `wiki/index.md` (new entries, source counts, refreshed summaries).
9. Plan an append to `wiki/log.md` with prefix `## [YYYY-MM-DD] ingest | <source title>`.
10. **Present the full multi-file diff to the user. Write nothing to disk until explicit approval. On approval, apply all changes atomically.**

#### 5.7.2 `query`
Triggered when the user asks a question.

Recipe — *frontmatter-first, body-only-when-needed:*

1. Read `wiki/index.md` first to scan candidate pages by `type` / source-count / summary.
2. For each candidate, read **only frontmatter** (~100 tokens). Filter by `tags`, `summary` overlap with the question, and `updated` recency.
3. Read the `## Lead` section of survivors. For most queries, the lead is enough.
4. Read full page bodies only if the lead is insufficient for the question.
5. Synthesize an answer using only cited sources and previously verified wiki pages.
6. Cite every factual claim with a `[[wikilink]]` to its source page.
7. End the answer with: `Confidence: High | Medium | Low | Unknown — <one-line justification>`.
8. Offer: *"File this synthesis back into the wiki as a new page?"* (User opts in.)

Confidence cap: **if the answer is built only on `llm-explainer` sources, confidence cannot exceed Medium.**

#### 5.7.3 `lint`
Triggered periodically by the user.

Two modes:

- **Fast lint (default).** Frontmatter-only checks; cheap. Runs on every `lint` request unless the user says "deep" or "full".
- **Deep lint (opt-in).** Reads page bodies; expensive. Triggered by "full lint" or "deep lint".

Recipe — produces a report and regenerates `wiki/health.md`:

*Fast-lint checks (frontmatter only):*

1. Required frontmatter fields present (`type`, `tags`, `sources`, `updated`, `summary`).
2. Every path in `sources:` resolves to a real file in `raw/`.
3. Every `[[wikilink]]` in `index.md` resolves.
4. Find provisional pages (`sources:` contains only `llm-explainer` paths).
5. Find stale explainers (>180 days from `date:` in their frontmatter).
6. Find pages with `updated:` older than the latest source they cite (possibly stale).

*Deep-lint checks (read bodies):*

7. Find uncited claims (sentences without a `[[wikilink]]` to a source).
8. Find unmarked inferences (claims not traceable to any cited source).
9. Verify lead-section bullets match the page's claims (no drift).
10. Find concepts mentioned in multiple pages but lacking their own page.

*Both modes:*

11. Find unresolved contradictions in `contradictions.md`.
12. Find orphan pages (no inbound links from any other wiki page).
13. Suggest concrete next actions (e.g., "search web for primary source for [[wiki/X.md]]").
14. Write the result to `wiki/health.md`, replacing previous content. Update `last_full_lint:` only on deep-lint runs.
15. Append an entry to `log.md` with prefix `## [YYYY-MM-DD] lint | fast` or `## [YYYY-MM-DD] lint | deep`.

#### 5.7.4 `explain`
Triggered when the user wants to learn something they don't yet have a source for.

Recipe:
1. Produce the explanation in chat — do **not** write it to disk yet.
2. Iterate with the user: follow-ups, clarifications, refinements.
3. When user says "file this" (or equivalent), write to `raw/llm-explainers/<topic>.md` with full frontmatter (§5.6).
4. Then run the standard `ingest` workflow on the new file (§5.7.1) — including diff-and-approve.
5. State out loud in chat that the resulting wiki page is provisional and offer to fetch a primary source.

### 5.8 Verification regime — the 8 rules

These are the schema's hard rules. The agent MUST follow them.

1. **Citation rule.** Every factual claim in `wiki/` cites at least one file in `raw/` or another wiki page that ultimately cites `raw/`.
2. **Verbatim-quote rule.** For load-bearing or non-obvious claims, embed a verbatim quote from the source in a collapsible block adjacent to the claim.
3. **Provenance tags.** Use `[claim]`, `[synthesis]`, `[inference]`, `[gap]` (§5.4) where ambiguity exists.
4. **No training-data rule.** *"If a fact is not in `raw/` or in cited wiki pages, you do not know it. Mark `[gap]` or surface as a question. Do not assert it."* This is the single most important rule.
5. **Contradiction tracking.** On ingest, check new sources against existing claims. Log conflicts to `wiki/contradictions.md` and `log.md`. Never silently overwrite.
6. **Diff-and-approve on ingest.** Present a diff and wait for explicit approval before writing files.
7. **Confidence label.** Every query answer ends with `Confidence: …`. Confidence ≤ Medium when only `llm-explainer` sources are cited. Low/Unknown answers may not be filed back into the wiki without a second explicit approval.
8. **Lint extension.** The `lint` operation explicitly checks for: uncited claims, broken/missing citations, unmarked inferences, provisional pages, stale explainers, unresolved contradictions, orphans, missing pages.

### 5.9 Logging convention

`wiki/log.md` is append-only. Every entry starts with:

```markdown
## [YYYY-MM-DD] <op> | <subject>
```

So `grep "^## \[" wiki/log.md | tail -10` works.

### 5.10 Self-evolution
"If a convention in this file is unclear or fights the work, propose an edit to this file in chat. Do not silently work around it."

### 5.11 Token efficiency
The twelve schema-level rules listed in §6a (split into "intrinsic cost caps" — rules 1–7 — and "read-time minimization" — rules 8–12). The agent must treat them as soft budgets and surface its intent when it expects to exceed them (e.g. *"this ingest will need to read ~25 wiki page bodies — OK to proceed?"*).

## 6. Visibility signals for weak sources

`llm-explainer`-backed content shows its weakness in **eight** places:

1. **Citation path.** `[[raw/llm-explainers/...]]` is self-documenting in any wiki page.
2. **Frontmatter** (`source_type: llm-explainer`).
3. **Caveat field** in explainer frontmatter — a literal warning string.
4. **Provisional banner** at the top of any wiki page whose `sources:` frontmatter contains only `llm-explainer`-typed paths (auto-added on ingest, auto-removed once the page's `sources:` includes at least one non-explainer source).
5. **`index.md` marker** — explainer-only entries are tagged `*(provisional: explainer-backed)*`.
6. **Confidence cap** in query answers (rule 7).
7. **`wiki/health.md` dashboard** — lists every provisional page, every stale explainer, with suggested next actions.
8. **Active suggestion during ingest** — when an `llm-explainer` is the only source, the agent says so in chat and offers to fetch a primary source.

## 6a. Token efficiency — schema rules

LLM-driven wikis can become expensive at scale. These rules keep cost predictable without adding code, embeddings, or vector search. They live in `AGENTS.md` (as section 5.11) and are enforced by the agent on itself.

Two groupings:

- **Rules 1–7 — intrinsic cost caps.** Limit how big each artifact gets in the first place.
- **Rules 8–12 — read-time minimization.** When operations run, read only what's needed.

#### Intrinsic cost caps

1. **Cap `AGENTS.md` at ~2000 tokens.** Every chat session pays this cost. Bullet-dense, not prose.
2. **Cap source summary pages at ~500 tokens.** Bullet-point key claims, not narrative retellings. The full source remains in `raw/` for drill-down when needed.
3. **Cap wiki entity / concept pages at ~1500 tokens.** Split into sub-pages if larger. Keeps queries cheap.
4. **Two-pass ingest.** Pass 1: read the source alone, extract candidate entities/concepts. Pass 2: consult `index.md`, then read only existing pages that match candidates. Avoids the "read 15 pages just in case" pattern.
5. **`grep` before `index.md` for focused queries.** When the user asks about a specific named entity, search the wiki for the term first and read only matching files. Use `index.md` for exploratory ("what do I know about area X?") queries.
6. **Lint is incremental by default.** `wiki/health.md` records `last_full_lint:` in its frontmatter. Routine lint reads only pages updated since that date plus pages flagged on the previous report. A full lint must be requested explicitly ("full lint").
7. **Don't edit `AGENTS.md` casually.** Stable text preserves prompt cache across sessions. Edit only when a convention is genuinely broken.

#### Read-time minimization

8. **`summary:` frontmatter on every wiki page.** A single short line — the page's elevator pitch — that lets the agent decide whether to read the body. Same pattern as Cursor skill `description:` fields. Required, not optional.
9. **Frontmatter-first scanning.** When the agent is finding pages relevant to a query or ingest, it reads only frontmatter (~100 tokens/page) for candidates first, then reads `## Lead` sections of the survivors, and only reads full bodies if the lead is insufficient. Roughly a 3–4× input-token reduction per query.
10. **Soft pages-read budget per operation.** Defaults: query ≤ 5 page bodies, ingest ≤ 10, incremental lint ≤ 20. If the agent estimates it needs more, it pauses and asks the user before proceeding.
11. **Index entries carry summary, not just link.** `index.md` line format: `- [[wiki/<page>.md]] — type:<type> — <N> sources — "<summary>"`. The index alone often answers "do I have notes on X?" without any page reads.
12. **Fast lint vs. deep lint.** Fast lint reads only frontmatter (broken `sources:` paths, missing required fields, stale dates). Deep lint reads bodies (uncited claims, unmarked inferences, contradictions). Fast lint is cheap and runs by default; deep lint is opt-in and slower.

These are guidelines, not hard contracts — the agent should treat them as soft budgets and surface them to the user when it intends to exceed them ("this ingest will need to read ~25 page bodies; OK to proceed?").

## 7. `init.sh` — bootstrapper

A single bash script (~50 lines, no dependencies). Usage:

```bash
# Bootstrap into a new folder:
bash <(curl -sSL https://raw.githubusercontent.com/<user>/filefox/main/init.sh) my-wiki

# Bootstrap into the current folder:
./init.sh .
```

Behavior:

1. Validate target folder (exists or can be created; warn if not empty).
2. Create `raw/`, `raw/llm-explainers/`, `wiki/` with `.gitkeep`s.
3. Fetch and write `AGENTS.md` from the repo's `main` branch via `curl`. (Trade-off: requires network at init time. Avoids drift between source-of-truth `AGENTS.md` and an embedded copy in `init.sh`.)
4. Fetch and write starter `wiki/index.md`, `wiki/log.md`, `wiki/health.md`, `wiki/contradictions.md` from `main` via `curl`.
5. `git init` if not already a git repo.
6. Print a 4-line "next steps" message:
   ```text
   Done. Next:
     1. Drop a source into raw/ or paste a URL.
     2. Open this folder in your AGENTS.md-aware coding agent (Cursor, Codex, Aider, Claude Code, ...).
     3. Ask: "ingest this".
   ```

No flags, no presets, no language runtime. Auditable in one screen.

## 8. Worked example: `examples/ai-study/`

Theme: **Studying the Transformer architecture.** Picks up multiple source types naturally (papers, articles, lecture slides) and matches the user's stated need to mix common-knowledge LLM explainers with primary sources.

Required contents:

- `raw/papers/attention-is-all-you-need.md` — summary of the original paper (with frontmatter `source_type: paper`).
- `raw/articles/illustrated-transformer.md` — markdown of a blog explainer.
- `raw/lecture-slides/cs224n-attention.md` — class lecture slides.
- `raw/llm-explainers/positional-encoding.md` — one LLM explainer (to demonstrate the type).
- `wiki/index.md` — catalog with provisional markers visible.
- `wiki/log.md` — 4 ingest entries + 1 query entry + 1 lint entry.
- `wiki/health.md` — example dashboard output, including a provisional-page entry.
- `wiki/contradictions.md` — at least one demonstrative entry.
- `wiki/transformer.md` — well-cited entity page.
- `wiki/attention.md` — entity page citing the paper, the article, and slides.
- `wiki/positional-encoding.md` — provisional page, explainer-only, with banner.
- `wiki/sources/<slug>.md` — one source summary per source.
- `examples/ai-study/README.md` — points readers at the most illustrative pages and flags the provisional banner.

The example must demonstrate **every** verification rule and **every** visibility signal at least once. Every wiki page in the example must use the lead-section convention (frontmatter `summary:` + `## Lead` bullets) so users see the pattern that makes frontmatter-first scanning work.

## 9. README

Sections in order (60-second comprehension target):

1. **What is Filefox** — one paragraph adapted from the source pattern.
2. **30-second demo** — animated GIF or clear screenshot sequence: drop a file in `raw/`, ask the agent to ingest, see the wiki update.
3. **Quickstart** — two paths: "Use this template" button (primary) **or** `bash <(curl -sSL ...) my-wiki` (alternate).
4. **How to query** — concrete example session showing the agent reading `index.md`, drilling into pages, citing with `[[wikilinks]]`, ending with `Confidence:`. Plus a small phrase table:

   | Phrase | What the agent does |
   |---|---|
   | "ingest this" / "ingest \<path\>" | runs §5.7.1 |
   | "query: …" / a plain question | runs §5.7.2 |
   | "explain X" | runs §5.7.4 |
   | "lint the wiki" | runs §5.7.3 |

5. **How Filefox handles hallucination** — one short page summarizing rules 1, 4, 6, 7 in plain language. Links to `AGENTS.md` for full detail.
6. **Use cases** — bullets: research, class notes, book companion, personal journal, hobby deep-dives. Each with a one-line "what would `raw/` contain."
7. **What you need** — links to free / paid agent options (Cursor, Codex, Aider, Claude Code).
8. **Cost and model selection** — short paragraph: "Ingest, lint, and explain work well on cheap or smaller models — they're mostly mechanical text manipulation. Reserve frontier models for hard queries and synthesis. Filefox's schema is model-agnostic; pick whatever balances cost and quality for you. See §5.11 of `AGENTS.md` for the schema's built-in token-efficiency rules."
9. **Credit** — links to `llm-wiki.md` and Karpathy.
10. **License** — MIT.

## 10. Success criteria (v1 release)

1. A new user can go from "see the GitHub repo" → "first ingest into their own wiki" in under five minutes.
2. The worked example is convincing enough that someone reading it understands the pattern without re-reading `llm-wiki.md`.
3. The repo has zero install dependencies (bash is enough). No "broken on someone's machine" risk.
4. `AGENTS.md` is short enough that a coding agent reliably picks it up each session (target: under ~2000 tokens).
5. Every verification rule and visibility signal is demonstrated at least once in the worked example.
6. The README answers "how do I query?" and "how does Filefox handle hallucination?" within the first two screen-heights.
7. Median ingest reads ≤ ~25k input tokens, and median query reads ≤ ~8k input tokens, on a wiki of up to 100 pages. (Guidepost, not contract — if we blow past it, the recipes or frontmatter-first scanning need tightening.)

## 11. Out of scope (explicit non-goals for v1)

- Python or Node packages.
- A `filefox` CLI binary (beyond `init.sh`).
- Vector / embedding search; `qmd` integration.
- Source-file checksum / hash-anchoring.
- Web UI, Electron app, Obsidian plugin.
- Domain-specific presets.
- Automated tests for example content.
- A separate `wiki/provisional/` folder (rejected — would break wikilinks on promotion).
- Splitting `AGENTS.md` into per-operation files (Cursor-skill-style progressive disclosure of the schema). Considered and deferred — at ~2000 tokens the schema is small enough that the file-count overhead outweighs token savings. The actual read-time cost concern is addressed by frontmatter-first scanning of wiki **content** (§6a rules 8–12), which is where the leverage lives. Revisit if the schema grows past ~5000 tokens or 8+ operations.
- Slash-command shortcuts shipped per agent (users can add their own).

## 12. Open questions / risks

- **Agent compliance with `AGENTS.md` rules.** Different agents read `AGENTS.md` with different rigor. Mitigation: keep it short, use clear imperative voice, demonstrate via the worked example.
- **Token budget.** As `wiki/` grows, an agent loading `index.md` may exceed context. Mitigation flagged for v2: hierarchical indexes, or `qmd` integration. Not v1.
- **No-agent users.** If a user lands on the repo without an agent set up, the README's "what you need" section is the only on-ramp. We may want a "no agent? paste this prompt" fallback later.
- **License of example content.** All worked-example sources are paraphrased / fabricated to avoid copyright concerns. Must be clearly labeled as such in `examples/ai-study/README.md`.
