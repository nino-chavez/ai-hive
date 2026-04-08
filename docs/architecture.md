# AI Hive — Architecture Design Document

**Version:** 0.1.0-draft
**Date:** 2026-04-08
**Author:** Nino Chavez
**Status:** RFC (Request for Comments)

---

## 1. Problem Statement

Modern software development increasingly involves multiple humans working alongside AI coding agents. Current tooling assumes a 1:1 relationship — one human, one agent, one session. This breaks down when a team of developers, each with their own agent, needs to collaborate on a single codebase without:

- Agents duplicating or contradicting each other's work
- Merge conflicts from parallel, uncoordinated edits
- Context fragmentation across sessions and machines
- Loss of architectural decisions between sessions

AI Hive is a coordination layer that turns a group of independent human-agent pairs into a unified development team with shared memory, task ownership, and conflict-free parallel execution.

---

## 2. Design Principles

1. **Repository as source of truth.** The Git repo is the canonical state. All coordination artifacts live in the repo or reference it directly.
2. **Infrastructure-light.** A single Supabase project provides auth, database, realtime, and edge functions. No Redis, no Kafka, no custom servers at MVP.
3. **Agent-framework agnostic.** Any agent that can read files and call an MCP server can participate. No lock-in to Claude Code, Cursor, Copilot, or any specific tool.
4. **Human-in-the-loop by default.** Agents propose; humans approve. The system makes it easy to supervise, not easy to ignore.
5. **Graceful degradation.** If the coordination layer goes down, developers fall back to normal Git workflows. Nothing is lost.

---

## 3. System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AI HIVE SYSTEM                              │
│                                                                     │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                       │
│  │ Machine A │  │ Machine B │  │ Machine C │    ... N participants  │
│  │           │  │           │  │           │                        │
│  │ Human +   │  │ Human +   │  │ Human +   │                        │
│  │ Agent     │  │ Agent     │  │ Agent     │                        │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                       │
│        │              │              │                              │
│        └──────────────┼──────────────┘                              │
│                       │                                             │
│              ┌────────▼────────┐                                    │
│              │   MCP Server    │  Streamable HTTP                   │
│              │  (Shared Tools) │                                    │
│              └────────┬────────┘                                    │
│                       │                                             │
│              ┌────────▼────────┐                                    │
│              │    Supabase     │  Blackboard State                  │
│              │  (Shared Brain) │  Realtime Sync                     │
│              └─────────────────┘                                    │
│                                                                     │
│              ┌─────────────────┐                                    │
│              │   Git Repo      │  Worktrees + Constitution          │
│              │ (Source of Truth)│  decisions.md (append-only)        │
│              └─────────────────┘                                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Architecture Diagrams

### 4.1 High-Level Component Architecture

```mermaid
graph TB
    subgraph Participants
        MA[Machine A<br/>Human + Agent]
        MB[Machine B<br/>Human + Agent]
        MC[Machine C<br/>Human + Agent]
    end

    subgraph Coordination Layer
        MCP[MCP Server<br/>Streamable HTTP]
        SB[Supabase<br/>Blackboard DB + Realtime]
    end

    subgraph Repository
        GIT[Git Remote<br/>GitHub / GitLab]
        WTA[Worktree A<br/>feature/auth]
        WTB[Worktree B<br/>feature/api]
        WTC[Worktree C<br/>feature/ui]
    end

    MA -->|MCP Client| MCP
    MB -->|MCP Client| MCP
    MC -->|MCP Client| MCP

    MCP -->|Read/Write State| SB

    MA -->|git push/pull| GIT
    MB -->|git push/pull| GIT
    MC -->|git push/pull| GIT

    GIT --- WTA
    GIT --- WTB
    GIT --- WTC

    style MCP fill:#4f46e5,color:#fff
    style SB fill:#3ecf8e,color:#fff
    style GIT fill:#f97316,color:#fff
```

### 4.2 Blackboard Architecture — State Flow

The blackboard is the central shared state. Agents don't talk to each other — they read from and write to the blackboard. Supabase Realtime pushes changes to all subscribers.

