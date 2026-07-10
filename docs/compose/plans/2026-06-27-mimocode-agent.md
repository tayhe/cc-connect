# Mimocode Agent Adapter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use compose:subagent (recommended) or compose:execute to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `mimocode` as a new agent type to cc-connect, mirroring the existing `opencode` adapter with mimocode-specific binary name, paths, and identifiers.

**Architecture:** Direct copy of `agent/opencode/` → `agent/mimocode/` with targeted replacements. Zero changes to core engine code. The plugin registration pattern (`init()` + `RegisterAgent`) handles wiring automatically.

**Tech Stack:** Go, same as existing opencode adapter.

## Global Constraints

- Binary name: `mimo` (not `mimocode`)
- Agent registration name: `mimocode`
- DB path: `~/.local/share/mimocode/mimocode.db` (or `$XDG_DATA_HOME/mimocode/mimocode.db`)
- Memory files: `MIMOCODE.md` (project) and `~/.mimocode/MIMOCODE.md` (global)
- Log prefix: `mimocode:` / `mimocodeSession:`
- All string replacements are case-sensitive exact matches

---

### Task 1: Create mimocode agent package

**Covers:** Core adapter implementation

**Files:**
- Create: `agent/mimocode/mimocode.go` (copy from `agent/opencode/opencode.go`)
- Create: `agent/mimocode/session.go` (copy from `agent/opencode/session.go`)
- Create: `agent/mimocode/session_test.go` (copy from `agent/opencode/session_test.go`)
- Create: `agent/mimocode/opencode_model_test.go` → `agent/mimocode/mimocode_model_test.go`
- Create: `agent/mimocode/opencode_workdir_race_test.go` → `agent/mimocode/mimocode_workdir_race_test.go`
- Create: `agent/mimocode/provider_resume_test.go` (copy from `agent/opencode/provider_resume_test.go`)

- [ ] **Step 1: Copy opencode directory**

```bash
cp -r agent/opencode agent/mimocode
```

- [ ] **Step 2: Rename test files**

```bash
cd agent/mimocode
mv opencode_model_test.go mimocode_model_test.go
mv opencode_workdir_race_test.go mimocode_workdir_race_test.go
```

- [ ] **Step 3: Apply replacements to mimocode.go**

Replace in `agent/mimocode/mimocode.go`:
- Line 1: `package opencode` → `package mimocode`
- Line 23: `core.RegisterAgent("opencode", New)` → `core.RegisterAgent("mimocode", New)`
- Line 72: `core.ParseCmdOpts(opts, "opencode")` → `core.ParseCmdOpts(opts, "mimo")`
- Line 83: error message `"opencode: %q CLI not found..."` → `"mimocode: %q CLI not found..."`
- Line 197: `func (a *Agent) Name() string { return "opencode" }` → `func (a *Agent) Name() string { return "mimocode" }`
- All `slog.Info("opencode:` / `slog.Debug("opencode:` / `slog.Warn("opencode:` → `slog.Info("mimocode:` etc.
- `opencodeProjectModelCachePath` → `mimocodeProjectModelCachePath`
- `opencodePersistentModelCache` → `mimocodePersistentModelCache`
- `opencodeModelDiscoverySnapshot` → `mimocodeModelDiscoverySnapshot`
- `loadOpencodePersistentModelCache` → `loadMimocodePersistentModelCache`
- `opencodeSessionEntry` → `mimocodeSessionEntry`
- `listOpencodeSessions` → `listMimocodeSessions`
- `opencodeDBPath` → `mimocodeDBPath`
- DB path: `"opencode", "opencode.db"` → `"mimocode", "mimocode.db"`
- `OPENCODE.md` → `MIMOCODE.md`
- `".opencode", "OPENCODE.md"` → `".mimocode", "MIMOCODE.md"`

- [ ] **Step 4: Apply replacements to session.go**

Replace in `agent/mimocode/session.go`:
- Line 1: `package opencode` → `package mimocode`
- `opencodeSession` → `mimocodeSession`
- `newOpencodeSession` → `newMimocodeSession`
- `opencodeImageExt` → `mimocodeImageExt`
- All `slog.Debug("opencodeSession:` / `slog.Info("opencodeSession:` / `slog.Error("opencodeSession:` → `mimocodeSession:`

- [ ] **Step 5: Apply replacements to test files**

In all `*_test.go` files under `agent/mimocode/`:
- `package opencode` → `package mimocode`
- All references to `opencode` types/functions → `mimocode` equivalents

- [ ] **Step 6: Verify compilation**

```bash
go build ./agent/mimocode/
```

Expected: no errors

---

### Task 2: Create plugin wiring file

**Covers:** Agent registration

**Files:**
- Create: `cmd/cc-connect/plugin_agent_mimocode.go`

