# Mercy OS — Application Infrastructure & Dev Workflow (v1.1)

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
* Logic is reused across MCP server and optional GUI (if it exists)
* ADK Web UI replaces Mercy Shell for the testing and alpha phase

---

## Application Structure (per tool)

```
apps/<tool-name>/
├── logic.py        # core functionality (pure, reusable)
├── mcp_server.py   # MCP wrapper (exposes functions via fastmcp)
└── gui.py          # optional
```

No `manifest.json`. No `/etc/mercy/tools/` registration.

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

### 3. Agent definition (`mercy/agent/agent.py`)

Tools are registered directly on the agent via `McpToolset`. Each tool is a subprocess connected over stdio.

```python
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

## Nix Integration

Each tool must:

### 1. Build a binary

```nix
buildPythonApplication {
  pname = "mercy-<tool>-mcp";
  src = ./apps/<tool>;
  propagatedBuildInputs = [ pkgs.python3Packages.fastmcp ];
  installPhase = ''
    mkdir -p $out/bin
    cp mcp_server.py $out/bin/mercy-<tool>-mcp
    chmod +x $out/bin/mercy-<tool>-mcp
  '';
}
```

### 2. No manifest registration required

Tool discovery is handled by ADK at agent init time, not by the filesystem.

---

## Dev Workflow

### 1. Create tool

```
mkdir -p apps/mytool
```

Add:

* `logic.py`
* `mcp_server.py`

---

### 2. Register tool in agent

Add a new `McpToolset` entry to `mercy/agent/agent.py`.

---

### 3. Add to flake

Package the binary in `flake.nix`. No manifest registration step.

---

### 4. Rebuild system

```fish
sudo nixos-rebuild switch --flake .#mercy
```

---

### 5. Run and test

```fish
cd mercy/agent
adk web --port 8000
```

Access at `http://localhost:8000`. Select `mercy_agent` and interact.

---

## Mental Model

* You are building **capabilities**, not apps
* Each tool: small, composable, stateless
* ADK owns the orchestration layer — do not reimplement it

---

## Anti-patterns

Avoid:

* Writing a custom router or tool dispatcher
* Filesystem-based tool discovery (manifests, `/etc/mercy/tools`)
* Embedding logic in the MCP layer
* Long-running MCP server processes
* Async agent creation patterns (breaks deployment)

---

## Definition of Done (for a tool)

A tool is complete when:

* `logic.py` contains pure functions
* `mcp_server.py` exposes them via `fastmcp`
* The binary is registered in `flake.nix`
* A `McpToolset` entry for it exists in `agent.py`
* ADK selects and executes it correctly via `adk web`
* No changes were required outside the tool folder + flake + agent.py

---

## Stack Summary

| Concern | Solution |
|---|---|
| LLM orchestration | Google ADK (`LlmAgent`) |
| Tool protocol | MCP via `fastmcp` (stdio) |
| Tool-agent bridge | `McpToolset` + `StdioConnectionParams` |
| UI (testing/alpha) | ADK Web UI (`adk web`) |
| Packaging | Nix (`buildPythonApplication`) |
| Language | Python 3.10+ |
