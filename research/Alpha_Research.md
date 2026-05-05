# MERCY OS — Alpha Research

Research findings and architectural decisions informing the Alpha build, with notes on what is deferred to Beta and V1.

---

## Alpha Scope

### Agentic Dispatch

Already validated in the test build. ADK owns the orchestration layer. MCP tools are isolated subprocesses registered code-level on the agent. No custom router. This carries forward unchanged into Alpha.

### Memory Layer

Alpha introduces a two-tier persistent memory system. Both tiers are local and single-user.

```
System prompt construction
        |
        ├── Tier 1: SQLite
        │   └── injected as frozen block at session start
        │
        └── Tier 2: Qdrant + FastEmbed
            └── queried semantically per turn → top-k chunks injected
                        ↑
                   FastEmbed (local, CPU)
```

**Tier 1 — Active Memory (SQLite)**

A small, bounded store of key facts injected into every system prompt. Two logical stores:

* `agent_memory` — environment facts, conventions, learned behaviours, completed task diary
* `user_profile` — preferences, communication style, working patterns, corrections

Character limits are enforced at write time to keep system prompts bounded. The agent writes to both stores autonomously during sessions. Changes persist to disk immediately but the injected system prompt block is a frozen snapshot captured at session start — it does not update mid-session.

**Tier 2 — Session History (Qdrant + FastEmbed)**

All session turns are stored as vector points in Qdrant. FastEmbed runs locally on CPU to produce embeddings — no API calls. Storage is turn-by-turn (not summarised) for retrieval granularity and implementation simplicity.

Each point payload carries: `role`, `timestamp`, `session_id`, `text`.

On each turn, the current input is embedded and used to query Qdrant for semantically relevant past turns. The top-k results are injected into context alongside the Tier 1 block.

**Write path:** turn → FastEmbed → vector → Qdrant point. Memory entries additionally → SQLite.

**Read path:** input → FastEmbed → Qdrant query → top-k chunks + SQLite block → system prompt.

### Stack Additions for Alpha

| Concern | Solution |
|---|---|
| Active memory store | SQLite |
| Vector store | Qdrant (local service) |
| Embedding model | FastEmbed (local, CPU) |

---

## Beta Scope

### Durable Execution Runtime

The agent loop currently runs entirely in-process. All execution state is held in memory for the duration of a run. For Alpha's complexity level this is acceptable. For Beta — with longer-running tasks, multi-step pipelines, and HITL workflows — it is not.

**Agentspan** is the planned solution for Beta. It is a durable execution runtime, not an agent framework. It wraps the existing ADK pipeline without changing agent definitions, tools, or routing logic.

What it adds:

* **Crash recovery** — execution state is held server-side. A worker restart picks up from the last completed step.
* **Durable HITL** — approval pauses are stored server-side and wait indefinitely, with no in-memory state at risk.
* **Execution history** — every run stored with inputs, outputs, token usage, and per-step timing.
* **Long-running task support** — tasks detached from the worker process lifecycle.

Agentspan will be self-hosted. Its viability as a system service is to be evaluated based on resource footprint at integration time.

Integration is a one-line wrap of the existing agent:

```python
from agentspan.agents import AgentRuntime

with AgentRuntime() as runtime:
    result = runtime.run(root_agent, prompt)
```

---

## V1 Scope

### In-House Durable Runtime

Agentspan is a validated reference implementation, not a permanent dependency. V1 replaces it with a purpose-built runtime designed for Mercy OS specifically.

Core components to build:

* **Execution state store** — persistent store of run state at every step boundary (SQLite baseline)
* **Orchestrator** — reads state, determines next step, dispatches to worker; survives worker restarts
* **Worker protocol** — stateless between steps; registers tools, executes on instruction, returns result
* **HITL gate** — pause/resume mechanism backed by the state store, exposed via CLI and API
* **Run history** — queryable log of all runs built as a read layer over the state store
* **Runtime API** — the interface agent code uses to start, step, pause, and resume runs

Known hard problems: exactly-once execution guarantees, step boundary definition within the ADK loop, worker reconnection without duplication.
