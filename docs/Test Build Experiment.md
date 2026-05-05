# MERCY OS — End-to-End Experiment v1.2 (ADK + MCP Native)

> Open ADK Web UI → type prompt → ADK + Gemini decides → calls MCP tool via stdio → returns result
> Built with Nix, pure Python, using ADK's native MCP integration.

---

## 1. Project Structure

```
mercy-os/
├── flake.nix
├── configuration.nix
├── apps/
│   └── calculator/
│       ├── default.nix
│       ├── logic.py
│       └── mcp_server.py
└── mercy/
    └── agent/
        ├── default.nix
        ├── agent.py
        └── __init__.py
```

---

## 2. Calculator Logic

### `apps/calculator/logic.py`

```python
def add(a: int, b: int) -> int:
    return a + b

def subtract(a: int, b: int) -> int:
    return a - b
```

---

## 3. MCP Server

### `apps/calculator/mcp_server.py`

```python
from fastmcp import FastMCP
import logic

mcp = FastMCP("calculator")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return logic.add(a, b)

@mcp.tool()
def subtract(a: int, b: int) -> int:
    """Subtract b from a."""
    return logic.subtract(a, b)

if __name__ == "__main__":
    mcp.run()
```

---

## 4. Calculator Derivation

### `apps/calculator/default.nix`

```nix
{ python3Packages }:

python3Packages.buildPythonApplication {
  pname = "mercy-calculator-mcp";
  version = "0.1";
  src = ./.;
  propagatedBuildInputs = [ python3Packages.fastmcp ];
  installPhase = ''
    mkdir -p $out/bin
    cp mcp_server.py $out/bin/mercy-calculator-mcp
    chmod +x $out/bin/mercy-calculator-mcp
  '';
}
```

---

## 5. ADK Agent

### `mercy/agent/agent.py`

```python
import os
from google.adk.agents import LlmAgent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

root_agent = LlmAgent(
    model="gemini-flash-latest",
    name="mercy_agent",
    instruction="You are a system agent. Route user requests to the correct tool and return the result.",
    tools=[
        McpToolset(
            connection_params=StdioConnectionParams(
                server_params=StdioServerParameters(
                    command="mercy-calculator-mcp",
                    args=[],
                )
            )
        ),
    ],
)
```

### `mercy/agent/__init__.py`

```python
from . import agent
```

### `mercy/agent/default.nix`

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

## 6. flake.nix

```nix
{
  description = "Mercy OS Experiment v1.2";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-26.05";

  outputs = { self, nixpkgs }:
  let
    system = "x86_64-linux";
    pkgs = import nixpkgs { inherit system; };
  in {
    packages.${system} = {
      mercy-calculator-mcp = pkgs.callPackage ./apps/calculator {};
      mercy-agent          = pkgs.callPackage ./mercy/agent {};
    };

    nixosConfigurations.mercy = nixpkgs.lib.nixosSystem {
      inherit system;
      modules = [
        ./configuration.nix
        {
          environment.systemPackages = [
            self.packages.${system}.mercy-calculator-mcp
            self.packages.${system}.mercy-agent
          ];
        }
      ];
    };
  };
}
```

---

## 7. configuration.nix

```nix
{ config, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  services.xserver.enable = false;

  nix.settings.experimental-features = [ "nix-command" "flakes" ];
}
```

---

## How to Run

```fish
# Verify each package builds in isolation first
nix build .#mercy-calculator-mcp
nix build .#mercy-agent

# Build and switch
sudo nixos-rebuild switch --flake .#mercy

# Set API key and launch agent
set -x GOOGLE_API_KEY "your-key-here"
adk web --port 8000
```

Access at `http://localhost:8000`.

---

## End-to-End Flow

1. You type: `add 3 and 5`
2. ADK Runner receives the message and passes it to the `LlmAgent`
3. Gemini selects the `add` tool from the registered `McpToolset`
4. ADK spawns `mercy-calculator-mcp` as a subprocess over stdio
5. `mcp_server.py` calls `logic.add(3, 5)`
6. Result `8` is returned to ADK, relayed to the UI

---

## Adding Another Tool (e.g. Greeter)

### Files to create

```
apps/greeter/
├── default.nix
├── logic.py
└── mcp_server.py
```

### `apps/greeter/logic.py`

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"
```

### `apps/greeter/mcp_server.py`

```python
from fastmcp import FastMCP
import logic

mcp = FastMCP("greeter")

@mcp.tool()
def greet(name: str) -> str:
    """Greet a user by name."""
    return logic.greet(name)

if __name__ == "__main__":
    mcp.run()
```

### `apps/greeter/default.nix`

```nix
{ python3Packages }:

python3Packages.buildPythonApplication {
  pname = "mercy-greeter-mcp";
  version = "0.1";
  src = ./.;
  propagatedBuildInputs = [ python3Packages.fastmcp ];
  installPhase = ''
    mkdir -p $out/bin
    cp mcp_server.py $out/bin/mercy-greeter-mcp
    chmod +x $out/bin/mercy-greeter-mcp
  '';
}
```

### Add to `flake.nix`

```nix
mercy-greeter-mcp = pkgs.callPackage ./apps/greeter {};
```

And add to `environment.systemPackages`:

```nix
self.packages.${system}.mercy-greeter-mcp
```

### Add to `mercy/agent/agent.py`

```python
McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="mercy-greeter-mcp",
            args=[],
        )
    )
),
```

### Verify and rebuild

```fish
nix build .#mercy-greeter-mcp
sudo nixos-rebuild switch --flake .#mercy
```

---

## Key Point

> Adding a tool = **3 files + 1 callPackage entry + 1 systemPackages entry + 1 McpToolset entry**

No router changes. No manifest files. No filesystem registration.

---

## Known Limitations (acceptable for test)

* No streaming
* No retries
* No persistent memory (future roadmap item)
* ADK Web UI is dev-only, not for production
* API key managed via runtime environment variable

---

## Assessment

### Correct

* ADK owns orchestration — no custom routing logic
* MCP protocol used correctly over stdio
* Tools and agent are isolated Nix packages, composable and independently buildable
* Architecture is not a dead end — memory, multi-agent, and TUI layers can be added incrementally
* CI can build and validate each package independently via `nix build .#<package>`

### Removed complexity vs v1.1

* Inline derivations in `flake.nix` → each app has its own `default.nix`
* `.env` file in Nix store → runtime environment variable
* `nixos-24.05` → `nixos-26.05`