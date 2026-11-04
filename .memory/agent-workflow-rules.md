---
name: agent-workflow-rules
description: Why the user mandates brainstorm → decisions → phase split → per-phase plan summary → detailed plan → review → code → test, evidence-only work, rule check and progress report after each phase
metadata:
  type: feedback
---

The steps are canonical in CLAUDE.md ("Agent Workflow (Mandatory)"). The user set them on 2026-10-24.

- Evidence only: work that cannot be checked against a PRD copied into `docs/` or a user decision will not be approved. An incomplete component breakdown is also rejected, so unclear scope is discussed before planning.
- Phase split before plan summary: long plans tend to change mid-execution and drag unrelated details with them; one summary per phase isolates that change.
- Rule check after big tasks and progress report after each phase: keep rules current and progress visible.

**Why:** the user wants verifiable, approvable work and small, isolated plans that survive change.
**How to apply:** follow CLAUDE.md "Agent Workflow" for every non-trivial task; when evidence is missing, stop and present purpose + component breakdown + questions instead of guessing.
