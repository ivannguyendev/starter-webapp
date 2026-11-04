# CLAUDE.md

Guidance for Claude Code and other AI agents working in this repository.

This file has two parts:

- **Project profile** (Project, Architecture & Commands, Testing): facts about this specific project. Empty (`TBD`) until they are decided and approved, then filled during bootstrap.
- **Core rules** (Input & Docs through Gotchas): apply to every project built from this starter. Change them only on an explicit user decision.

## Project Bootstrap

While any Project profile section is still `TBD`, the project is not bootstrapped.

1. Treat the dev's first request as the bootstrap task and run the Agent Workflow on it.
2. Do not scaffold, install packages or pick a stack until the brainstorm (step 1) approves the project purpose and stack.
3. Fill each profile section only when its facts are approved, in the close-out (step 9) of the phase that decided them. Never fill a section with guesses.
   - Project: after the purpose is approved.
   - Architecture & Commands: once the scaffold exists.
   - Testing: once the test stack is chosen.
4. Replace the starter name (`starter-webapp`) with the project name in README.md and in this file.

## Project

> **TBD (bootstrap):** product purpose, target users and roles, core flows, measurable quality goals, out of scope.

## Architecture & Commands

> **TBD (after scaffold):** stack and versions, folder layout (one line per folder), config locations, key commands (dev, build, lint, typecheck, test variants). List only folders and commands that exist.

## Testing

> **TBD (when test stack is chosen):** test layers and where each lives, which behavior goes to which layer, test data isolation, setup/teardown approach, environments and which of them allow writes, PR gate time budget.

## Input & Docs

- Input material (PRDs, mockups, API docs) goes in `.local/`. It is transient and gitignored, and can be deleted at any time.
- Never link to `.local/` files from any tracked file (code, docs, CLAUDE.md, README, `INDEXES.csv`, `.memory/`).
- Copy permanent requirements into `docs/` and exact UI copy into constants in the code.
- `docs/` holds finalized docs only, including phase summaries in `docs/plans/`.

## Code index

`INDEXES.csv` at the repo root is the context-search entry point (columns `path,kind,purpose,key_symbols,used_by,tests`). Read it before searching the tree.

- **Row:** one file, folder or config, with the contexts that can be retrieved from it (listed in `purpose`/`key_symbols`). No line numbers.
- **Scope:** only official files, folders, configs and docs that are confirmed correct and useful as retrieval context. Never index transient inputs (`.local/`), plans or drafts.
- **Update:** for every new or modified file, check whether it is in scope or changes an existing row. If so, add or update the row in the same change.

## Repo Memory

`.memory/` is the shared, tracked memory for all agents: context, decisions and the *why* behind rules. Read `.memory/MEMORY.md` at the start of every session.

- **Propose, never silently commit:** agents must never autonomously create or edit `.memory/` files. Any addition or update must be proposed as a draft to the user for explicit approval ("go" / "ok").
- **Evidence-based only:** only store confirmed user decisions, lasting architectural choices, and permanent preferences. Never store agent hypotheses, unconfirmed assumptions, scratchpad notes, or transient discussions.
- **The Why, not The How:** rules belong in this file (`CLAUDE.md`). `.memory/` explains the *why* and context; it must never store implementation details, code snippets, or duplicates of rules.
- **Content standard:** one fact per file, frontmatter `name/description/metadata.type` (`user | feedback | project | reference`), and an index line in `.memory/MEMORY.md`. Update an existing file instead of duplicating it.
- **Security & boundaries:** never store secrets, tokens, credentials, or links to transient input files (`.local/`).
- **Pruning & ground truth:** code and official documentation (`docs/`) are ground truth over memory. If a memory conflicts with code or is obsolete, report it to the user to prune or update; never silently follow stale memory.

## Agent Workflow (Mandatory)

Applies to every non-trivial task: project bootstrap, new feature, refactor, standards change. `<agent dir>` is `.claude` for Claude and `.agent` for other agents. `YYMMDD` comes from `date +%y%m%d`.

1. **Brainstorm:** read README, CLAUDE.md, `.memory/`, `INDEXES.csv` and the relevant docs.
   - **Evidence first:** every requirement and design decision must trace to a source: requirements copied into `docs/`, or an explicit user decision. Never invent behavior, UI copy, limits or endpoints.
   - If the source is missing or unclear, stop and present the goal, a proposed component breakdown and the open questions. An incomplete breakdown will not be approved.
2. **Save decisions:** write `<agent dir>/plans/YYMMDD-<feature>-design.md` with decisions, assumptions, rejected alternatives (with why) and open risks.
3. **Reviewer: phase split:** before any plan summary, a reviewer assesses the scope and discusses with the user whether to split it into milestones/phases. Each phase must be deliverable and verifiable on its own, so later changes stay isolated.
4. **Plan summary (current phase only):** write `docs/plans/YYMMDD-<feature>-p<N>-summary.md` with goal, scope/out-of-scope, deliverables, acceptance criteria, evidence sources and risks. The user approves it.
5. **Detailed plan:** write `<agent dir>/plans/YYMMDD-<feature>-p<N>.md`, only after the phase summary is approved.
6. **Review:** a reviewer checks the detailed plan against the summary and evidence. Resolve all Critical/Important findings before coding.
7. **Code:** follow the detailed plan and report every deviation.
8. **Test:**
   - Run the narrowest command and report exit code and pass/fail/flaky.
   - Update the phase summary to match the actual build.
   - Update `INDEXES.csv` per "Code index".
9. **Close-out:**
   - Rule check: look for new rules, conventions or decisions the user confirmed during the task. Update CLAUDE.md with rules, including Project profile sections now decided. Propose confirmed decisions from the design log to the user as draft memories for `.memory/` (do not commit without explicit user approval), since `<agent dir>/plans/` is gitignored. State what changed, or "no new rules".
   - Progress: state the phase just finished, the next phase, and all remaining phases.

## Definition of Done for Merging to Main

1. **Plan & Summary Completion:** all tasks in the phase plan (`<agent dir>/plans/YYMMDD-<feature>-p<N>.md`) are complete, and `docs/plans/YYMMDD-<feature>-p<N>-summary.md` reflects the actual build. Any steps that deviated from "Expected" have been reported and resolved.
2. **All Green:** the lint, typecheck and test commands listed in "Architecture & Commands" all pass cleanly.
3. **No Critical / Important Findings:** zero unresolved Critical or Important findings. All Minor findings are documented in the plan or docs.
4. **Go-Live Readiness in PR:** the PR description states the readiness level and any remaining items required before production go-live.

## Gotchas

- Keep UI labels, button names and specialized terms in their original language.
- Decision logs and detailed plans belong in `.claude/plans/` or `.agent/plans/`. They are gitignored like `.local/`; never un-ignore them. Never put detailed plans in `docs/`.
