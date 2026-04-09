# llm-wiki

A Claude Code skill implementing [Andrej Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) pattern — the LLM maintains a persistent, compounding wiki between you and your raw sources.

## How to install

```bash
git clone git@github.com:highsummer/llm-wiki.git .claude/skills/llm-wiki
```

## Usage

Tell Claude Code to ingest a source — a URL, a paper, a file. It creates `.wiki/` at the project root, saves the raw source, and compiles it into a wiki article. Subsequent ingests update existing articles and add new ones. The wiki grows from there.

```
"ingest https://arxiv.org/abs/1803.10122"    # add a source
"what do I know about world models?"      # query the wiki
"insight mining"                              # explore for non-obvious patterns
"lint wiki"                                   # health-check
```

The `.wiki/` directory is an [Obsidian](https://obsidian.md/) vault. Open it in Obsidian to browse articles, follow cross-references, and use graph view. The LLM writes; you read.

## Changes from the original

The original is intentionally abstract. This skill fills in implementation details, mostly discovered through use in an academic research context.

- **Page type system.** Seven explicit types (`summary`, `concept`, `entity`, `comparison`, `overview`, `synthesis`, `archive`) with creation triggers, so the wiki self-organizes rather than becoming a flat list of summaries.
- **Overview and synthesis pages.** Auto-generated when topic directories reach 3+ articles (overview) or a pattern spans 3+ topics (synthesis). The "compounding" part of the original that didn't happen naturally without explicit rules.
- **Insight Mining.** A fourth operation alongside Ingest/Query/Lint. Undirected exploration across all articles to surface non-obvious connections, contradictions, gaps, and hypotheses.
- **Insight-driven Query updates.** Query can now write back cross-article connections and corrections it discovers while answering, instead of being read-only.
- **OpenReview collection.** For paper sources, appends public peer reviews to the raw file during Ingest.
- **Deterministic lint.** Lint split into auto-fixable checks (broken links, missing index entries) and heuristic checks (reported only).

## Acknowledgments

Core idea by [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).
