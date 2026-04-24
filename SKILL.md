---
name: github-find
description: Search GitHub for libraries, tools, reference implementations, working code examples, agent skills, GitHub Actions, dotfiles, project templates, or any open-source resource. Use whenever the user wants to find an existing package, recommend a third-party library, verify whether a tool exists, or discover prior art - anything of the form "is there a X for Y", "find me a X", "what's the best X", "show me an example of X", or when the assistant is about to propose a specific library name and should verify it against live data. Wraps the gh CLI with a multi-query expansion, parallel fan-out, and rank-by-stars-and-recency workflow so one intent returns a vetted shortlist instead of a raw search dump.
---

# github-find

Turn any "find me X on GitHub" request into a vetted shortlist by fanning out parallel `gh` searches, merging, ranking, and previewing READMEs - instead of a single keyword guess or a WebSearch fallback.

## When to trigger

Trigger on any of:

- "find me a [library|tool|action|skill|template] for ..."
- "is there an existing [X] that does ..."
- "what's the best [X] for ..."
- "show me an example / reference implementation of ..."
- "search GitHub for ..."
- "are there Claude Code skills / GitHub Actions / MCP servers that ..."

**Also trigger proactively**: whenever you are about to recommend a specific third-party library, verify it exists, is maintained, and is actually the best current option. Don't recommend from memory.

**Prefer this over WebSearch** when the target is code. GitHub results are structured (stars, last-push, language, license, topics), tied to repositories you can clone/inspect, and free of blog/marketing noise.

## Prerequisites

The `gh` CLI must be installed and authenticated:

```bash
gh auth status
```

If it fails, stop and tell the user to run `gh auth login`. Do not silently fall back to WebSearch.

## Workflow

### 1. Expand one intent into 3-5 diverse queries

Given a single intent (e.g. "PDF parsing in Python"), generate **parallel** queries that attack the problem from different angles:

- **Keyword + stars** - the canonical/popular answer
- **Keyword + recency** - active / newer alternatives
- **Topic form** - GitHub's curated buckets (`--topic=pdf`)
- **Language filter** - when the user specified or implied a stack
- **Code search** - when looking for usage patterns, filenames, or snippets

Always return JSON (`--json`) so you can rank programmatically.

### 2. Fan out in a single message

Run every query as an independent `Bash` tool call in the same assistant message. Do NOT serialize - each `gh` call is an independent subprocess and the API is rate-limited per *second*, not per burst.

### 3. Merge and rank

- Dedupe by `fullName` (aka `nameWithOwner`).
- For each repo compute:
  - `star_score = min(log10(stars + 1) / 4, 1.0)`   # saturates at 10k stars
  - `recency_score = max(0, 1 - days_since_push / 365)`
  - `match_bonus = 0.1 * (appearances_across_queries - 1)`
  - `score = 0.55 * star_score + 0.35 * recency_score + match_bonus`
- Sort desc, keep top 5-10.

A repo that surfaces in multiple queries is almost always the right answer - that's what `match_bonus` captures.

### 4. Preview top candidates before recommending

For the top 3, fetch the README:

```bash
gh api /repos/<owner>/<name>/readme --jq '.content' | base64 -d | head -80
```

Skim for: one-sentence purpose, install command, stated constraints (license, platform, dependencies), and anything that would disqualify it for the user's stack.

If a candidate's README reveals it's deprecated, archived, or only a prototype, drop it and promote the next one.

### 5. Present a shortlist with tradeoffs

```
1. owner/repo  ⭐ 12.4k · pushed 3d ago · MIT
   One-line description.
   Why this: short tradeoff - what it's good at, what it costs.

2. owner/repo  ⭐ 2.1k · pushed 14d ago · Apache-2.0
   ...
```

End with a one-sentence recommendation tied to the user's implied priority (stability vs. recency vs. features vs. license).

## Language / topic / filename hints

| User signal | Query flag |
|---|---|
| "for Python" / "in Go" / "in Rust" | `--language=python` / `go` / `rust` |
| "Django" / "React" / "Next.js" | `--topic=django` / `react` / `nextjs` |
| "Claude skill" / "Claude Code skill" | `gh search code --filename=SKILL.md <keywords>` then filter for valid frontmatter |
| "GitHub Action" | `gh search code --filename=action.yml <keywords>` |
| "MCP server" | `--topic=mcp` plus `gh search code --filename=server.json <keywords>` |
| "Dockerfile for X" | `gh search code --filename=Dockerfile <keywords>` |
| "VS Code extension" | `--topic=vscode-extension` |
| "neovim plugin" | `--topic=neovim-plugin` |