```mermaid
graph LR
    subgraph Blackboard ["Supabase Blackboard"]
        TASKS[Tasks Table<br/>status, owner, priority]
        DECISIONS[Decisions Table<br/>append-only log]
        SESSIONS[Sessions Table<br/>active agents, heartbeat]
        LOCKS[Locks Table<br/>file-level advisory locks]
    end

    A1[Agent 1] -->|claim task| TASKS
    A1 -->|log decision| DECISIONS
    A1 -->|register session| SESSIONS
    A1 -->|acquire lock| LOCKS

    A2[Agent 2] -->|read tasks| TASKS
    A2 -->|read decisions| DECISIONS
    A2 -->|check sessions| SESSIONS
    A2 -->|check locks| LOCKS

    TASKS -.->|Realtime notify| A1
    TASKS -.->|Realtime notify| A2
    DECISIONS -.->|Realtime notify| A1
    DECISIONS -.->|Realtime notify| A2

    style TASKS fill:#3ecf8e,color:#fff
    style DECISIONS fill:#3ecf8e,color:#fff
    style SESSIONS fill:#3ecf8e,color:#fff
    style LOCKS fill:#3ecf8e,color:#fff
```

### 4.3 Task Lifecycle — State Machine

```mermaid
stateDiagram-v2
    [*] --> Open: Task created

    Open --> Claimed: Agent claims task
    Open --> Blocked: Dependency unmet

    Claimed --> InProgress: Agent starts work
    Claimed --> Open: Agent releases (timeout / crash)

    InProgress --> Review: Agent pushes branch
    InProgress --> Blocked: Discovers dependency
    InProgress --> Open: Agent crashes (heartbeat timeout)

    Blocked --> Open: Dependency resolved

    Review --> Approved: Human approves PR
    Review --> InProgress: Human requests changes

    Approved --> Merged: CI passes + merge
    Approved --> Review: CI fails

    Merged --> [*]
```

### 4.4 Parallel Execution with Git Worktrees

```mermaid
sequenceDiagram
    participant H as Human (Lead)
    participant BB as Blackboard
    participant A1 as Agent 1
    participant A2 as Agent 2
    participant A3 as Agent 3
    participant GIT as Git Remote

    H->>BB: Create tasks T1, T2, T3
    
    par Agents claim work
        A1->>BB: Claim T1 (auth module)
        A2->>BB: Claim T2 (API routes)
        A3->>BB: Claim T3 (UI components)
    end

    par Isolated execution
        A1->>GIT: Create worktree → feature/t1-auth
        A1->>A1: Implement in isolation
        A1->>BB: Log decision: "Using JWT with httpOnly cookies"
    and
        A2->>GIT: Create worktree → feature/t2-api
        A2->>BB: Read decisions (sees JWT choice)
        A2->>A2: Implement API with JWT validation
    and
        A3->>GIT: Create worktree → feature/t3-ui
        A3->>A3: Implement UI components
    end

    A1->>GIT: Push feature/t1-auth
    A1->>BB: Move T1 → Review

    A2->>GIT: Push feature/t2-api
    A2->>BB: Move T2 → Review

    A3->>GIT: Push feature/t3-ui
    A3->>BB: Move T3 → Review

    H->>GIT: Review PRs, resolve conflicts, merge
    H->>BB: Move T1, T2, T3 → Merged
```

### 4.5 MCP Server — Tool Topology

```mermaid
graph TB
    subgraph MCP Server
        direction TB
        HIVE[hive/status<br/>Show active agents and tasks]
        CLAIM[hive/claim_task<br/>Claim a task from the board]
        RELEASE[hive/release_task<br/>Release a claimed task]
        DECIDE[hive/log_decision<br/>Append to decisions log]
        LOCK[hive/acquire_lock<br/>Advisory lock on files]
        UNLOCK[hive/release_lock<br/>Release file lock]
        CONTEXT[hive/get_context<br/>Project constitution + recent decisions]
        NOTIFY[hive/notify<br/>Send message to team channel]
    end

    subgraph Backing Services
        DB[(Supabase<br/>Postgres)]
        RT[Supabase<br/>Realtime]
        GH[GitHub API]
    end

    HIVE --> DB
    CLAIM --> DB
    RELEASE --> DB
    DECIDE --> DB
    LOCK --> DB
    UNLOCK --> DB
    CONTEXT --> DB
    NOTIFY --> RT

    CLAIM -.-> GH
    HIVE -.-> GH

    style MCP Server fill:#1e1b4b,color:#fff
```

### 4.6 Session Lifecycle and Heartbeat

