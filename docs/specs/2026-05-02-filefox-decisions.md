# Filefox — decisions covered by the design

**Purpose:** a one-page index of every topic discussed and decided during the design session, with pointers to where each decision lives in the full spec ([2026-05-02-filefox-design.md](./2026-05-02-filefox-design.md)). Use this as a fast lookup; use the design doc for the actual rules.

---

## 1. Foundational decisions (shape of the project)

| Topic | Decision | Spec section |
|---|---|---|
| Project shape | Harness for `AGENTS.md`-aware coding agents (no CLI tool, no app) | §3 |
| Agent target | Agnostic — uses the open `AGENTS.md` standard | §3 |
| Distribution | GitHub template repo + `init.sh` bootstrapper | §3, §7 |
| Domain coverage | One generic schema, no domain presets | §3, §11 |
| Worked example theme | "Studying the Transformer" — covers papers + articles + slides + an LLM explainer | §8 |
| Repository layout | Three top-level concepts: `raw/` (sources), `wiki/` (LLM output), `AGENTS.md` (rules) | §4 |

## 2. The schema (`AGENTS.md`) — what it tells the agent

| Topic | Spec section |
|---|---|
| Role definition ("wiki maintainer, not chatbot") | §5.1 |
| Three-layer architecture restated | §5.2 |
| Page conventions: frontmatter (`type`, `tags`, `sources`, `updated`, `summary`), wikilinks, lead-section structure | §5.3 |
| Provenance tags inside pages: `[claim]`, `[synthesis]`, `[inference]`, `[gap]` | §5.4 |
| Source taxonomy: `paper`, `article`, `lecture-slides`, `book`, `transcript`, `note`, `llm-explainer` | §5.5 |
| `llm-explainer` as a special source type with required model/date/caveat frontmatter | §5.6 |
| The four operations: `ingest`, `query`, `lint`, `explain` (each with full recipe) | §5.7 |
| Logging convention (greppable date prefix in `log.md`) | §5.9 |
| Self-evolution rule | §5.10 |
| Token-efficiency rules pointer | §5.11 |

## 3. Operations — the recipes

| Operation | What it does | Spec section |
|---|---|---|
| `ingest` | Reads a new source, plans cross-page updates, presents diff for approval, logs it | §5.7.1 |
| `query` | Searches the wiki via index → frontmatter → lead → body, answers with citations + confidence | §5.7.2 |
| `lint` | Health-checks the wiki (fast mode = frontmatter only, deep mode = reads bodies); regenerates `health.md` | §5.7.3 |
| `explain` | LLM teaches you something, then files the explanation as a labeled `llm-explainer` source | §5.7.4 |

## 4. Hallucination control — the 8 verification rules (§5.8)

| # | Rule |
|---|---|
| 1 | Every factual claim cites a source |
| 2 | Verbatim quote for load-bearing claims |
| 3 | Provenance tags where ambiguity exists |
| 4 | "No training data" — unsupported facts must be marked `[gap]` |
| 5 | Contradictions tracked in `contradictions.md`, never silently overwritten |
| 6 | Diff-and-approve on ingest (human in the loop) |
| 7 | Confidence label on every query answer; cap at Medium for explainer-only |
| 8 | Lint catches uncited claims, broken links, unmarked inferences, etc. |

## 5. Trust visibility — 8 places weakness shows up (§6)

How `llm-explainer`-backed content makes its weakness visible:

1. Citation path
2. Frontmatter
3. Caveat field
4. Provisional banner on pages
5. `index.md` marker
6. Confidence cap on queries
7. `wiki/health.md` dashboard
8. Active suggestion during ingest

## 6. Cost / token efficiency — 12 rules in two groups (§6a)

**Intrinsic cost caps (rules 1–7):**

- `AGENTS.md` ≤ ~2000 tokens
- Source summaries ≤ ~500 tokens
- Wiki pages ≤ ~1500 tokens
- Two-pass ingest
- `grep` before `index.md` for focused queries
- Incremental lint by default
- Don't edit `AGENTS.md` casually (preserves prompt cache)

**Read-time minimization (rules 8–12) — the skill-style progressive-disclosure idea applied to wiki content:**

- Required `summary:` frontmatter on every page
- Frontmatter-first scanning during query and ingest
- Soft pages-read budgets (query ≤ 5, ingest ≤ 10, lint ≤ 20)
- `index.md` entries carry summary, not just link
- Fast lint (frontmatter only) vs. deep lint (reads bodies)

## 7. Bootstrapper (§7)

`init.sh` — a single bash script (~50 lines, no language runtime) that creates folders, fetches `AGENTS.md` and starter wiki files from the repo, runs `git init`, prints next steps.

## 8. README structure — 10 sections (§9)

What it is → demo → quickstart → how to query (with phrase table) → how Filefox handles hallucination → use cases → what you need (agent options) → cost & model selection → credit (Karpathy) → license.

## 9. Success criteria — 7 measurable goals (§10)

1. User onboards in ≤ 5 minutes
2. Worked example is self-explanatory
3. Zero install dependencies
4. `AGENTS.md` ≤ ~2000 tokens
5. Every rule + visibility signal demonstrated in the example
6. README answers core questions in first 2 screen-heights
7. Median ingest ≤ 25k tokens, median query ≤ 8k tokens

## 10. Out of scope (§11) — deliberate non-goals

- Python / Node packages; CLI binaries beyond `init.sh`
- Vector / embedding search, `qmd` integration
- Source-file checksums / hash-anchoring
- Web UI, Electron app, Obsidian plugin
- Domain-specific presets
- Automated tests for example content
- Splitting `wiki/` into a `provisional/` folder (rejected — would break wikilinks)
- Splitting `AGENTS.md` into per-operation files (Cursor-skill-style schema disclosure — deferred; the wiki-content frontmatter-first approach addresses the actual cost concern)

## 11. Open questions / risks acknowledged (§12)

- Agent compliance with `AGENTS.md` instructions
- Token budget at very large wiki sizes
- No-agent users (need clear onboarding to free options)
- Copyright handling for example content

---

*This index reflects the state of the design as of the close of the brainstorming session. If a decision is later revisited, update both this file and the corresponding section of the design doc.*
