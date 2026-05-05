# CI/CD Infrastructure Reference


## Version Control System Strategy

**Platform:** GitHub

**Branch structure:**

```
main → always buildable, CI gates merges via PR
feature/<package> → short-lived, human-authored commits
```

**Rules:**
- `main` is protected — no direct commits
- All changes enter via PR from a `feature/*` branch
- PRs require CI to pass before merge
- Feature branches are deleted after merge
- `release/x.y` branches are reserved for V1+ when backporting becomes necessary

**Git worktrees** are the recommended pattern for parallel agent-assisted development. Each agent gets an isolated worktree on its own feature branch, sharing the same `.git` object store.

```fish
git worktree add ../mercy-os-agent-1 feature/<package-a>
git worktree add ../mercy-os-agent-2 feature/<package-b>
```

---

## CI/CD Server/Platform Strategy

**Platform:** GitHub Actions with a self-hosted runner

The self-hosted runner is an LXC container on the development machine. GitHub Actions orchestrates the workflow. Compute stays local.

**Stack:**

| Concern | Solution |
|---|---|
| VCS & PR workflow | GitHub |
| CI orchestration | GitHub Actions |
| CI compute | Self-hosted Actions runner (LXC) |
| Test framework | pytest |

**Trigger:** CI runs on every PR targeting `main`. Merge is blocked until CI passes.

**Workflow — `.github/workflows/ci.yml`:**

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: pip install fastmcp pytest

      - name: Run tests
        run: pytest apps/${{ github.head_ref }} --tb=short
```

**Test scope per PR:**

- Unit tests — `logic.py` functions in isolation
- MCP contract tests — tool exposes expected functions with correct signatures over stdio
- ADK tool selection tests — agent routes prompts to the correct tool
- End-to-end tests — full path from prompt through ADK, MCP subprocess, logic, and back

---

## Caching Strategy

**Stack:**

```
CI Runner → Caching Proxy (Go) → nix-serve → Nix store
```

**nix-serve** caches built Nix derivations on the LXC. Instead of rebuilding derivations on every CI run, the runner fetches pre-built results from the local cache.

```fish
nix-env -i nix-serve
nix-serve -p 5000
```

Point the runner at it in `flake.nix`:

```nix
nix.settings.substituters = [ "http://localhost:5000" ];
```

**Caching Proxy** sits in front of nix-serve and caches HTTP responses to reduce repeated hits. Requires refactoring for persistent disk storage before production use — current implementation is in-memory only and does not survive restarts.

**pip dependencies** are cached between runs via GitHub Actions cache:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: pip-${{ hashFiles('**/requirements.txt') }}
```

---

## Build Artifact Repo Strategy



---

## Test Environments Strategy