```mermaid
sequenceDiagram
    participant Agent
    participant MCP as MCP Server
    participant DB as Supabase

    Agent->>MCP: hive/register_session
    MCP->>DB: INSERT into sessions (agent_id, machine, heartbeat)
    MCP-->>Agent: session_id, project constitution

    loop Every 30 seconds
        Agent->>MCP: hive/heartbeat
        MCP->>DB: UPDATE sessions SET heartbeat = now()
    end

    Note over DB: Background: check for stale sessions<br/>(heartbeat > 90s ago)

    alt Agent crashes
        DB->>DB: Mark session dead after 90s
        DB->>DB: Release all tasks owned by session → Open
        DB->>DB: Release all locks owned by session
    end

    Agent->>MCP: hive/deregister_session
    MCP->>DB: DELETE from sessions
```

---

## 5. Data Model

### 5.1 Entity Relationship Diagram

```mermaid
erDiagram
    PROJECTS ||--o{ TASKS : contains
    PROJECTS ||--o{ DECISIONS : contains
    PROJECTS ||--o{ SESSIONS : hosts
    PROJECTS ||--o{ LOCKS : manages
    SESSIONS ||--o{ TASKS : owns
    SESSIONS ||--o{ LOCKS : holds

    PROJECTS {
        uuid id PK
        text name
        text repo_url
        text default_branch
        jsonb constitution
        timestamptz created_at
    }

    TASKS {
        uuid id PK
        uuid project_id FK
        uuid session_id FK "nullable — null when Open"
        text title
        text description
        text status "open | claimed | in_progress | review | approved | merged | blocked"
        text branch_name "nullable"
        text[] file_scope "files this task will touch"
        int priority "1=critical, 5=nice-to-have"
        uuid blocked_by FK "nullable — references another task"
        timestamptz created_at
        timestamptz updated_at
    }

    DECISIONS {
        uuid id PK
        uuid project_id FK
        uuid session_id FK
        text category "architecture | library | convention | bugfix"
        text summary
        text rationale
        timestamptz created_at
    }

    SESSIONS {
        uuid id PK
        uuid project_id FK
        text agent_type "claude-code | cursor | copilot | custom"
        text machine_name
        text human_name
        text current_branch
        text status "active | idle | dead"
        timestamptz heartbeat
        timestamptz created_at
    }

    LOCKS {
        uuid id PK
        uuid project_id FK
        uuid session_id FK
        text file_path
        text lock_type "exclusive | shared"
        timestamptz acquired_at
        timestamptz expires_at
    }
```

### 5.2 Key Indexes

| Table | Index | Purpose |
|-------|-------|---------|
| `tasks` | `(project_id, status)` | Fast lookup of claimable tasks |
| `tasks` | `(session_id) WHERE status IN ('claimed', 'in_progress')` | Find tasks owned by a crashing session |
| `sessions` | `(project_id, status)` | List active participants |
| `sessions` | `(heartbeat) WHERE status = 'active'` | Stale session detection |
| `locks` | `(project_id, file_path)` | Conflict check before claiming |
| `decisions` | `(project_id, created_at DESC)` | Recent decisions feed |

---

## 6. Component Details

### 6.1 MCP Server

**Runtime:** Node.js on Vercel Functions (Fluid Compute)
**Protocol:** Streamable HTTP (stateless, scalable)
**Auth:** Supabase JWT — each participant authenticates via Supabase Auth; the MCP server validates the JWT on every request.

#### Tool Definitions

| Tool | Parameters | Returns | Side Effects |
|------|-----------|---------|-------------|
| `hive/status` | `project_id` | Active sessions, task summary, recent decisions | None |
| `hive/claim_task` | `project_id, task_id` | Task details + suggested branch name | Sets task.status → claimed, task.session_id |
| `hive/release_task` | `task_id` | Confirmation | Sets task.status → open, clears session_id |
| `hive/update_task` | `task_id, status, branch_name?` | Updated task | Updates task record |
| `hive/log_decision` | `project_id, category, summary, rationale` | Decision record | Appends to decisions table |
| `hive/acquire_lock` | `project_id, file_paths[], lock_type` | Lock IDs or conflict error | Inserts lock records |
| `hive/release_lock` | `lock_ids[]` | Confirmation | Deletes lock records |
| `hive/get_context` | `project_id` | Constitution + last 20 decisions + active locks | None |
| `hive/register_session` | `project_id, agent_type, machine_name, human_name` | Session ID + full project context | Inserts session |
| `hive/heartbeat` | `session_id` | Ack | Updates heartbeat timestamp |
| `hive/notify` | `project_id, message, severity` | Delivery confirmation | Broadcasts via Supabase Realtime |
| `hive/fit_check` | `project_id, description` | Matching decisions, overlapping tasks, relevant files | Searches for existing implementations; auto-runs on task creation |
| `hive/publish_contract` | `project_id, contract_name, schema` | Contract record | Publishes a typed interface for cross-department consumption |
| `hive/get_contracts` | `project_id` | All active contracts | None |

