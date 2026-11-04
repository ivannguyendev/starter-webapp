# Repo memory index

Shared, agent-neutral memory for this repo (Claude Code and other agents). Tracked in git.
Rules live in CLAUDE.md; these files keep context and the *why*. One fact per file, frontmatter `name/description/metadata.type` (`user | feedback | project | reference`), link with `[[name]]`. Update an existing file instead of adding a duplicate; delete memories proven wrong. Never store secrets or links to transient input files.

- [Language preferences](language-preferences.md) — reply Vietnamese; README VN; CLAUDE.md + memory EN
- [Agent workflow rules](agent-workflow-rules.md) — why: evidence-only, phase split before plan summary, rule check + progress report
