# Mercy Infrastructure Overview

## What We Are Building

Mercy is a governed AI infrastructure product. The OS is not the product — the governance layer and AI tooling are. Mercy OS refers to a hardened Ubuntu base image deployed to client endpoints. The core IP lives in the governance server and the agent layer that runs on top of it.

---

## Architecture

```
Mercy Shell (Tauri + React chat GUI)
    ↓ OAuth Authorization Code Flow
Keycloak (Identity Provider)
    ↓ Token exchange → internal JWT
Governance Server
    ├── Identity & authentication
    ├── Policy enforcement
    ├── Audit logging
    ├── Agent registry
    └── Inference routing (cloud or on-prem)
        └── Pydantic AI Sub-agents
            └── Company data sources (Postgres + pgvector)
```

---

## Components

### 1. Client Image (Mercy OS)

A hardened, minimal Ubuntu base image deployed to employee endpoints. Its purpose is tamper-proofing and consistency, not general-purpose computing.

- Built from a minimal Ubuntu base via a versioned image pipeline
- Root filesystem mounted read-only at boot
- Runs Mercy Shell — the primary agent interface for the user
- No local AI logic — all requests route through the governance server
- Updates delivered as new images; rollback is switching to the previous image version
- Application isolation enforced via systemd service hardening or Docker, governed by server-side policy
- Reproducibility is guaranteed by the CI/CD pipeline and image versioning

### 2. Mercy Shell

The primary agent on the Mercy OS client. A Tauri + React desktop application presenting a chat interface to the user.

- Authenticates via Keycloak using the OAuth Authorization Code Flow
- Never communicates directly with sub-agents or inference endpoints
- All messages are routed through the governance server
- Filesystem access is explicitly allowlisted at the Tauri level — default access is nothing
- On login, fetches the list of permitted sub-agents from the governance server

### 3. Keycloak

Self-hosted identity provider running in a dedicated container.

- Owns user credentials and authentication entirely
- Issues tokens to Mercy Shell via the Authorization Code Flow
- The governance server exchanges Keycloak tokens for internal JWTs
- Supports enterprise identity federation (LDAP, Active Directory) when required

### 4. Governance Server

The core of the Mercy product. All client interactions are mediated here.

- **Identity & authentication** — validates Keycloak tokens, issues internal JWTs, manages sessions
- **Policy enforcement** — defines what sub-agents and data sources each role can access; deny-by-default
- **Audit logging** — append-only record of every action taken through the system
- **Agent registry** — authoritative list of approved sub-agents, their versions, permitted roles, and enabled state
- **Inference routing** — proxies requests to the appropriate sub-agent and inference endpoint based on policy

### 5. Agent Layer

Pydantic AI sub-agents, each running as an isolated service and registered centrally in the governance server.

- Sub-agents are developed, versioned, and registered through a defined CI/CD pipeline
- No sub-agent is accessible to a client until it is registered and a policy permits it
- Inference providers are pluggable — cloud or on-prem, swappable without client changes
- The Document Q&A agent is the first sub-agent: retrieves relevant document chunks from Postgres via pgvector and calls the inference endpoint

---

## Key Design Decisions

| Concern | Approach |
|---|---|
| Client immutability | Read-only Ubuntu base image |
| Rollback | Image-level versioning via CI/CD pipeline |
| Application isolation | Tauri allowlist (client), systemd hardening / Docker (server-side) |
| Reproducibility | CI/CD pipeline, not the package manager |
| Authentication | Keycloak, OAuth Authorization Code Flow |
| Authorization | Governance server policy engine, deny-by-default |
| Audit trail | Governance server, append-only Postgres table |
| Inference flexibility | Pluggable provider, routed via governance server |
| RAG capability | Postgres + pgvector, embedded at the data layer |
| Observability | OpenTelemetry across the full request path |

---

## What Was Replaced

| Previous Approach | Current Approach |
|---|---|
| NixOS base | Hardened Ubuntu base image |
| Nix flakes + derivations | Debian packages, Docker images, virtualenvs |
| `nixos-rebuild switch` | Image pipeline + standard deployment tooling |
| Nix store reproducibility | CI/CD pipeline reproducibility |
| Package-level rollback | Image-level rollback |
| Tool registry | Agent registry (sub-agents with identities, versions, and permitted roles) |

---

## Delivery Pipeline (Develop → Ship → Maintain)

- **Develop** — sub-agents and governance server components built and tested in isolated environments
- **Ship** — new client images and server updates built by CI, versioned, distributed via HTTPS
- **Maintain** — governance server handles policy updates without requiring client image changes; client image updates are atomic and rollback-capable
