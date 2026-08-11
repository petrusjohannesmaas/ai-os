# Mercy PoC — Implementation Roadmap

---

## Overview

The goal is a working demo of the full Mercy loop:

```
Mercy Shell (Tauri + React chat GUI)
    → Keycloak authentication (OAuth Authorization Code Flow)
    → Governance Server (JWT, policy, agent registry, audit)
    → Document Q&A Sub-agent (Pydantic AI)
    → Inference endpoint (hosted API provider)
    → Response back to Mercy Shell
```

All components run in local VMs — the VM boundary is the isolation model for the PoC.

---

## Phase 1 — Infrastructure Setup

### 1.1 VM Environment
- Provision three VMs: `mercy-keycloak`, `mercy-governance`, `mercy-shell`
- Each component runs in its own VM; the VM boundary is the isolation model
- Configure a shared internal network between the VMs
- Ensure VMs can resolve each other by hostname

### 1.2 Postgres
- Deploy Postgres inside the `mercy-governance` VM
- Enable the `pgvector` extension
- Create the initial database and user

### 1.3 Keycloak
- Deploy Keycloak in the `mercy-keycloak` VM
- Create a Mercy realm
- Create a test client (the governance server)
- Create at least two test users with different roles (admin, employee)

---

## Phase 2 — Governance Server

### 2.1 Project Scaffold
- Initialize a FastAPI project with SQLModel and Postgres
- Configure OpenTelemetry for logs, metrics, and traces
- Define the base database schema:
  - Users
  - Sessions
  - Roles
  - Agents
  - Audit log

### 2.2 Identity Layer
- Mercy Shell authenticates via Keycloak using the OAuth Authorization Code Flow
- Implement the token validation endpoint (receives the Keycloak token)
- Implement internal JWT issuance after successful token exchange
- Store active sessions in Postgres

### 2.3 Policy Engine
- Define a simple role-based policy model (which roles can access which agents)
- Implement a policy check middleware that runs on every request
- Deny-by-default: if no policy permits the request, it is rejected

### 2.4 Agent Registry
- Define the agent schema (name, version, description, permitted roles, enabled flag)
- Implement CRUD endpoints for registering and managing agents
- Implement the agent lookup endpoint — given a user JWT, return permitted agents

### 2.5 Audit Logging
- Implement an append-only audit log table in Postgres
- Log every inbound request: who, what agent, what action, timestamp, outcome
- No audit log entry should ever be deleted or updated — insert only

### 2.6 Inference Routing
- Implement a proxy endpoint that receives a request from Mercy Shell
- Validates JWT → checks policy → looks up agent → forwards to the inference endpoint
- Records the full exchange in the audit log
- For the PoC, the inference endpoint is a hosted API provider, wired through the
  pluggable provider interface so the swap path is exercised

### 2.7 Admin UI
- Serve a minimal admin UI from the governance server
- Used for document ingestion (see 3.1) and agent/policy management
- Admin-only, gated by role

---

## Phase 3 — Document Q&A Sub-agent

### 3.1 Document Ingestion (server-side)
- Ingestion runs entirely on the governance server, never on the client
- Expose document upload through the admin UI (Phase 2.7)
- Chunk uploaded documents and generate embeddings server-side
- Store embeddings in Postgres via pgvector

### 3.2 Retrieval
- Implement a vector similarity search against stored embeddings
- Return the top N relevant chunks for a given query

### 3.3 Agent Logic
- Implement the Document Q&A agent as a Pydantic AI agent, running as an isolated service
- Agent receives a query + retrieved chunks, calls the inference endpoint, returns a response
- Register the agent in the governance server agent registry

---

## Phase 4 — Mercy Shell

Mercy Shell is a Tauri + React desktop application.

### 4.1 Authentication Flow
- Implement the Keycloak login screen using the OAuth Authorization Code Flow
- Exchange the Keycloak token with the governance server for an internal JWT
- Store the JWT in application state (not browser storage)

### 4.2 Chat Interface
- Build the chat GUI in React
- Each message is routed to the governance server, not directly to any agent or
  inference endpoint
- Display responses returned from the governance server

### 4.3 Agent Awareness
- On login, fetch the list of permitted agents from the governance server
- Allow the user to select or invoke a sub-agent from within the chat interface

---

## Phase 5 — PoC Validation

### 5.1 Full Loop Test
- Log in as a test user via Mercy Shell
- Upload a document through the admin UI on the governance server
- Ask a question about the document via Mercy Shell
- Confirm the response is accurate and sourced from the document

### 5.2 Policy Test
- Attempt to access an agent with a role that is not permitted
- Confirm the governance server rejects the request

### 5.3 Audit Test
- After running the full loop, query the audit log
- Confirm every step — auth, agent lookup, inference call — is recorded

### 5.4 Observability Check
- Confirm OpenTelemetry is emitting traces that cover the full request path
- Confirm logs and metrics are accessible

### 5.5 Auth Flow & Provider Swap
- Confirm login completes end to end via the OAuth Authorization Code Flow
- Confirm the inference provider can be swapped via config without client changes,
  using the hosted API provider as the test target

---

## Deliverables

| Deliverable | Description |
|---|---|
| Governance Server | Running FastAPI service with all five responsibilities (identity, policy, audit, registry, routing) |
| Admin UI | Server-side UI for document ingestion and agent/policy management |
| Keycloak | Configured realm with test users and roles |
| Document Q&A Agent | Pydantic AI agent registered in the agent registry, capable of answering document questions |
| Mercy Shell | Tauri + React chat GUI that completes the full authentication and query loop |
| Audit Log | Queryable record of all PoC activity |
| Observability | Traces, logs, and metrics covering the full request path |