### 6.2 Supabase Configuration

**Tables:** `projects`, `tasks`, `decisions`, `sessions`, `locks` (see data model above)

**Row Level Security:**
- All tables scoped to `project_id`
- Participants can only read/write within projects they belong to
- Session writes restricted to the owning agent's auth token
- Decisions table is append-only (no UPDATE or DELETE)

**Realtime Subscriptions:**
- `tasks` — all participants subscribe to task status changes
- `decisions` — all participants subscribe to new decisions
- `sessions` — dashboard subscribes to session status changes

**Edge Function — `reap-stale-sessions`:**
- Runs on a cron (every 60 seconds)
- Finds sessions where `heartbeat < now() - interval '90 seconds'`
- Marks session as `dead`
- Releases all tasks owned by that session back to `open`
- Releases all locks held by that session

### 6.3 Repository Constitution Files

These files live in the repo root and are read by every agent at session start via `hive/get_context`.

```
repo-root/
├── CLAUDE.md          # Agent instructions (read by Claude Code natively)
├── AGENTS.md          # Role definitions, agent-specific rules
├── decisions.md       # Append-only log (also mirrored in DB)
├── .hive/
│   ├── config.yaml    # Project ID, Supabase URL, MCP endpoint
│   └── tasks.yaml     # Optional: seed tasks for new sessions
└── ...
```

**AGENTS.md** defines:
- Agent roles available (architect, implementer, tester, reviewer)
- File ownership boundaries (e.g., "Agent handling auth MUST NOT modify UI components")
- Merge rules (e.g., "All PRs require at least one human approval")
- Naming conventions for branches (`feature/t{task_id}-{slug}`)

### 6.4 Conflict Prevention Strategy

```mermaid
flowchart TD
    A[Agent claims task] --> B{Check file_scope<br/>against active locks}
    B -->|No conflicts| C[Acquire locks on file_scope]
    C --> D[Create git worktree<br/>from latest main]
    D --> E[Work in isolation]
    E --> F[Push feature branch]
    F --> G[Release locks]
    G --> H[Move task → Review]

    B -->|Conflict detected| I{Can narrow scope?}
    I -->|Yes| J[Reduce file_scope<br/>re-check locks]
    I -->|No| K[Wait or pick<br/>different task]

    J --> B
```

---

## 7. Deployment Architecture

```mermaid
graph TB
    subgraph Vercel
        MCP_FN[MCP Server<br/>Vercel Function<br/>Fluid Compute]
        DASH[Dashboard<br/>Next.js App]
    end

    subgraph Supabase
        PG[(Postgres<br/>+ RLS)]
        RT[Realtime<br/>WebSocket]
        AUTH[Auth<br/>JWT]
        EDGE[Edge Functions<br/>Cron: reap sessions]
    end

    subgraph Developer Machines
        DEV1[VS Code + Claude Code<br/>MCP Client]
        DEV2[Cursor<br/>MCP Client]
        DEV3[Terminal + Agent<br/>MCP Client]
    end

    DEV1 -->|Streamable HTTP| MCP_FN
    DEV2 -->|Streamable HTTP| MCP_FN
    DEV3 -->|Streamable HTTP| MCP_FN

    MCP_FN -->|SQL| PG
    MCP_FN -->|Broadcast| RT
    MCP_FN -->|Verify JWT| AUTH

    DASH -->|Subscribe| RT
    DASH -->|Query| PG
    DASH -->|Login| AUTH

    EDGE -->|Cron trigger| PG

    style Vercel fill:#000,color:#fff
    style Supabase fill:#1c1c1c,color:#3ecf8e
```

---

## 8. Competitive Landscape

As of April 2026, the multi-composer problem is being actively tackled by several emerging tools. AI Hive's differentiation is its **blackboard-first, infrastructure-light approach** — using Supabase + MCP as the coordination layer rather than requiring a dedicated orchestration runtime.

### 8.1 Direct Competitors — Multi-Agent Repo Coordination

