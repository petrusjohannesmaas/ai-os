# Mercy OS — Application Infrastructure & Dev Workflow (v1.2)

---

## Core Infrastructure

**Execution pipeline**

```
ADK Web UI (dev/testing)
        |
ADK Runner (LlmAgent + Gemini)
        |
McpToolset (StdioConnectionParams)
        |
MCP Tool Binary (subprocess, stdio)
        |
Logic (pure functions)
```

**Key properties**

* ADK handles the event loop, LLM calls, tool selection, and tool execution — no custom router
* MCP servers are isolated subprocesses spawned on demand via stdio
* Tool registration is code-level — tools are declared in `agent.py`, not via filesystem manifests
* Every component — tools and agent — is a first-class Nix package
* ADK Web UI replaces Mercy Shell for the testing and alpha phase

---

## Project Structure

```
mercy-os/
├── flake.nix
├── configuration.nix
├── apps/
│   └── <tool-name>/
│       ├── default.nix     # Nix derivation
│       ├── logic.py        # core functionality (pure, reusable)
│       └── mcp_server.py   # MCP wrapper (exposes functions via fastmcp)
└── mercy/
    └── agent/
        ├── default.nix     # Nix derivation
        ├── agent.py
        └── __init__.py
```

No `manifest.json`. No `/etc/mercy/tools/` registration. No `.env` file.

---

## Contracts (must be respected)

### 1. Logic layer

* No I/O, no UI, no side effects
* Deterministic functions only

---

### 2. MCP server

* Uses `fastmcp` to expose functions as MCP tools
* Communicates over stdio (spawned as subprocess by ADK)
* Accepts structured input, returns structured output

```python
from fastmcp import FastMCP
import logic

mcp = FastMCP("<tool-name>")

@mcp.tool()
def my_function(param: str) -> str:
    return logic.my_function(param)

if __name__ == "__main__":
    mcp.run()
```

---

### 3. Tool derivation (`apps/<tool-name>/default.nix`)

Each tool is its own Nix package. `flake.nix` composes them via `callPackage`.

```nix
{ python3Packages }:

python3Packages.buildPythonApplication {
  pname = "mercy-<tool>-mcp";
  version = "0.1";
  src = ./.;
  propagatedBuildInputs = [ python3Packages.fastmcp ];
  installPhase = ''
    mkdir -p $out/bin
    cp mcp_server.py $out/bin/mercy-<tool>-mcp
    chmod +x $out/bin/mercy-<tool>-mcp
  '';
}
```

---

### 4. Agent definition (`mercy/agent/agent.py`)

API key is read from the environment at runtime — not stored in the Nix store.

```python
import os
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

root_agent = LlmAgent(
    model="gemini-flash-latest",
    name="mercy_agent",
    instruction="Route user requests to the correct tool.",
    tools=[
        McpToolset(
            connection_params=StdioConnectionParams(
                server_params=StdioServerParameters(
                    command="mercy-<tool>-mcp",
                    args=[],
                )
            )
        ),
        # add more McpToolset entries per tool
    ],
)
```

---

### 5. Agent derivation (`mercy/agent/default.nix`)

```nix
{ python3Packages }:

python3Packages.buildPythonApplication {
  pname = "mercy-agent";
  version = "0.1";
  src = ./.;
  propagatedBuildInputs = [ python3Packages.google-adk ];
  installPhase = ''
    mkdir -p $out/bin
    cp agent.py $out/bin/mercy-agent
    chmod +x $out/bin/mercy-agent
  '';
}
```

---

### 6. Flake composition (`flake.nix`)

`flake.nix` composes packages via `callPackage`. Each app's derivation is self-contained.

```nix
packages.${system} = {
  mercy-calculator-mcp = pkgs.callPackage ./apps/calculator {};
  mercy-agent          = pkgs.callPackage ./mercy/agent {};
};
```

---

### 7. System packages (`configuration.nix`)

All Mercy packages ship with the OS.

```nix
environment.systemPackages = [
  self.packages.${system}.mercy-calculator-mcp
  self.packages.${system}.mercy-agent
];
```

---

## Dev Workflow

### 1. Create tool

```
mkdir -p apps/<tool-name>
```

Add:

* `default.nix`
* `logic.py`
* `mcp_server.py`

---

### 2. Register tool in agent

Add a new `McpToolset` entry to `mercy/agent/agent.py`.

---

### 3. Expose in flake

Add a `callPackage` entry to `flake.nix` and include the package in `configuration.nix`.

---

### 4. Verify the build in isolation

Before rebuilding the system, confirm the derivation builds cleanly:

```fish
nix build .#mercy-<tool>-mcp
```

---

### 5. Rebuild system

```fish
sudo nixos-rebuild switch --flake .#mercy
```

---

### 6. Run and test

Set your API key in the current shell session, then launch the agent:

```fish
set -x GOOGLE_API_KEY "your-key-here"
adk web --port 8000
```

Access at `http://localhost:8000`. Select `mercy_agent` and interact.

---

## Mental Model

* You are building **capabilities**, not apps
* Each tool: small, composable, stateless
* Every component is a Nix package — tools, agent, and OS are one reproducible unit
* ADK owns the orchestration layer — do not reimplement it

---

## Anti-patterns

Avoid:

* Writing a custom router or tool dispatcher
* Filesystem-based tool discovery (manifests, `/etc/mercy/tools`)
* Embedding logic in the MCP layer
* Long-running MCP server processes
* Async agent creation patterns (breaks deployment)
* Inline derivations in `flake.nix` — each package belongs in its own `default.nix`
* Storing secrets in the Nix store

---

## Definition of Done (for a tool)

A tool is complete when:

* `logic.py` contains pure functions
* `mcp_server.py` exposes them via `fastmcp`
* `default.nix` defines the derivation
* A `callPackage` entry exists in `flake.nix`
* The package is listed in `configuration.nix`
* A `McpToolset` entry for it exists in `agent.py`
* `nix build .#mercy-<tool>-mcp` succeeds
* ADK selects and executes it correctly via `adk web`

---

## Stack Summary

| Concern | Solution |
|---|---|
| LLM orchestration | Google ADK (`LlmAgent`) |
| Tool protocol | MCP via `fastmcp` (stdio) |
| Tool-agent bridge | `McpToolset` + `StdioConnectionParams` |
| UI (testing/alpha) | ADK Web UI (`adk web`) |
| Packaging | Nix (`buildPythonApplication` + `callPackage`) |
| Secrets | Runtime environment variable (`GOOGLE_API_KEY`) |
| Language | Python 3.10+ |