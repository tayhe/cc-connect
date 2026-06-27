# MiMoCode Agent Adapter

cc-connect 原生支持 MiMoCode（mimo CLI）。本文档记录添加过程、设计决策和后续维护要点。

## 背景

MiMoCode 是 opencode 的 fork，CLI 接口几乎完全一致。cc-connect 已有 opencode adapter，需要添加 mimocode 作为独立 agent 类型。

### 为什么选择 cc-connect 而非 hapi

| 维度 | cc-connect | hapi |
|------|-----------|------|
| Agent 注册机制 | Go `init()` + `RegisterAgent()` 自注册，零核心代码修改 | 硬编码 `AGENT_FLAVORS` 数组 + 15+ 文件散布在 4 个 package |
| 添加新 agent 的改动量 | 复制一个目录 + 1 个 5 行的 wiring 文件 | 修改 15+ 文件，新建 10-15 个文件 |
| 上游 merge 冲突风险 | 低（改动隔离在独立目录） | 高（改动分散在 shared/cli/web/hub） |
| 架构成熟度 | 15 个 agent 均用同一模式，插件架构成熟 | 有 AgentRegistry 但实际未使用 |

**结论：** cc-connect 的插件架构更适合添加新 agent，改动可控，上游升级友好。

## 设计决策：复制 vs 共享基底

### 方案 A：直接复制 opencode adapter（采用）

复制 `agent/opencode/` → `agent/mimocode/`，全局替换包名、二进制名、路径。

**优点：**
- 实施简单，照搬上游改名即可
- mimocode 和 opencode 各自独立，互不影响
- 上游改动时 diff 1:1 映射，无需理解抽象层

**缺点：**
- ~1300 行代码重复
- 上游修复 bug 需手动同步

### 方案 B：抽取共享基底（未采用）

将通用逻辑提取到 `agent/opencodebase/`，opencode 和 mimocode 各写薄配置层。

**为什么不采用：**
- 上游没有这个抽象层，维护时需要将上游改动"翻译"到你的架构里
- 上游重构时你的抽象边界可能失效，需要再次重构
- "自动受益"的优势是假的——你仍然需要在共享基底里改，而且要同时考虑两个消费者

**核心原则：** 如果上游没有某个抽象层，你自己创建它不会降低维护成本，反而增加认知负担。

## 实际改动清单

### 新建文件

| 文件 | 说明 |
|------|------|
| `agent/mimocode/mimocode.go` | Agent 实现（从 opencode.go 复制） |
| `agent/mimocode/session.go` | Session 实现（从 session.go 复制） |
| `agent/mimocode/mimocode_model_test.go` | 模型相关测试 |
| `agent/mimocode/mimocode_workdir_race_test.go` | 并发安全测试 |
| `agent/mimocode/session_test.go` | Session 测试 |
| `agent/mimocode/provider_resume_test.go` | Provider 恢复测试 |
| `cmd/cc-connect/plugin_agent_mimocode.go` | 插件注册（5 行） |

### 修改文件

| 文件 | 改动 |
|------|------|
| `tests/blackbox/helper/agents.go` | 添加 mimocode import |
| `tests/blackbox/helper/env.go` | 添加 mimocode 测试用例（3 处 switch case） |
| `tests/blackbox/p1/agents.go` | 添加 mimocode import |
| `tests/blackbox/p2/agents.go` | 添加 mimocode import |
| `tests/integration/agent_integration_test.go` | 添加 mimocode import + 测试用例 |
| `tests/e2e/smoke_test.go` | 添加 mimocode 到 agent 列表 |
| `provider-presets.json` | 12 个 provider 添加 mimocode 配置 |

### 关键替换对照表

| 原文 (opencode) | 替换为 (mimocode) |
|-----------------|-------------------|
| `package opencode` | `package mimocode` |
| `RegisterAgent("opencode", New)` | `RegisterAgent("mimocode", New)` |
| `ParseCmdOpts(opts, "opencode")` | `ParseCmdOpts(opts, "mimo")` |
| `Name() string { return "opencode" }` | `Name() string { return "mimocode" }` |
| `opencodeDBPath()` | `mimocodeDBPath()` |
| `~/.local/share/opencode/opencode.db` | `~/.local/share/mimocode/mimocode.db` |
| `OPENCODE.md` | `MIMOCODE.md` |
| `~/.opencode/OPENCODE.md` | `~/.mimocode/MIMOCODE.md` |
| `opencodeSession` | `mimocodeSession` |
| `newOpencodeSession` | `newMimocodeSession` |
| `.opencode-models.json` | `.mimocode-models.json` |
| 日志前缀 `opencode:` | `mimocode:` |
| 日志前缀 `opencodeSession:` | `mimocodeSession:` |

## 配置使用

在 `config.toml` 中：

```toml
[[projects]]
name = "my-project"
work_dir = "/path/to/project"

[projects.agent]
type = "mimocode"
```

可选配置：

```toml
[projects.agent.options]
cmd = "mimo"          # 二进制名，默认 "mimo"
model = "anthropic/claude-sonnet-4-20250514"
mode = "default"      # "default" 或 "yolo"
```

## 后续维护：如何同步上游 opencode adapter 的改动

当上游 cc-connect 更新了 `agent/opencode/` 的代码，按以下步骤同步到 mimocode：

### 1. 查看上游 diff

```bash
cd ~/Projects/cc-connect
git log --oneline agent/opencode/ | head -20
git diff <old-commit> HEAD -- agent/opencode/
```

### 2. 将改动应用到 mimocode

对于每个改动的文件：
- 如果是 `opencode.go` 的改动 → 应用到 `mimocode.go`
- 如果是 `session.go` 的改动 → 应用到 `session.go`
- 如果是测试文件的改动 → 应用到对应的 `*_test.go`

### 3. 需要注意的替换规则

应用改动时，将以下内容替换为 mimocode 版本：
- 字符串 `"opencode"` → `"mimocode"`（但注意 `"opencode-ai/opencode"` 这类 URL 不要改）
- 函数名/类型名中的 `opencode` → `mimocode`
- 日志前缀 `"opencode:"` → `"mimocode:"`

### 4. 验证

```bash
go build ./agent/mimocode/
go vet ./agent/mimocode/
go test ./agent/mimocode/ -v -count=1
```

### 5. 新增文件

如果上游在 `agent/opencode/` 新增了文件，复制到 `agent/mimocode/` 并做同样的替换。

## MiMoCode 与 OpenCode CLI 接口差异

两者接口几乎完全一致，以下是已知差异：

| 功能 | OpenCode | MiMoCode |
|------|----------|----------|
| 二进制名 | `opencode` | `mimo` |
| DB 路径 | `~/.local/share/opencode/opencode.db` | `~/.local/share/mimocode/mimocode.db` |
| Memory 文件 | `OPENCODE.md` | `MIMOCODE.md` |
| 全局配置目录 | `~/.opencode/` | `~/.mimocode/` |
| ACP 子命令 | `opencode acp` | `mimo acp` |
| Plugin 系统 | 无 | `mimo plugin` |

如果 MiMoCode 将来修改了 CLI 接口（如 NDJSON event 格式、run 参数等），需要在 mimocode adapter 中单独处理，不再与 opencode 保持一致。
