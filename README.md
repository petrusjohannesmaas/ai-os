# Mercy

Mercy is governed AI infrastructure for organizations. It lets employees use AI
assistants over company data while giving the organization full control over who
can access what — and a complete record of everything the AI is asked to do.

The operating system Mercy ships with is not the product. The product is the
**governance server** that sits between every employee and every AI agent, plus
the **agents** that run on top of it.

## The problem

Companies want staff to use AI on internal data — contracts, policies, support
tickets, docs. But handing employees a public chatbot means:

- Sensitive data leaves the building with no controls
- No record of what was asked or answered
- No way to say "support can use the ticket agent, but not the finance agent"
- No control over which model or provider processes the data

Mercy exists so a company can offer AI assistants and still answer, at any time:
who used it, what they asked, what data it touched, and under what policy.

## What it looks like in use

A single request, end to end:

1. An employee opens **Mercy Shell**, the chat app on their Mercy OS laptop.
2. They log in through **Keycloak** — the same SSO they use for everything else.
3. Mercy Shell asks the **governance server** which agents this person may use.
   It gets back a list — say, just the Document Q&A agent.
4. The employee types: *"Summarize the changes to our returns policy this quarter."*
5. Mercy Shell sends that to the governance server. It never talks to an AI directly.
6. The server checks policy (is this person allowed to use Document Q&A?), logs the
   request, then routes it to the **Document Q&A agent**.
7. The agent pulls the relevant document chunks from Postgres via pgvector and calls
   the configured model.
8. The answer flows back through the governance server — logged again — to the employee.

Later, IT or compliance can open the audit log and see exactly who asked what, when,
and which agent and data were involved.

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

Everything the employee does passes through the governance server. Nothing on the
client talks to an AI, a data source, or another agent on its own.

---

## Components

### 1. Mercy OS — the client image

The locked-down laptop OS employees actually use. Its job is to be tamper-proof and
identical on every machine, not to be a general-purpose computer.

- Built from a minimal Ubuntu base via a versioned image pipeline
- Root filesystem mounted read-only at boot
- Runs Mercy Shell as the primary interface; no local AI logic
- Updates ship as new images; rollback is switching to the previous image version
- Isolation enforced via systemd hardening or Docker, governed by server-side policy

### 2. Mercy Shell — the chat app

The only way in for the employee. A Tauri + React desktop application presenting a
chat interface.

- Authenticates via Keycloak using the OAuth Authorization Code Flow
- Never communicates directly with sub-agents or inference endpoints — every message
  is routed through the governance server
- Filesystem access is allowlisted at the Tauri level; default access is nothing
- On login, fetches the list of permitted sub-agents from the governance server

### 3. Keycloak — login

Off-the-shelf, self-hosted identity provider so Mercy never manages passwords itself.

- Owns user credentials and authentication entirely
- Issues tokens to Mercy Shell via the Authorization Code Flow, which the governance
  server exchanges for internal JWTs
- Supports enterprise identity federation (LDAP, Active Directory) when required

### 4. Governance server — the product

The single checkpoint every request passes through.

- **Identity & authentication** — validates Keycloak tokens, issues internal JWTs,
  manages sessions
- **Policy enforcement** — defines which agents and data sources each role can reach;
  deny-by-default
- **Audit logging** — append-only record of every action taken through the system
- **Agent registry** — authoritative list of approved sub-agents, their versions,
  permitted roles, and enabled state
- **Inference routing** — proxies requests to the right sub-agent and inference
  endpoint based on policy

### 5. Agent layer — the assistants

The AI assistants themselves: Pydantic AI sub-agents, each running as an isolated
service and registered centrally.

- Developed, versioned, and registered through a defined CI/CD pipeline
- No sub-agent is reachable by a client until it is registered and a policy permits it
- Inference providers are pluggable — cloud or on-prem, swappable without client changes
- The Document Q&A agent is the first sub-agent: retrieves document chunks from Postgres
  via pgvector and calls the inference endpoint

---

## Key design decisions

| Concern | Approach |
|---|---|
| Client immutability | Read-only Ubuntu base image |
| Rollback | Image-level versioning via CI/CD pipeline |
| Application isolation | Tauri allowlist (client), VM isolation (server-side) |
| Reproducibility | CI/CD pipeline, not the package manager |
| Authentication | Keycloak, OAuth Authorization Code Flow |
| Authorization | Governance server policy engine, deny-by-default |
| Audit trail | Governance server, append-only Postgres table |
| Inference flexibility | Pluggable provider, routed via governance server |
| RAG capability | Postgres + pgvector, embedded at the data layer |
| Observability | OpenTelemetry across the full request path |

---

## Delivery pipeline

- **Develop** — sub-agents and governance components built and tested in isolation
- **Ship** — new client images and server updates built by CI, versioned, distributed
  over HTTPS
- **Maintain** — the governance server handles policy updates without client image
  changes; client image updates are atomic and rollback-capable