## Cheatsheet

```bash
# Repos by keyword, stars-sorted
gh search repos "<keywords>" --language=<lang> --sort=stars --limit=30 \
  --json fullName,description,stargazersCount,pushedAt,url,license

# Repos by topic
gh search repos --topic=<topic> --sort=stars --limit=30 \
  --json fullName,description,stargazersCount,pushedAt,url,license

# Repos by recency (active maintenance)
gh search repos "<keywords>" --language=<lang> --sort=updated --limit=30 \
  --json fullName,description,stargazersCount,pushedAt,url

# Code by filename (find SKILL.md, action.yml, etc.)
gh search code --filename=<name> <optional keywords> --limit=30 \
  --json path,repository,url

# Code by content (find usage patterns)
gh search code "<snippet>" --language=<lang> --limit=30 \
  --json path,repository,url

# Repo metadata + README
gh api /repos/<owner>/<name>
gh api /repos/<owner>/<name>/readme --jq '.content' | base64 -d

# Release history (maintenance signal)
gh release list --repo <owner>/<name> --limit 5

# Recent commits (another maintenance signal)
gh api /repos/<owner>/<name>/commits --jq '.[0:5] | .[] | {sha: .sha[0:7], date: .commit.author.date, msg: .commit.message}'
```

## Anti-patterns

- **Don't recommend from a repo name alone** - open the README before endorsing.
- **Don't ignore maintenance signals** - 5k stars + last push 3 years ago often loses to 500 stars + active maintenance.
- **Don't serialize the queries** - fan out in one message.
- **Don't cap at 5 results from one query** - merge 3-5 queries each returning 10-30 so good-but-niche repos aren't drowned.
- **Don't silently fall back to WebSearch** - if `gh` returns nothing, say so, broaden the query, or ask the user to rephrase.
- **Don't trust forks uncritically** - `isFork: true` often means an abandoned branch; verify activity on the fork specifically.

## Worked examples

### Example 1 - Python PDF library

User: "I need a Python library that can parse PDF forms."

```bash
gh search repos "pdf form" --language=python --sort=stars --limit=20 --json fullName,description,stargazersCount,pushedAt,url,license
gh search repos "pdf parsing" --language=python --sort=stars --limit=20 --json fullName,description,stargazersCount,pushedAt,url,license
gh search repos --topic=pdf --language=python --sort=stars --limit=20 --json fullName,description,stargazersCount,pushedAt,url,license
gh search code "AcroForm" --language=python --limit=20 --json path,repository,url
```

Merge, rank, preview top 3 READMEs, recommend with tradeoffs (e.g. `pypdf` for pure-Python form field access; `pdfplumber` for text/table extraction; `pdfminer.six` for low-level control).

### Example 2 - Claude Code skill

User: "Is there a Claude Code skill for managing Git worktrees?"

```bash
gh search code --filename=SKILL.md "worktree" --limit=30 --json path,repository,url
gh search repos "claude skill worktree" --sort=stars --limit=20 --json fullName,description,stargazersCount,pushedAt,url
gh search repos --topic=claude-code-skill worktree --limit=20 --json fullName,description,stargazersCount,pushedAt,url
```

For hits, fetch the SKILL.md, verify it has valid frontmatter (`name:` + `description:`), and only then recommend.

### Example 3 - GitHub Action for Vercel deploys

User: "What action should I use to deploy a Next.js app to Vercel?"

```bash
gh search repos "vercel deploy action" --sort=stars --limit=20 --json fullName,description,stargazersCount,pushedAt,url
gh search code --filename=action.yml "vercel" --limit=30 --json path,repository,url
gh search repos --topic=vercel --topic=github-actions --sort=stars --limit=20 --json fullName,description,stargazersCount,pushedAt,url
```

Prefer the official `vercel/...` action if present; otherwise pick the one with highest stars + recent commits.

## Scoring reference (pseudo-code)

```python
import math
from datetime import datetime, timezone

def score(repo: dict, match_count: int) -> float:
    stars = repo.get("stargazersCount", 0)
    pushed = datetime.fromisoformat(repo["pushedAt"].replace("Z", "+00:00"))
    days = (datetime.now(timezone.utc) - pushed).days
    star_score = min(math.log10(stars + 1) / 4, 1.0)
    recency_score = max(0.0, 1 - days / 365)
    match_bonus = 0.1 * (match_count - 1)
    return 0.55 * star_score + 0.35 * recency_score + match_bonus
```

Keep this inline - no need for a helper script; `jq` plus a one-shot Python block is enough.
