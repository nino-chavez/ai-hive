# AI Hive

A coordination layer that turns multiple human-agent pairs into a unified development team — shared memory, task ownership, and conflict-free parallel execution across machines.

**Status:** Architecture RFC. Nothing built yet. This repo currently holds the design artifacts.

## The Gap

Current agent tooling assumes 1 human + 1 agent + 1 session. The solo-composer problem is well-served (CrewAI, AutoGen, LangGraph, Claude Code subagents, Switchman, Conductor). The multi-composer problem — multiple solo devs, each with their own agent swarm, working the same codebase — is not. File locks prevent conflicts; they don't coordinate intent.

AI Hive targets that gap with three primitives:

- **Blackboard** — Supabase as shared state with realtime sync
- **MCP server** — agent-agnostic tools (`hive/claim_task`, `hive/log_decision`, `hive/fit_check`)
- **Constitution files** — `CLAUDE.md` / `AGENTS.md` / `decisions.md` as the canonical context layer

The repo is the source of truth. The coordination layer degrades gracefully — if Supabase goes down, the team falls back to normal git workflows.

## Documents

- [`docs/architecture.md`](docs/architecture.md) — full RFC: design principles, component diagrams, MCP tool surface, schema, phased build plan, competitive landscape.
- [`docs/document-resonance.md`](docs/document-resonance.md) — meta-analysis across five inputs (research synthesis, practitioner conversation, Gemini Deep Research, architecture doc, tool verification). Identifies what converged, what created productive tension, what was absent, and what to build first.

## Why this repo exists publicly

To get sharp eyes on the design before code is written. The interesting open questions are in `document-resonance.md` under "What Was Absent" — onboarding mid-project, cost management across composers, testing coordination, human-to-human communication, and the veto mechanism.

Feedback welcome via issues.