| Tool | What it does | Relationship to AI Hive |
|------|-------------|------------------------|
| **Switchman** ([switchman.dev](https://switchman.dev), `npm i -g switchman-dev`) | File locking, task queues, merge confidence reviews. Works with Claude Code, Cursor, Codex, Windsurf, Aider. Ships as npm CLI + MCP server. | **Closest competitor.** Solves file-level conflict prevention. AI Hive goes further with shared decision memory, department contracts, fit analysis, and a persistent blackboard. Switchman could potentially be a building block *within* a hive. |
| **Microsoft Conductor** ([github.com/microsoft/conductor](https://github.com/microsoft/conductor)) | YAML-defined multi-agent workflows. Parallel execution, human-in-the-loop gates, evaluator-optimizer loops, real-time DAG visualization. Python CLI. MIT license. | **Complementary.** Conductor defines *how* agents execute workflows. AI Hive defines *how multiple humans coordinate* those workflows. A composer could use Conductor internally while the hive coordinates across composers. |
| **OpenClaw Command Center** ([github.com/jontsai/openclaw-command-center](https://github.com/jontsai/openclaw-command-center)) | Real-time dashboard for the OpenClaw agent framework. Session monitoring, token cost tracking ("LLM Fuel Gauges"), system health. Node.js, localhost. | **Reference architecture for Phase 3.** OpenClaw itself is a personal AI assistant framework (WhatsApp/Telegram/Slack), not a coding agent. But the Command Center's dashboard pattern — sessions, costs, health — is exactly what AI Hive's observability layer needs. |

### 8.2 Solo Composer Tools (Solved Problem)

| Tool | What it does | Gap for multi-composer |
|------|-------------|----------------------|
| **Claude Code** (subagents) | One human spawns parallel agents via `Agent` tool; worktree isolation built-in | Single session, single human, no cross-machine coordination |
| **Cursor** (background agents) | Multi-file agent mode within one IDE | Tied to one IDE instance, no shared state |
| **Jeremy King's Composer** | Custom orchestration — human defines work, agents execute autonomously | Solo workflow; no protocol for a second composer to join |
| **Aider** | Terminal-based AI pair programming with git integration | Single-agent, single-human |

### 8.3 Multi-Agent Frameworks (Different Abstraction Layer)

| Tool | What it does | Relationship to AI Hive |
|------|-------------|------------------------|
| **CrewAI** | Role-based agent collaboration | Agents-only; no model for human composers. Could run inside a department. |
| **AutoGen** | Conversational multi-agent loops | Agent-to-agent debate. Useful within a department for complex reasoning tasks. |
| **LangGraph** | Stateful agent graphs | Low-level primitive; could power agent logic within a hive participant. |
| **OpenHands** (formerly OpenDevin) | Autonomous AI software engineer | Single-agent. Could be one participant's agent of choice within the hive. |

### 8.4 Infrastructure Primitives (Building Blocks)

| Tool | Role in AI Hive |
|------|----------------|
| **MCP (Model Context Protocol)** | Agent-to-tool interop layer — AI Hive's MCP server is the shared tool surface |
| **Supabase Realtime** | Blackboard transport — push state changes to all participants |
| **Git worktrees** | Isolation primitive — prevents file collisions during parallel work |
| **Vercel Functions** | Hosting for MCP server and dashboard |

### 8.5 Where AI Hive Fits

```mermaid
quadrantChart
    title Multi-Agent Coordination Landscape
    x-axis "Single Human" --> "Multi Human"
    y-axis "Single Agent" --> "Multi Agent"

    Aider: [0.1, 0.15]
    OpenHands: [0.1, 0.3]
    Cursor Background: [0.15, 0.5]
    Claude Code Subagents: [0.2, 0.7]
    Composer: [0.2, 0.85]
    CrewAI: [0.3, 0.8]
    LangGraph: [0.4, 0.6]
    Switchman: [0.6, 0.7]
    Conductor: [0.5, 0.8]
    AI Hive: [0.85, 0.85]
```

Switchman and Conductor have moved into the center of the chart — they handle multi-agent coordination and are starting to address multi-human scenarios. AI Hive's position in the upper-right is differentiated by the **blackboard + department + contract model** — not just preventing file collisions (Switchman) or defining workflows (Conductor), but maintaining shared project intelligence across human-agent teams.

### 8.6 Build vs. Integrate Decision

| Component | Build (AI Hive) | Integrate (Existing Tool) | Recommendation |
|-----------|----------------|--------------------------|----------------|
| File locking | `locks` table + MCP tools | Switchman | **Evaluate Switchman first.** If it covers file-level locking well, use it and focus AI Hive on higher-level coordination. |
| Workflow definition | AGENTS.md + task board | Conductor | **Build our own.** Conductor's YAML is agent-workflow-level; AI Hive needs project-level task coordination. Different granularity. |
| Dashboard | Next.js + Supabase Realtime | OpenClaw Command Center | **Reference, don't fork.** OpenClaw's dashboard is tied to its agent framework. Build AI Hive's dashboard but study its UX patterns. |
| Decision memory | `decisions` table + fit_check | Nothing exists | **Build.** This is novel to AI Hive. |
| Department contracts | `contracts` table + MCP tools | Nothing exists | **Build.** This is novel to AI Hive. |

---

## 9. Phase Plan

### Phase 1 — Foundation (MVP)

**Goal:** Two humans with Claude Code agents collaborate on one repo without conflicts.

| Component | Scope |
|-----------|-------|
| Supabase schema | `projects`, `tasks`, `decisions`, `sessions`, `locks` tables with RLS |
| MCP Server | Core tools: `status`, `claim_task`, `release_task`, `log_decision`, `get_context`, `register_session`, `heartbeat` |
| Repo templates | `CLAUDE.md`, `AGENTS.md`, `decisions.md`, `.hive/config.yaml` |
| Session reaper | Supabase Edge Function cron for stale session cleanup |
| Git workflow | Manual worktree creation, branch naming convention enforced by AGENTS.md |

### Phase 2 — Automation

**Goal:** Reduce manual coordination overhead.

| Component | Scope |
|-----------|-------|
| File locking | `acquire_lock` / `release_lock` tools, conflict detection |
| Auto-worktree | MCP tool that creates worktree + branch when task is claimed |
| Notifications | Supabase Realtime broadcasts to all active agents on task changes |
| CLI companion | `hive` CLI for humans to manage tasks, view status outside of agent context |

### Phase 3 — Observability

**Goal:** Real-time visibility into hive activity.

| Component | Scope |
|-----------|-------|
| Dashboard | Next.js app showing active sessions, task board, decision log |
| Activity feed | Real-time stream of all hive events |
| Metrics | Tasks completed/hour, avg time in review, conflict rate |
| Agent health | Heartbeat visualization, crash history |

### Phase 4 — Intelligence

**Goal:** The hive gets smarter over time.

| Component | Scope |
|-----------|-------|
| Semantic memory | pgvector on decisions table for "have we solved this before?" queries |
| Auto-decomposition | Agent that breaks epics into tasks with file_scope predictions |
| Conflict prediction | Analyze file_scope overlaps before claiming to suggest sequencing |
| Cross-session context | Vector search over past sessions for onboarding new agents mid-project |

---

## 10. Security Model

| Concern | Mitigation |
|---------|-----------|
| Agent impersonation | Each session authenticated via Supabase Auth JWT; MCP server validates on every call |
| Unauthorized repo access | Git SSH keys are per-machine; MCP server never touches Git directly |
| State tampering | RLS policies scope all writes to authenticated project members |
| Decision integrity | Decisions table is append-only — no UPDATE/DELETE allowed by RLS |
| Lock starvation | Locks have `expires_at`; reaper releases expired locks automatically |
| Runaway agents | Heartbeat timeout auto-releases resources; humans can kill sessions via dashboard |

---

## 11. The Multi-Composer Problem

> "For solo dev orchestrating virtual coworkers (agents), the problem is pretty much solved. But how do you organize and manage multiple solo devs doing the same thing?"

This section addresses the core unsolved problem that AI Hive targets — informed by conversations with Jeremy King (creator of Composer, a solo-dev orchestration system).

### 10.1 Solo Composer vs. Hive: The Spectrum

```mermaid
graph LR
    subgraph Solo Composer
        H1[Human] -->|orchestrates| A1[Agent 1]
        H1 -->|orchestrates| A2[Agent 2]
        H1 -->|orchestrates| A3[Agent 3]
        H1 -->|quality gate| OUT1[Output]
    end

    subgraph Multi-Composer Hive
        HA[Human A] -->|orchestrates| AA1[Agent A1]
        HA -->|orchestrates| AA2[Agent A2]
        HB[Human B] -->|orchestrates| AB1[Agent B1]
        HB -->|orchestrates| AB2[Agent B2]
        HC[Human C] -->|orchestrates| AC1[Agent C1]
        
        AA1 & AA2 & AB1 & AB2 & AC1 -->|shared state| BB[Blackboard]
        BB -->|context| AA1 & AA2 & AB1 & AB2 & AC1
        
        HA & HB & HC -->|decisions| BB
    end

    style Solo Composer fill:#1e1b4b,color:#fff
    style Multi-Composer Hive fill:#064e3b,color:#fff
```

The solo composer pattern (Jeremy's model, Cursor background agents, Claude Code subagents) is solved because there's one brain making all decisions. The multi-composer problem introduces **distributed consensus** — multiple humans with potentially conflicting visions, each commanding their own agent swarm.

### 10.2 Three Coordination Models

```mermaid
graph TB
    subgraph Territory ["Territory Model (Recommended for MVP)"]
        direction TB
        T_HA[Human A<br/>owns: auth, users] --> T_ZONE_A[auth/<br/>users/]
        T_HB[Human B<br/>owns: API, data] --> T_ZONE_B[api/<br/>data/]
        T_HC[Human C<br/>owns: UI, pages] --> T_ZONE_C[ui/<br/>pages/]
        T_ZONE_A & T_ZONE_B & T_ZONE_C --> T_BOUNDARY[Boundary Contracts<br/>shared types, interfaces]
    end

    subgraph Hierarchy ["Hierarchy Model"]
        direction TB
        H_LEAD[Lead Human<br/>final authority] --> H_A[Human A]
        H_LEAD --> H_B[Human B]
        H_LEAD --> H_C[Human C]
    end

    subgraph Consensus ["Consensus Model"]
        direction TB
        C_HA[Human A] ---|propose| C_BOARD[Decision Board<br/>vote/object window]
        C_HB[Human B] ---|propose| C_BOARD
        C_HC[Human C] ---|propose| C_BOARD
        C_BOARD -->|silence = consent| C_ACCEPT[Accepted]
        C_BOARD -->|objection| C_DISCUSS[Discussion Thread]
    end

    style Territory fill:#064e3b,color:#fff
    style Hierarchy fill:#4a1d96,color:#fff
    style Consensus fill:#92400e,color:#fff
```

| Model | When it works | When it breaks |
|-------|--------------|----------------|
| **Territory** | Clear module boundaries, hackathon pace, 2-5 people | Shared concerns (auth touches everything), unclear ownership |
| **Hierarchy** | Established tech lead, enterprise teams, critical projects | Bottleneck on lead, lead becomes unavailable |
| **Consensus** | Research projects, design decisions, small teams with trust | Time pressure (voting is slow), deadlocks |

**AI Hive defaults to Territory with Consensus fallback.** Each human-agent pair owns a domain (enforced by file_scope and locks). Cross-domain decisions go to the blackboard with an objection window.

### 10.3 The Department Analogy

Jeremy's insight — "model orchestration like an enterprise team with departments" — maps directly to the territory model with one addition: **departments have interfaces**.

```mermaid
graph TB
    subgraph Department A ["Auth Department (Human A)"]
        DA_H[Human A] --> DA_A1[Agent: implement]
        DA_H --> DA_A2[Agent: test]
    end

    subgraph Department B ["API Department (Human B)"]
        DB_H[Human B] --> DB_A1[Agent: implement]
        DB_H --> DB_A2[Agent: test]
    end

    subgraph Department C ["UI Department (Human C)"]
        DC_H[Human C] --> DC_A1[Agent: implement]
        DC_H --> DC_A2[Agent: test]
    end

    DA_A1 -->|publishes| CONTRACT_AB[Contract: AuthService<br/>types + endpoints]
    DB_A1 -->|consumes| CONTRACT_AB
    DB_A1 -->|publishes| CONTRACT_BC[Contract: API Schema<br/>types + routes]
    DC_A1 -->|consumes| CONTRACT_BC

    CONTRACT_AB --> BB[Blackboard<br/>contract registry]
    CONTRACT_BC --> BB

    style Department A fill:#1e3a5f,color:#fff
    style Department B fill:#3b1f5e,color:#fff
    style Department C fill:#1a4731,color:#fff
```

Each department:
- **Owns** a set of files/directories (enforced by locks)
- **Publishes** contracts (TypeScript interfaces, API schemas) to the blackboard
- **Consumes** contracts from other departments
- **Reports status** upward to the shared task board
- Has **one human** as the decision-maker within that department

When a contract changes, all consuming departments are notified via Supabase Realtime. The human in each affected department decides how to adapt.

### 10.4 Fit Analysis — "Is This Already Built?"

Jeremy identified a gap in his own system: nothing prevents an agent from reimplementing something that already exists. In a multi-composer hive, this risk multiplies — Human B's agent might build a utility that Human A's agent already shipped.

The hive addresses this at two levels:

**Level 1 — Passive (Constitution)**
The `decisions.md` log and `hive/get_context` tool give every agent visibility into what's been built. When an agent starts a task, it reads the full decision history first.

**Level 2 — Active (Fit Check)**
A new MCP tool:

| Tool | Parameters | Returns | Logic |
|------|-----------|---------|-------|
| `hive/fit_check` | `project_id, description` | Matching decisions, overlapping tasks, relevant file paths | Searches decisions + active/completed tasks for semantic overlap. Returns "already implemented," "in progress by [session]," or "no match" |

At MVP, fit_check is a keyword search across decisions and task titles. At Phase 4, it becomes a semantic search via pgvector. The key insight: **fit_check runs automatically when a task is created**, not just when claimed. This prevents duplicate tasks from entering the board at all.

```mermaid
sequenceDiagram
    participant H as Human B
    participant MCP as MCP Server
    participant DB as Blackboard

    H->>MCP: Create task: "Add JWT auth middleware"
    MCP->>DB: hive/fit_check("JWT auth middleware")
    DB-->>MCP: Match found: Decision #14 "Using JWT with httpOnly cookies"<br/>Task T1 (merged): "Implement auth module"
    MCP-->>H: WARNING: Likely already implemented.<br/>See: Decision #14, Task T1.<br/>Create anyway? [y/N]
```

### 10.5 The Human Role Evolution

Jeremy's observation — "devs no longer create the actual product, they evolve agent configurations and work on the plumbing" — describes where this is heading. In the hive model, each human's role shifts:

| Traditional Role | Hive Role |
|-----------------|-----------|
| Write code | Define tasks, review agent output |
| Debug locally | Monitor agent sessions, intervene on failures |
| Attend standups | Read the blackboard, log decisions |
| Resolve merge conflicts | Define territory boundaries, maintain contracts |
| Maintain docs | Maintain constitution files (CLAUDE.md, AGENTS.md) |

The humans become **department leads** — they spend most of their time on requirements, ideation, and quality gates. The "plumbing" Jeremy mentioned is the constitution: tuning agent instructions, adjusting file scopes, refining decision-making rules.

---

## 12. Open Questions

1. **Task granularity:** Should the system enforce a maximum file_scope per task, or trust agents to self-regulate?
2. **Decision conflicts:** What happens when two agents log contradictory decisions simultaneously? First-write-wins? Human arbitration?
3. **Cross-repo hives:** Should a single hive span multiple repositories (e.g., frontend + backend), or one hive per repo?
4. **Agent capability discovery:** Should agents declare their capabilities (e.g., "I can write tests but not infrastructure"), and should the system use this for task matching?
5. **Offline mode:** How much should work when the Supabase connection is unavailable? Just Git, or a local blackboard that syncs later?
6. **Territory negotiation:** When module boundaries are unclear (e.g., "auth touches everything"), how do humans negotiate ownership? Pre-project planning session? Dynamic renegotiation via the blackboard?
7. **Contract enforcement:** Should the system enforce that departments only interact through published contracts (strict), or allow ad-hoc cross-department file edits with warnings (permissive)?
8. **Fit check threshold:** How aggressive should duplicate detection be? Too sensitive = false positives blocking legitimate work. Too loose = duplicate implementations.
9. **Scaling composers:** The current design assumes 2-5 human composers. At 10+, does territory still work, or does the system need hierarchy with territory underneath?
10. **Solo-to-hive transition:** Can a solo dev using their own Composer-like workflow join a hive mid-project without retooling? What's the minimum integration surface?

---

## 13. Glossary

| Term | Definition |
|------|-----------|
| **Blackboard** | Shared state store (Supabase Postgres) that agents read from and write to instead of communicating directly |
| **Constitution** | The combination of `CLAUDE.md`, `AGENTS.md`, and project config that defines rules all agents must follow |
| **Decision** | An append-only record of an architectural or implementation choice, with rationale |
| **Hive** | A single collaborative project instance — one repo, one Supabase project, one MCP server, N participants |
| **Lock** | Advisory file-level lock preventing two agents from modifying the same files simultaneously |
| **Participant** | A human-agent pair operating on one machine |
| **Session** | A single agent's active connection to the hive, tracked by heartbeat |
| **Worktree** | An isolated Git working directory allowing parallel branch development without checkout conflicts |
| **Composer** | A human who orchestrates one or more agents — the hive coordinates multiple composers |
| **Territory** | A bounded set of files/directories owned by one composer; enforced by locks and file_scope |
| **Department** | A composer + their agents, operating within a territory, publishing and consuming contracts |
| **Contract** | A typed interface (API schema, TypeScript types) published by one department for others to consume |
| **Fit Check** | Automated search for existing implementations before creating or claiming a task |
