---
name: llm-wiki
description: "Use when building or maintaining a personal LLM-powered knowledge base. Triggers: ingesting sources, querying wiki knowledge, linting wiki quality, mining insights, active reference collection, 'add to wiki', 'what do I know about', 'ingest', 'lint wiki', 'insight mining', 'research', 'find papers', 'collect references', 'survey literature', or any mention of 'LLM wiki'."
---

# LLM Wiki

Build and maintain a personal knowledge base using LLMs. All content lives under `.wiki/` — simultaneously a structured knowledge repository and an Obsidian vault. You manage two areas within it: `raw/` (immutable source material) and topic directories (compiled knowledge articles). Sources go into raw/, you compile them into articles, and the wiki compounds over time.

Core principles:
- "The LLM writes and maintains the wiki; the human reads and asks questions."
- "The wiki is a persistent, compounding artifact."

## Architecture

Everything lives under `.wiki/` at the project root:

**.wiki/raw/** — Immutable source material. You read, never modify. Organized by topic subdirectories (e.g., `.wiki/raw/transformers/`).

**.wiki/\<topic\>/** — Compiled knowledge articles. You have full ownership. One level of topic subdirectories only: `.wiki/<topic>/<article>.md`. Directory names: lowercase, hyphenated.

**Special files at `.wiki/` root:**
- `.wiki/index.md` — Global index. One row per article, grouped by topic, with link + summary + Updated date. The LLM reads this first when answering queries.
- `.wiki/log.md` — Append-only operation log. Each entry starts with `## [YYYY-MM-DD] <operation> | <title>` for parseability.

**SKILL.md** (this file) — Schema layer. Defines structure, conventions, and workflow rules.

Templates live in `references/` relative to this file. Read them for exact format specifications.

### Obsidian Vault

`.wiki/` is designed to be opened directly as an Obsidian vault. On first open, Obsidian creates `.wiki/.obsidian/` with its configuration.

**Frontmatter** — Every article includes YAML frontmatter with at minimum a `tags` field. This enables:
- **Dataview** queries across the knowledge base
- **Tag pane** navigation and filtering
- **Properties** view in Obsidian's sidebar

**Links** — Use standard markdown links with relative paths (`[Title](../topic/article.md)`). Portable and fully supported by Obsidian's graph view, backlinks, and link resolution.

**Graph view** — Shows article interconnections visually. Write cross-references generously to build a rich graph. Orphan articles (no inbound links) degrade the graph — Lint catches these.

**Recommended plugins:** Dataview, Graph Analysis, Calendar, Tag Wrangler, Obsidian Web Clipper (for fetching sources).

### Initialization

Triggers only on the first Ingest. Check whether `.wiki/` structure exists. Create only what is missing; never overwrite existing files:

- `.wiki/` directory
- `.wiki/raw/` directory
- `.wiki/index.md` — heading `# Wiki Index`, empty body
- `.wiki/log.md` — heading `# Wiki Log`, empty body

Obsidian creates `.wiki/.obsidian/` automatically on first vault open. Do not pre-create it.

If Query or Lint cannot find the wiki structure, tell the user: "Run an ingest first to initialize the wiki." Do not auto-create.

---

## Ingest

Fetch a source into `.wiki/raw/`, then compile it into `.wiki/<topic>/`. Always both steps, no exceptions.

### Fetch (raw/)

1. Get the source content using whatever web or file tools your environment provides. If nothing can reach the source, ask the user to paste it directly.

2. Pick a topic directory. Check existing `.wiki/raw/` subdirectories first; reuse one if the topic is close enough. Create a new subdirectory only for genuinely distinct topics.

3. Save as `.wiki/raw/<topic>/YYYY-MM-DD-descriptive-slug.md`.
   - Slug from source title, kebab-case, max 60 characters.
   - Published date unknown → omit the date prefix from the file name (e.g., `descriptive-slug.md`). The metadata Published field still appears; set it to `Unknown`.
   - If a file with the same name already exists, append a numeric suffix (e.g., `descriptive-slug-2.md`).
   - Include metadata header: source URL, source-type, collected date, published date.
   - **`source-type` field** classifies the provenance of each raw file:
     - `academic` — Peer-reviewed paper, journal article, or conference publication. **Default when omitted.**
     - `internal` — Team experiments, internal reports, unpublished findings.
     - `meeting-note` — Notes from research meetings, advisor discussions, lab sessions.
     - `blog` — Blog posts, tutorials, non-peer-reviewed technical writing.
   - Preserve original text. Clean formatting noise. Do not rewrite opinions.

   See `references/raw-template.md` for the exact format.

4. **OpenReview review collection (for paper sources)**

   When the source is an academic paper with publicly available reviews on OpenReview:
   - Search for the paper's review page on OpenReview (`https://openreview.net/forum?id=...`)
   - Collect each reviewer's assessment and record it in the `## OpenReview Reviews` section at the bottom of the raw file
   - Fields to collect: Rating, Confidence, key Strengths, key Weaknesses, final Decision
   - Separate reviewers with `### Reviewer [N]` headings
   - Preserve review text faithfully; for excessively long reviews, extract key points from strengths/weaknesses
   - If the paper cannot be found on OpenReview or reviews are not public → skip this section entirely and proceed

### Compile (wiki/)

#### Page Types

Every wiki page has a type, indicated by the `type` tag in frontmatter. Choose the right type when creating or merging content:

| Type | Tag | Purpose | When to create |
|------|-----|---------|----------------|
| **Summary** | `summary` | Summary of a single paper/method/tool | When a source makes an independent contribution during Ingest |
| **Concept** | `concept` | Comprehensive coverage of a single concept or technique | When multiple sources address the same concept (merge target) |
| **Entity** | `entity` | Dedicated page for a specific model, dataset, tool, or benchmark | When referenced in 3+ other articles |
| **Comparison** | `comparison` | Systematic comparison across methods/approaches | When 3+ methods solve the same problem differently |
| **Overview** | `overview` | Bird's-eye view of an entire topic directory | Automatic — see Overview rules below |
| **Synthesis** | `synthesis` | Cross-topic pattern analysis | Automatic — see Synthesis rules below |
| **Archive** | `archive` | Snapshot of a Query result saved to wiki | Only when explicitly requested by the user |

Include the type tag in the existing `tags` field. Example: `tags: [concept, diffusion, scene-aware]`.

During Ingest, do not default to summary — choose the type that fits the source's nature. When sources accumulate around the same concept, consider consolidating multiple summaries into a single concept page.

#### Placement Rules

Determine where the new content belongs:

- **Same core thesis as existing article** → Merge into that article. Add the new source to Sources/Raw. Update affected sections.
- **New concept** → Create a new article in the most relevant topic directory. Name the file after the concept, not the raw file.
- **Spans multiple topics** → Place in the most relevant directory. Add See Also cross-references to related articles elsewhere.

These are not mutually exclusive. A single source may warrant merging into one article while also creating a separate article for a distinct concept it introduces. In all cases, check for factual conflicts: if the new source contradicts existing content, annotate the disagreement with source attribution. When merging, note the conflict within the merged article. When the conflicting content lives in separate articles, note it in both and cross-link them.

See `references/article-template.md` for article format. Key points:
- YAML frontmatter with `tags`, `created`, `updated` fields.
- Sources field in frontmatter: author, organization, or publication name + date, semicolon-separated. **Prefix non-academic sources with their type marker:** `[Internal]`, `[Meeting]`, or `[Blog]`. Academic sources have no prefix (default). Example: `"[Meeting] Hwang (2026-02-04); Peng et al. (SIGGRAPH 2021)"`.
- Raw field in frontmatter: markdown links to `.wiki/raw/` files, semicolon-separated.
- Relative paths from `.wiki/<topic>/` to raw use `../raw/<topic>/<file>.md`.

### Source Provenance in Article Body

When compiling content from non-academic sources (raw files with `source-type` other than `academic`), mark the claims in the article body with an inline provenance tag:

- `**[Internal]**` for internal experiments/reports
- `**[Meeting]**` for meeting notes
- `**[Blog]**` for blog/non-peer-reviewed sources

Place the tag at the start of the paragraph or section that relies on the non-academic source. Academic-sourced content carries no tag (it is the default). This ensures readers can distinguish "Paper X reports Y" from "Our team observed Y" at a glance.

### Cascade Updates

After the primary article, check for ripple effects:

1. Scan articles in the same topic directory for content affected by the new source.
2. Scan `.wiki/index.md` entries in other topics for articles covering related concepts.
3. Update every article whose content is materially affected. Each updated file gets its `updated` frontmatter date refreshed.

Archive pages are never cascade-updated (they are point-in-time snapshots).

### Overview Pages

A bird's-eye view page for an entire topic directory. Filename: `_overview.md` (underscore prefix to sort first).

**Auto-generation trigger:** When a topic directory reaches **3+ articles**, create an overview page during Post-Ingest. If one already exists, update it.

**Content structure:**
1. Topic definition and scope — the problem space and boundaries this topic covers
2. Taxonomy of approaches — classify methods based on articles in the directory
3. Inter-article relationship map — which articles are successors/alternatives/extensions of others
4. Open problems — limitations commonly mentioned across articles in the directory
5. Links to all articles in the directory

**Rules:**
- Overview references only wiki articles within the directory. Do not cite raw files directly.
- `tags: [overview, <topic-name>]`
- Leave Sources/Raw frontmatter empty (synthesized from other wiki articles).
- Subject to cascade updates — when any article in the same directory changes, update the overview too.

### Synthesis Pages

Cross-topic pattern analysis pages. Placed in `.wiki/synthesis/`.

**Auto-generation trigger:** Create or update a synthesis page during Post-Ingest when any of these conditions are met:
- The same technique/concept appears across 3+ distinct topic directories (e.g., diffusion used in scene-aware, physics-based, and text-to-motion)
- The same limitation or trade-off is observed across 2+ topics
- A new source explicitly connects previously separate topics

**Content structure:**
1. Cross-topic pattern identification — shared techniques, trends, and limitations across topics
2. Inter-topic relationship analysis — how advances in one topic affect others
3. Integrated implications — higher-level insights invisible from individual topic articles
4. Related article links — all relevant articles across participating topics

**Rules:**
- `tags: [synthesis, <pattern-name>]`
- Leave Sources/Raw frontmatter empty.
- If an existing synthesis page covers the same pattern, merge into it. Create a new page only for genuinely new patterns.
- Filenames reflect the pattern: `diffusion-across-domains.md`, `physics-plausibility-tradeoffs.md`, etc.
- Must reference at least 2 topics and 3+ wiki articles. Single-topic patterns belong in Overview pages.

### Post-Ingest

Update `.wiki/index.md`: add or update entries for every touched article. When adding a new topic section, include a one-line description. The Updated date reflects when the article's knowledge content last changed, not the file system timestamp. See `references/index-template.md` for format.

Append to `.wiki/log.md`:

```
## [YYYY-MM-DD] ingest | <primary article title>
- Raw: raw/<topic>/<filename>.md
- Updated: <cascade-updated article title>
```

Omit `- Updated:` lines when no cascade updates occur.

---

## Research

Active discovery and ingestion of CS research papers. Instead of the user providing individual URLs, the LLM searches for relevant papers from top CS venues, evaluates them, and ingests the best candidates into the wiki.

**Triggers:**
- "research X"
- "find papers about Y"
- "collect references on Z"
- "survey the literature on W"

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| topic | (required) | Research focus — a question, concept, or subfield |
| scope | `standard` | `quick` (3–5 papers), `standard` (5–10), `deep` (10–20) |
| venues | CS top venues | Restrict to specific venues. Default: CVPR, ICCV, ECCV, NeurIPS, ICLR, ICML, SIGGRAPH, SIGGRAPH Asia, TOG, TPAMI |
| date_range | Last 3 years | Search period |
| exclusions | none | Papers or subtopics to skip |

### Pipeline

#### Step 1: SCOPE

1. Read `.wiki/index.md` to understand what the wiki already covers.
2. Decompose the topic into search angles:
   - Core methods and techniques
   - Key datasets and benchmarks
   - Competing or alternative approaches
   - Recent extensions and follow-up work
3. Identify which existing wiki topics overlap (to avoid redundant ingestion).
4. Present the scope to the user as a brief confirmation before proceeding. If the user redirects, adjust. If no response within the conversational turn, proceed.

#### Step 2: PLAN

Generate search queries targeting multiple channels:

- **Venue-specific**: `site:openreview.net`, `site:openaccess.thecvf.com`, `site:arxiv.org`
- **Semantic**: Natural-language queries capturing the research question from different angles
- **Citation-chain**: If the wiki already has related papers, search for works that cite or are cited by them
- **Recency-filtered**: Date-constrained queries for the specified range

Produce 5–15 queries depending on scope.

#### Step 3: RETRIEVE

Execute searches in parallel:

1. Run all planned queries using WebSearch (or available search tools) concurrently.
2. For each result, extract: title, authors, venue, year, URL, abstract (if available).
3. Deduplicate by title similarity.
4. Score each candidate for source quality.

**Source quality scoring** uses venue tiers for CS papers:

| Tier | Score | Venues |
|------|-------|--------|
| Tier-1 | 95 | CVPR, ICCV, ECCV, NeurIPS, ICLR, ICML, SIGGRAPH, TOG |
| Tier-2 | 85 | AAAI, IJCAI, RSS, CoRL, ICRA, TPAMI, IJCV, 3DV, WACV |
| Tier-3 | 75 | Workshop papers at Tier-1 venues, well-cited arXiv preprints |
| Unranked arXiv | 55 | arXiv without venue publication |

Non-academic sources (blog posts, technical reports without peer review) score 55 or below. Prefer peer-reviewed venue papers whenever available.

**Quality thresholds** (minimum candidates before proceeding):
- `quick`: 8+ candidates
- `standard`: 15+ candidates
- `deep`: 25+ candidates

If thresholds are not met, broaden queries and retry once. If still insufficient, proceed with what is available and note the gap.

#### Step 4: GATE

Evaluate each candidate on five criteria and decide accept/reject:

1. **Venue quality** — Tier-1/2 venues get priority. Unranked arXiv accepted only if highly relevant and no better alternative exists.
2. **Relevance** — Does the abstract/title match the research scope? (LLM judgment)
3. **Redundancy** — Does the wiki already cover this paper? Check `.wiki/raw/` filenames and existing article content.
4. **Recency** — Prefer recent work, but accept seminal older papers that are foundational to the topic.
5. **Diversity** — Ensure coverage across subtopics identified in SCOPE. Do not over-collect from a single subtopic.

Rank candidates and accept the top N based on scope (`quick`: 3–5, `standard`: 5–10, `deep`: 10–20).

#### Step 5: INGEST (batch)

For each accepted paper, run the standard **Ingest** pipeline (Fetch → Compile → Cascade Updates → Post-Ingest).

Processing order: cluster papers by topic directory to minimize redundant cascade scans.

After all papers are ingested:
- Check Overview page triggers (3+ articles in a directory → create or update `_overview.md`).
- Check Synthesis page triggers (same concept across 3+ topics → create or update synthesis page).

### Post-Research

**Conversation output:**

Present to the user:
- Candidate table: title, venue, year, quality score, accepted/rejected, reason.
- List of wiki articles created or updated.
- Any new Overview or Synthesis pages generated.
- Suggested follow-up: narrower research topics or wiki queries to explore further.

**Wiki log:**

```
## [YYYY-MM-DD] research | <topic>
- Scope: <quick/standard/deep>
- Candidates: <N> found, <M> accepted
- Created: <new article titles>
- Updated: <cascade-updated article titles>
- New overview: <if any>
- New synthesis: <if any>
```

---

## Query

Search the wiki and answer questions. Triggers:
- "What do I know about X?"
- "Summarize everything related to Y"
- "Compare A and B based on my wiki"

### Source Scope Filtering

When the user specifies a source scope — e.g., "학술 연구 기준으로", "논문을 바탕으로", "based on papers" — restrict the answer to claims originating from `academic` sources only. Exclude or clearly separate content marked with `**[Internal]**`, `**[Meeting]**`, or `**[Blog]**` provenance tags. If no scope is specified, use all sources equally but still preserve provenance markers in the answer when citing non-academic claims.

### Steps

1. Read `.wiki/index.md` to locate relevant articles.
2. Read those articles and synthesize an answer. **Respect source scope filtering** if the user specified one — skip paragraphs/sections tagged with non-matching provenance markers.
3. Prefer wiki content over your own training knowledge. Cite sources with markdown links: `[Article Title](.wiki/topic/article.md)` (project-root-relative paths for in-conversation citations; within `.wiki/` files, use paths relative to the current file).
4. Output the answer in the conversation.

### Insight-Driven Updates

During query answering, if synthesizing across articles reveals new insights not already captured in the wiki, update the relevant articles. This includes:

- **Cross-article connections** previously unlinked (e.g., two methods share a technique neither article mentions)
- **Corrections** to outdated or inaccurate statements discovered through cross-referencing
- **Synthesis insights** that emerge only when combining information from multiple articles (e.g., a common limitation pattern, an implicit taxonomy)

**Rules:**
- Only update existing articles. Do not create new articles (use Ingest or Archiving for that).
- Each update must be traceable to wiki content read during the query — do not inject training knowledge.
- Refresh the `updated` frontmatter date on every modified article.
- Update `.wiki/index.md` if article summaries need revision.
- Append to `.wiki/log.md`:
  ```
  ## [YYYY-MM-DD] query-update | <article title>
  - Insight: <one-line description of what was added/changed>
  ```
- Mention the updates to the user at the end of the answer.

If no new insights emerge, do not modify any files.

### Archiving

When the user explicitly asks to archive or save the answer to the wiki:

1. Write the answer as a new wiki page. See `references/archive-template.md`. When converting conversation citations to the archive page, rewrite project-root-relative paths (e.g., `.wiki/topic/article.md`) to file-relative paths (e.g., `../topic/article.md` or `article.md` for same-directory).
   - YAML frontmatter with `tags` including `archive`.
   - Sources in frontmatter: markdown links to the wiki articles cited in the answer.
   - No Raw field (content does not come from raw/).
   - File name reflects the query topic, e.g., `transformer-architectures-overview.md`.
   - Place in the most relevant topic directory.
2. Always create a new page. Never merge into existing articles (archive content is a synthesized answer, not raw material).
3. Update `.wiki/index.md`. Prefix the Summary with `[Archived]`.
4. Append to `.wiki/log.md`:
   ```
   ## [YYYY-MM-DD] query | Archived: <page title>
   ```

---

## Insight Mining

Undirected exploration across the entire wiki to discover non-obvious connections, hypotheses, and conclusions. If Query is a targeted search for a specific question, Insight Mining is its parallel random walk counterpart.

**Triggers:**
- "insight mining"
- "find insights from wiki"
- "wiki insights"
- "analyze patterns"

### Process

**Phase 1: Comprehensive Read**

1. Read `.wiki/index.md` to map the full wiki landscape.
2. Read **all** wiki articles (excluding raw/). For large wikis, use the Agent tool for parallel reads.
3. From each article, extract and note:
   - Core claims and their supporting evidence
   - Stated limitations and open problems
   - Explicit and implicit connections to other articles
   - Recurring concepts, methods, and datasets

**Phase 2: Random Walk Exploration**

Follow inter-article connections along multiple paths simultaneously. Along each path, attempt the following:

- **Analogical reasoning**: Are there structurally similar problems/solutions across different topics?
- **Gap detection**: Does article A mention a challenge that article B has already solved, without either being aware?
- **Contradiction surfacing**: Do different articles make conflicting claims about the same phenomenon?
- **Trend extrapolation**: When articles are arranged chronologically, does a clear direction emerge?
- **Absent negative**: Are there important areas the wiki does not cover? (Concepts repeatedly mentioned across articles but lacking a dedicated page)
- **Hypothesis generation**: Can combining evidence from multiple articles yield an untested hypothesis?

**Phase 3: Insight Formulation**

Classify discovered insights into the following types:

| Type | Description | Example |
|------|-------------|---------|
| **Connection** | Newly discovered relationship between previously unlinked articles/concepts | "Body placement's VAE approach is structurally identical to SAMP's goal pose generation" |
| **Contradiction** | Conflicting claims across articles | "Article A claims occupancy is sufficient, but article B argues fine-grained contact is essential" |
| **Gap** | Important area not covered by the wiki | "All articles mention lack of evaluation standardization, but no dedicated article exists" |
| **Trend** | Temporal or methodological evolution direction | "2021-2024: all subfields shift from task-specific to foundation model utilization" |
| **Hypothesis** | Evidence-based untested hypothesis | "A unified contact + occupancy representation could resolve data scarcity" |

### Output

**Conversation output:**
- Report discovered insights organized by type
- Cite supporting wiki articles with inline links for each insight
- Indicate novelty and confidence for each insight: `[High/Medium/Low]`

**Wiki updates (automatic):**
- Discovered Connections → add cross-links to relevant articles' See Also sections
- Discovered Contradictions → add `**[Conflict]**` annotations to relevant articles
- New Synthesis patterns → create new pages in `.wiki/synthesis/` or update existing ones
- Update existing Synthesis pages if their content is no longer valid

**Not updated in wiki:**
- Gaps and Hypotheses are reported in conversation only. Do not write speculative content to the wiki.
- Do not rewrite existing article body text. Only add links and annotations.

**Logging:**
```
## [YYYY-MM-DD] insight-mining | <N> insights found
- Connection: <one-line description>
- Contradiction: <one-line description>
- Gap: <one-line description>
- Trend: <one-line description>
- Hypothesis: <one-line description>
- Updated: <modified article title>
- Created: synthesis/<new-page>.md (if any)
```

---

## Lint

Quality checks on the wiki. Two categories with different authority levels.

### Deterministic Checks (auto-fix)

Fix these automatically:

**Index consistency** — compare `.wiki/index.md` against actual `.wiki/` files (excluding index.md, log.md, raw/, and .obsidian/):
- File exists but missing from index → add entry with `(no summary)` placeholder. For Updated, use the article's frontmatter `updated` date if present; otherwise fall back to `created` date.
- Index entry points to nonexistent file → mark as `[MISSING]` in the index. Do not delete the entry; let the user decide.

**Internal links** — for every markdown link in `.wiki/` article files (body text and Sources metadata), excluding Raw field links and excluding index.md/log.md:
- Target does not exist → search `.wiki/` for a file with the same name elsewhere.
  - Exactly one match → fix the path.
  - Zero or multiple matches → report to the user.

**Raw references** — every link in a Raw field must point to an existing `.wiki/raw/` file:
- Target does not exist → search `.wiki/raw/` for a file with the same name elsewhere.
  - Exactly one match → fix the path.
  - Zero or multiple matches → report to the user.

**See Also** — within each topic directory:
- Add obviously missing cross-references between related articles.
- Remove links to deleted files.

**Frontmatter** — every article in `.wiki/<topic>/` must have valid YAML frontmatter:
- Missing frontmatter → add with `tags: []`, `created: <today>`, `updated: <today>`.
- Missing `tags` field → add `tags: []`.
- Missing `created`/`updated` → add with today's date.

### Heuristic Checks (report only)

Report findings without auto-fixing:

- Factual contradictions across articles
- Outdated claims superseded by newer sources
- Missing conflict annotations where sources disagree
- Orphan pages with no inbound links from other wiki articles
- Missing cross-topic references
- Concepts frequently mentioned but lacking a dedicated page
- Archive pages whose cited source articles have been substantially updated since archival
- Weak graph connectivity (isolated clusters that should be linked)
- Topic directories with 3+ articles but no `_overview.md`
- Cross-topic patterns (3+ topics sharing a technique) without a corresponding synthesis page
- Articles missing the required page-type tag (`summary`, `concept`, `entity`, `comparison`, `overview`, `synthesis`, `archive`)

### Post-Lint

Append to `.wiki/log.md`:

```
## [YYYY-MM-DD] lint | <N> issues found, <M> auto-fixed
```

---

## Conventions

- Standard markdown with relative links throughout. Obsidian-compatible.
- `.wiki/` supports one level of topic subdirectories only. No deeper nesting.
- YAML frontmatter on every wiki article with at minimum `tags`, `created`, `updated` fields.
- Today's date for log entries, Collected dates, and Archived dates. Updated dates reflect when the article's knowledge content last changed. Published dates come from the source (use `Unknown` when unavailable).
- Inside `.wiki/` files, all markdown links use paths relative to the current file. In conversation output, use project-root-relative paths (e.g., `.wiki/topic/article.md`).
- Ingest updates both `.wiki/index.md` and `.wiki/log.md`. Research runs batch Ingest internally (same update rules apply). Archive (from Query) updates both. Lint updates `.wiki/log.md` (and `.wiki/index.md` only when auto-fixing index entries). Queries may perform insight-driven updates to existing articles (see Query > Insight-Driven Updates).
- Tag taxonomy: lowercase, hyphenated. Every article must include exactly one page-type tag (`summary`, `concept`, `entity`, `comparison`, `overview`, `synthesis`, `archive`). Additional domain tags freely: `method`, `dataset`, `guide`, `benchmark`, etc.
- Excluded from Obsidian graph/search by convention: `.wiki/raw/`, `.wiki/log.md`. Configure via Obsidian Settings → Files and links → Excluded files if desired.
