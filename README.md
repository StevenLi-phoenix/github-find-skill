<!-- cover image: assets/cover.png (generated after initial push) -->
<p align="center">
  <img src="assets/cover.png" alt="github-find: the agent skill that makes GitHub the marketplace" width="720" />
</p>

# github-find

A **Claude Code Agent Skill** that makes the agent reach for `gh` CLI whenever the user wants to find something on GitHub - a library, a reference implementation, a Claude Code skill, a GitHub Action, a dotfile, anything.

No new CLI. No new marketplace. Just a prompt-level wrapper that turns "find me X" into a parallel multi-query search over real GitHub data, ranked by stars and recency, previewed with READMEs, recommended with tradeoffs.

> **Why this exists**: GitHub already *is* the largest agent-skill marketplace (and library marketplace, and action marketplace, and template marketplace). The gap isn't infrastructure - it's the instinct to search it well. This skill embeds that instinct.
>
> Background: [GitHub 是最大的 skill 商城](https://blog.lishuyu.top/posts/github%E6%98%AF%E6%9C%80%E5%A4%A7%E7%9A%84skill%E5%95%86%E5%9F%8E/)

## What the skill does

When triggered, it:

1. **Expands** one user intent into 3-5 parallel `gh search` queries (keyword + stars, keyword + recency, topic form, language-filtered, code-search when appropriate).
2. **Fans out** all queries in a single message (parallel `Bash` tool calls).
3. **Merges and ranks** deduped results by a composite of `log(stars)`, recency of last push, and how many queries a repo appeared in.
4. **Previews** top 3 READMEs via `gh api /repos/.../readme`.
5. **Recommends** a shortlist with one-line tradeoffs, tied to the user's implied priority (stability / recency / features / license).

## Install

### As a personal skill (Claude Code)

```bash
git clone https://github.com/StevenLi-phoenix/github-find-skill.git ~/.claude/skills/github-find
```

Claude Code discovers SKILL.md files under `~/.claude/skills/` automatically; no restart needed.

### As a project skill

```bash
git clone https://github.com/StevenLi-phoenix/github-find-skill.git .claude/skills/github-find
```

Checks in a `.claude/skills/github-find/` alongside your project so teammates get the same behavior.

## Prerequisites

- [`gh` CLI](https://cli.github.com/) installed
- Authenticated: `gh auth status` must pass (run `gh auth login` otherwise)

The skill will stop and ask you to authenticate rather than silently falling back.

## Usage

You don't invoke the skill directly - Claude triggers it on phrases like:

- "find me a Python library that parses PDF forms"
- "is there a Claude Code skill for managing Git worktrees"
- "what action should I use to deploy a Next.js app to Vercel"
- "recommend a Rust CLI framework"
- "show me reference implementations of a Raft consensus algorithm in Go"

See [SKILL.md](./SKILL.md) for the full trigger list, workflow, and examples.

## Disclaimer

This skill is provided as-is, with no warranty. Before using it, understand:

- **It is a prompt-level wrapper, not a sandbox.** It tells Claude to run `gh` CLI commands on your behalf. Those commands use your existing `gh` authentication and are subject to your token's scopes and GitHub's rate limits. Review [SKILL.md](./SKILL.md) in full before installing.
- **By default it only reads GitHub** (search + README fetch). It never clones a repo, installs a package, or runs code without asking.
- **Persisting any found resource to disk is opt-in.** A found repo might be a Claude Code skill, an MCP server, a CSS/HTML template, a dotfile bundle, a library, a GitHub Action, or a one-off reference. If you ask to install / clone / "固化" one, the skill will detect the resource type, show you the exact command and destination, and wait for your explicit "yes" before running. See the *Install / persist a found resource (consent-gated)* section of [SKILL.md](./SKILL.md). No silent clones. No batch installs. Package-manager commands (`pip install`, `npm install`, `cargo install`, `docker pull`, ...) are surfaced but never executed on your behalf.
- **GitHub search is a popularity + recency signal, not a quality signal.** A well-ranked result is not a vouched-for result. Audit any repo before importing it into your project, especially anything that runs at build time, ships binaries, or asks for credentials.
- **You are responsible for license compatibility.** The skill surfaces `license` where GitHub exposes it, but absence of a license field does not mean the repo is permissively licensed. Verify before use.
- **Third-party skills (including this one) should themselves be audited.** See [Anthropic's security guidance on Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview#security-considerations). Read the SKILL.md, understand the tool calls it prescribes, and only install from sources you trust.
- **Not affiliated with GitHub or Anthropic.** "GitHub" and "Claude" are trademarks of their respective owners.

## License

MIT - see [LICENSE](./LICENSE).
