# Getting Started

The governance server is the core of the product — everything else depends on it. The client image is meaningless without it, the agent registry has nothing to register against without it, and the CI/CD pipeline has nothing to deploy to without it.

Starting there also gives you the clearest PoC story: a governance server running locally that a single Mercy Shell client can authenticate against, route a document Q&A request through, and produce an audit log entry. That's the proof of concept in its simplest form.

## Stack

| Layer | Technology |
|---|---|
| Governance Server | FastAPI, SQLModel, Postgres, pgvector |
| Identity Provider | Keycloak (self-hosted, containerized) |
| Observability | OpenTelemetry |
| Agent Framework | Pydantic AI |
| Client (Mercy Shell) | Tauri + React |
| Containerization | LXC |

## SOP Order

1. **Governance Server** — identity, policy engine, audit log, agent registry
2. **Agent Layer** — Pydantic AI sub-agents, registered and governed centrally
3. **Delivery Pipeline** — CI/CD, image versioning, distribution
4. **Client Image** — hardened Ubuntu base running Mercy Shell
