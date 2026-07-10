# Local Modifications

本文件记录对 cc-connect 上游仓库的本地修改，方便追踪和同步。

## MiMoCode Agent Adapter

**日期：** 2026-06-27
**上游版本：** 当前 main 分支
**文档：** [docs/mimocode-agent.md](./docs/mimocode-agent.md)

### 改动概述

添加 `mimocode` 作为独立 agent 类型，使用 `mimo` CLI 二进制。基于 opencode adapter 复制修改。

### 新增文件

- `agent/mimocode/` — Agent 实现（从 `agent/opencode/` 复制）
- `cmd/cc-connect/plugin_agent_mimocode.go` — 插件注册

### 修改文件

- `tests/blackbox/helper/agents.go` — 添加 mimocode import
- `tests/blackbox/helper/env.go` — 添加 mimocode 测试用例
- `tests/blackbox/p1/agents.go` — 添加 mimocode import
- `tests/blackbox/p2/agents.go` — 添加 mimocode import
- `tests/integration/agent_integration_test.go` — 添加 mimocode 测试用例
- `tests/e2e/smoke_test.go` — 添加 mimocode 到 agent 列表
- `provider-presets.json` — 12 个 provider 添加 mimocode 配置

### 上游同步

当上游更新 `agent/opencode/` 时，需要将改动同步到 `agent/mimocode/`。详见 [docs/mimocode-agent.md](./docs/mimocode-agent.md#后续维护如何同步上游-opencode-adapter-的改动)。

## 上游同步记录

### 2026-07-10 同步至 upstream/main

- **commit:** `c45f35de` (merge of `a23fb904` upstream + 本地 mimocode adapter)
- **上游变更数:** 26 个 commit
- **冲突:** 仅 `Makefile`（`ALL_AGENTS` add/add + `ALL_PLATFORMS` 加 `cloud_web`），union 处理后按字母排序
- **`agent/opencode/` 未变动** — mimocode adapter 无需照搬更新
- **`core/interfaces.go` 仅新增可选接口** — Go 接口兼容性 OK，无 mimocode 实现改动
- **合并 commit:** `c45f35de Merge remote-tracking branch 'upstream/main'`

### Makefile 长期维护提示

`ALL_AGENTS` 和 `ALL_PLATFORMS` 是 fork 冲突高发区（任何 fork 用户加自己的 agent/peer 都会撞）。未来同步：
1. 冲突时优先 union 而不是 take one side
2. 按字母排序，便于对比
3. 永远保留本地 `mimocode` 项 + 上游所有项
