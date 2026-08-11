# Mercy — Build Order & Requirements

Companion to the PoC roadmap. Defines the order components are built in, and the
requirements and definition of done for each stage.

## Why the governance server comes first

The governance server is the core of the product — everything else depends on it.
The client image is meaningless without it, the agent registry has nothing to
register against, and the delivery pipeline has nothing to deploy to.

Starting here also gives the clearest PoC story: a governance server running locally
that a single Mercy Shell client can authenticate against, route a document Q&A
request through, and produce an audit log entry. That is the proof of concept in its
simplest form.

## Stack

| Layer | Technology |
|---|---|
| Governance Server | FastAPI, SQLModel, Postgres, pgvector |
| Identity Provider | Keycloak (self-hosted) |
| Observability | OpenTelemetry |
| Agent Framework | Pydantic AI |
| Client (Mercy Shell) | Tauri + React |
| Inference (PoC) | Hosted API provider, via pluggable interface |
| Component isolation | VMs (one per component) |

## Prerequisites

- A host able to run three VMs: `mercy-keycloak`, `mercy-governance`, `mercy-shell`
- Internal network with hostname resolution between the VMs
- Postgres with the `pgvector` extension available in the governance VM
- An API key for the hosted inference provider used in the PoC

## Product build order

The full product is built in four stages. The PoC covers stages 1–2 plus a client to
drive them; stages 3–4 are post-PoC.

1. **Governance Server** — identity, policy, audit, agent registry, routing
2. **Agent Layer** — Pydantic AI sub-agents, registered and governed centrally
3. **Delivery Pipeline** — CI/CD, image versioning, distribution *(post-PoC)*
4. **Client Image** — hardened Ubuntu base running Mercy Shell *(post-PoC)*

Within the PoC, the build order maps to the roadmap phases below. Each stage lists its
dependencies, requirements, and definition of done.

## PoC build stages

### Stage 0 — Infrastructure (Roadmap Phase 1)

**Depends on:** prerequisites

**Requirements**
- Three VMs provisioned; the VM boundary is the isolation model
- Shared internal network; VMs resolve each other by hostname
- Postgres + pgvector running in the governance VM
- Keycloak in its VM: a Mercy realm, a governance-server client, and at least two
  users with different roles (admin, employee)

**Done when:** the VMs are reachable, Postgres accepts connections with pgvector
enabled, and Keycloak issues tokens for the test users.

### Stage 1 — Governance Server (Roadmap Phase 2)

**Depends on:** Stage 0

**Functional requirements**
- Validate Keycloak tokens (Authorization Code Flow), issue internal JWTs, persist
  sessions
- Role-based policy engine, deny-by-default, enforced as middleware on every request
- Agent registry: CRUD plus permitted-agent lookup by JWT
- Append-only audit log (who / agent / action / timestamp / outcome); insert-only
- Inference routing proxy: JWT → policy → agent lookup → forward to inference endpoint,
  with the full exchange audited
- Admin UI (admin-only) for document ingestion and agent/policy management

**Technical requirements**
- FastAPI + SQLModel + Postgres; OpenTelemetry traces, logs, and metrics wired in from
  the start
- Inference endpoint is the hosted API provider, behind a pluggable provider interface

**Done when:** a valid user receives a JWT and a permitted-agent list, a disallowed
request is rejected, every request lands in the audit log, and traces cover the path.

### Stage 2 — Document Q&A Agent (Roadmap Phase 3)

**Depends on:** Stage 1

**Requirements**
- Ingestion runs server-side only (never on the client), driven through the admin UI:
  upload → chunk → embed → store in pgvector
- Vector similarity search returns the top N chunks for a query
- Agent implemented in Pydantic AI as an isolated service: query + chunks → inference
  → response
- Agent registered in the registry with its permitted roles

**Done when:** an uploaded document can be queried end to end and the answer is sourced
from it.

### Stage 3 — Mercy Shell (Roadmap Phase 4)

**Depends on:** Stage 1 (Stage 2 to demo Q&A)

**Requirements**
- Tauri + React desktop application
- Keycloak login via the Authorization Code Flow; exchange for an internal JWT; JWT
  held in app state, not browser storage
- All messages routed through the governance server, never directly to an agent or
  inference endpoint
- On login, fetch permitted agents; user can select or invoke a sub-agent

**Done when:** a user logs in, selects the Document Q&A agent, asks a question, and
sees the sourced answer.

### Stage 4 — Validation (Roadmap Phase 5)

**Depends on:** Stages 0–3

**Acceptance tests**
- Full loop: login → upload via admin UI → ask → sourced answer
- Policy: a disallowed role is rejected
- Audit: every step is recorded
- Observability: traces cover the full path; logs and metrics accessible
- Auth + swap: the Authorization Code Flow completes end to end, and the inference
  provider is swappable via config without client changes

## Dependency chain

```
Prereqs → Stage 0 (Infra) → Stage 1 (Governance Server) → ┬ Stage 2 (Document Q&A Agent) ┐
                                                          └ Stage 3 (Mercy Shell)        ┴ → Stage 4 (Validation)
```

Stages 2 and 3 depend only on Stage 1 and can proceed in parallel. Stage 4 needs both.

## Post-PoC (not built in the roadmap)

- **Delivery Pipeline** — CI/CD, image versioning, HTTPS distribution
- **Client Image (Mercy OS)** — hardened, read-only Ubuntu base running Mercy Shell,
  with image-level rollback