- [ ] **Step 1: Create plugin file**

```go
//go:build !no_mimocode

package main

import _ "github.com/chenhg5/cc-connect/agent/mimocode"
```

- [ ] **Step 2: Verify compilation**

```bash
go build ./cmd/cc-connect/
```

Expected: `mimocode` appears in registered agents

---

### Task 3: Update test infrastructure

**Covers:** Test support for new agent

**Files:**
- Modify: `tests/blackbox/helper/agents.go`
- Modify: `tests/blackbox/helper/env.go`
- Modify: `tests/blackbox/p1/agents.go`
- Modify: `tests/blackbox/p2/agents.go`
- Modify: `tests/integration/agent_integration_test.go`
- Modify: `tests/e2e/smoke_test.go`

- [ ] **Step 1: Add mimocode import to blackbox helper agents.go**

Add to import block:
```go
_ "github.com/chenhg5/cc-connect/agent/mimocode"
```

- [ ] **Step 2: Add mimocode cases to env.go**

In `requireAPIKey` function (~line 296), add after opencode case:
```go
case "mimocode":
    if os.Getenv("ANTHROPIC_API_KEY") == "" && !hasProviderEnv("mimocode") {
        t.Skipf("blackbox skip: ANTHROPIC_API_KEY not set for mimocode")
    }
```

In `applyProviderFromEnv` function (~line 333), add after opencode case:
```go
case "mimocode":
    apiKey = os.Getenv("ANTHROPIC_API_KEY")
    baseURL = os.Getenv("ANTHROPIC_BASE_URL")
```

In `agentBinName` function (~line 426), add after opencode case:
```go
case "mimocode":
    return "mimo"
```

- [ ] **Step 3: Add mimocode import to p1/agents.go and p2/agents.go**

Add to import block in both files:
```go
_ "github.com/chenhg5/cc-connect/agent/mimocode"
```

- [ ] **Step 4: Update integration test**

In `agent_integration_test.go`:
- Add import: `_ "github.com/chenhg5/cc-connect/agent/mimocode"` (use a different blank import name since `opencode` is already taken, e.g. `mimocode "github.com/chenhg5/cc-connect/agent/mimocode"`)
- Add `var _ = mimocode.New` (with the named import)
- In `skipUnlessAgentReady`, add case after opencode:
```go
case "mimocode":
    if os.Getenv("OPENAI_API_KEY") == "" && os.Getenv("ANTHROPIC_API_KEY") == "" {
        t.Skipf("skip %s: OPENAI_API_KEY or ANTHROPIC_API_KEY not set", agentType)
    }
```
- In `findAgentBin`, add case:
```go
case "mimocode":
    return "mimo", nil
```
- In the agents list (~line 656), add `"mimocode"` to the slice

- [ ] **Step 5: Update e2e smoke test**

In `smoke_test.go` `listRegisteredAgents()`, add `"mimocode"` to the agents slice.

- [ ] **Step 6: Verify all tests compile**

```bash
go build ./tests/...
```

Expected: no errors

---

### Task 4: Update provider-presets.json

**Covers:** Provider configuration for mimocode

**Files:**
- Modify: `provider-presets.json`

- [ ] **Step 1: Add mimocode entries**

For each provider entry in `provider-presets.json` that has an `"opencode"` key in its `"agents"` object, add a `"mimocode"` entry with the same value.

```bash
python3 -c "
import json
with open('provider-presets.json') as f:
    data = json.load(f)
for provider in data:
    if isinstance(provider, dict) and 'agents' in provider:
        if 'opencode' in provider['agents'] and 'mimocode' not in provider['agents']:
            provider['agents']['mimocode'] = provider['agents']['opencode']
with open('provider-presets.json', 'w') as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
    f.write('\n')
"
```

- [ ] **Step 2: Verify JSON validity**

```bash
python3 -c "import json; json.load(open('provider-presets.json')); print('OK')"
```

Expected: `OK`

---

### Task 5: Final verification

**Covers:** End-to-end build and test verification

- [ ] **Step 1: Full build**

```bash
go build ./...
```

Expected: no errors

- [ ] **Step 2: Run unit tests**

```bash
go test ./agent/mimocode/ -v -count=1
```

Expected: all tests pass (or skip if mimocode binary not available)

- [ ] **Step 3: Verify agent registration**

```bash
go run ./cmd/cc-connect/ --help 2>&1 | grep -i mimocode || echo "check registry manually"
```

- [ ] **Step 4: Commit**

```bash
git add agent/mimocode/ cmd/cc-connect/plugin_agent_mimocode.go tests/ provider-presets.json
git commit -m "feat: add mimocode agent adapter

Mirror opencode adapter for mimocode (mimo CLI binary).
Same NDJSON interface, different binary name and paths."
```
