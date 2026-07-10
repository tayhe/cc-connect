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

---

## 上游同步流程手册（标准 SOP）

这个 SOP 是 2026-07-10 完成首次实战同步后总结的经验。每当 upstream/main 有新提交，**按以下顺序**执行。

### 0. 前提确认（一次性，不必每次做）

- 工作目录：`cd ~/Projects/cc-connect`
- Remotes：`origin` = `tayhe/cc-connect`（你的 fork），`upstream` = `chenhg5/cc-connect`
- Go：`go version` 输出 `go1.22.x linux/arm64`（已知可用）
- npm 包路径：`~/.nvm/versions/node/v26.3.0/lib/node_modules/cc-connect/`
- systemd service：`systemctl --user status cc-connect`

**关键事实：npm 包装器 `run.js` 已经禁用自动重装版本检查**（第 60 行 `if (false && needsReinstall())`）。这意味着 `cp` 替换 binary 后重启不会被 npm 偷偷替换回上游版本。如果你未来跑 `npm install -g cc-connect@beta`，**这个 hack 会被覆盖**，需要重新改一遍。

### 1. 准备：清理工作树

```bash
cd ~/Projects/cc-connect
git status                          # 必须先看清楚有什么未提交
```

如果有未提交内容（M / A? / ??），先决定去留：
- **是 mimocode 相关改动**（Makefile、docs/compose、agent/mimocode/*）→ commit，单独写 commit message
- **是临时调试** → `git stash` / trash 掉

不要直接 `git stash` mimocode 相关，会丢上下文。

```bash
git add <明确要保留的文件>
git commit -m "本地 ... 简述"
```

### 2. 评估上游变更

```bash
git fetch upstream
```

#### 2a. 看上游到底动了什么数量和位置

```bash
echo "上游多出 N 个提交："
git rev-list --count main..upstream/main

echo "上游目录分布："
git diff --name-only main..upstream/main | awk -F/ '{print $1"/"$2}' | sort | uniq -c | sort -rn | head
```

#### 2b. **关键预判：上游有没有动 mimicode adapter 相关的核心文件**

```bash
# 决定你后续要不要"照搬"模式同步 agent/mimocode/
git diff --stat main..upstream/main -- agent/opencode/ agent/mimocode/ core/interfaces.go core/agent_session_*.go core/engine.go
```

三种结果：

| 输出 | 结论 | 行动 |
|------|------|------|
| 全为空 | 上游没改这些 | agent/mimocode/ 零改动 |
| 只显示 ADD 行 | 上游纯增加（接口扩展、新功能） | 优先做"兼容检查"：go build 一下看是否仍编译 |
| 显示 M（modify）| 上游改了行为 | 必须 diff 同步到 agent/mimocode/ 对应文件（替换包名 + 标识符） |

#### 2c. 看哪些文件会冲突（cheap 预判）

```bash
# 这是反向 diff —— 列出"本地有、上游没有"的文件，意味着这些将被视为新增（ADD），不是冲突
git diff -G 'mimocode' --name-status main..upstream/main
# 输出格式: D = delete from local perspective（其实是上游没有，本地独有）, M = 上游改了
```

只要所有 D 是 mimocode 相关的文件（你加的 + LOCAL-MODS.md），M 是你之前也动过的少数文件（Makefile、provider-presets.json、tests 下的 helper），冲突就有限。

### 3. 执行合并

```bash
git merge upstream/main --no-ff --no-commit
```

`--no-commit` 是有意的——让你**先看清状态再写 commit message**。

如果 git 输出 "Automatic merge failed"，看冲突文件列表：
- 进入 §4
- 如果只有 1-2 处冲突，shell 仍保持在 merge 状态

如果输出 "Already up to date"，上游你没拉到新东西，可以退出。

### 4. 处理冲突

#### 4a. 必查：`git status | grep -iE "unmerged|conflict"`

唯一一次教训：`Makefile` 的 `ALL_AGENTS` 和 `ALL_PLATFORMS` 是 **fork 冲突高发区**（任何 fork 用户加自己的 agent/platform 都会撞），处理方式：

```bash
# 1. 看冲突 hunk
git diff Makefile

# 2. 解决：union + 字母排序
#    <<<<<<< HEAD
#    ALL_AGENTS := ... mimocode opencode ...
#    =======
#    ALL_AGENTS := ... opencode ...       ← 上游没有 mimocode
#    >>>>>>> upstream/main
#
#    ALL_PLATFORMS := ... webex
#    =======
#    ALL_PLATFORMS := ... webex cloud_web  ← 上游加了新 platform
#    >>>>>>> upstream/main
#
# 最终：
#    ALL_AGENTS    := acp antigravity claudecode codex copilot cursor devin gemini iflow kimi mimocode opencode pi qoder reasonix tmux
#    ALL_PLATFORMS := cloud_web discord dingtalk feishu line matrix max qq qqbot slack telegram wecom weibo weixin webex
```

永远不要 `git checkout --theirs Makefile` —— 那会**抹掉你的 mimocode 注册**。

#### 4b. 其他文件冲突

对照 §2c 列出的 M 文件：

| 文件 | 冲突类型 | 解决 |
|------|---------|------|
| `provider-presets.json` | 两侧都加了不同 provider 块 | 保留全部。git 一般能自动 merge；如果不行，按 JSON 字段 union |
| `tests/blackbox/helper/agents.go`、`env.go`、`p1/agents.go`、`p2/agents.go` | 两侧都加了 agent 导入/用例 | 保留全部 import 和测试用例 |
| `tests/integration/agent_integration_test.go` | 同上 | 保留全部 |
| `tests/e2e/smoke_test.go` | 同上（agent 列表） | 保留全部 |

所有这些"列表/import 用例" 文件的冲突，本质都是 **add/add**—— git 经常能自动处理，处理不了就手动 union。

#### 4c. 解决后

```bash
git add <改好的文件>
git status | grep -iE "unmerged|conflict" || echo "冲突已解决"
```

### 5. 验证合并结果

```bash
# 5a. mimocode adapter 关键文件还在吗
git ls-files agent/mimocode/                      # 应该看到 6 个 .go 文件
git ls-files cmd/cc-connect/plugin_agent_mimocode.go
git ls-files LOCAL-MODS.md docs/mimocode-agent.md

# 5b. provider-presets.json 关键 agent 块完整
grep -c '"mimocode"' provider-presets.json
grep -c '"opencode"' provider-presets.json
```

### 6. 完成 merge commit

```bash
git commit --no-edit    # 用默认 merge message
# 或者写更具体的：
# git commit -m "merge upstream/main: 同步 N 个提交，仅 Makefile 冲突做 union 处理"
```

### 7. 编译

```bash
make build-noweb       # 默认 web=false，包含 ALL_AGENTS 和 ALL_PLATFORMS 全量
# 或 make build       # 包含 web UI
```

Go 编译时间 ~3-5 分钟。如果失败：
- 上游改了 `agent/opencode/` 接口 → 同步到 `agent/mimocode/`（按文件 diff + 替换包名）
- 上游 Go module 升级 → `go mod tidy`

### 8. 验证二进制

```bash
ls -la cc-connect                                       # 应该 20+ MB
./cc-connect --version
# 注意 commit: 字段应该 == merge commit 哈希
```

### 9. 部署

```bash
systemctl --user stop cc-connect
cp cc-connect /home/tayhe/.nvm/versions/node/v26.3.0/lib/node_modules/cc-connect/bin/cc-connect
systemctl --user start cc-connect
sleep 2
systemctl --user status cc-connect --no-pager | head
journalctl --user -u cc-connect --since "1 minute ago" --no-pager | grep -iE "error|panic|fatal|level=ERROR" | head -5
```

最后一条 grep **应该没有输出**。如果输出非空：
- `permission denied` → 可能是 binary 文件权限（`chmod +x`）
- `permission requested` + 飞书长连接成功 → 正常业务日志，不是 ERROR
- `panic: ... ` → 编译/部署没成功，回滚 binary

### 10. 记录这次同步

回到 `LOCAL-MODS.md` 的 "上游同步记录" 章节，按时间倒序追加一条：

```markdown
### YYYY-MM-DD 同步至 upstream/main

- **commit:** `XXXXXXX` (merge commit 哈希)
- **上游变更数:** N 个 commit
- **冲突:** <文件>（<原因>），<处理方式>
- **`<关键文件>` 未变动 / 变动情况** — 结论
- **`core/interfaces.go` 变动情况** — 是否影响 mimocode
```

### 11. 提交 LOCAL-MODS 更新

```bash
git add LOCAL-MODS.md
git commit -m "本地改动清单：记录 YYYY-MM-DD upstream 同步"
```

### 12. 推不推 fork？

`origin` (tayhe/cc-connect) **不必主动 push**——本地工作树和 fork 远程保持一致是好的，但本 SOP 默认只更新本地 + 部署。如果你要跨机器访问这次构建的代码，再 push。

---

## 流程核心原则（务必记住）

1. **永远是 merge，不是 rebase**：`git merge upstream/main --no-ff --no-commit`。Rebase 会重新打散 mimocode 提交的"原子性"，且 undo 复杂。
2. **永远不要 `git checkout --theirs/--ours` Makefile**：单边选择会丢 mimocode 或丢上游新 platform。Makefile 必须 union。
3. **先评估再 merge**：§2 的预判在 80% 的场景里能告诉你"这次同步会不会破坏 mimocode"。跳过这一步 = 把 build 完才发现要回滚。
4. **commit 永远干净**：merge 前先把所有未提交改动 commit 掉，不要堆在一个 merge commit 里——以后单独 revert 起来容易。
5. **LOCAL-MODS.md 是真相**：所有非标准同步决策（手动 union、绕过 npm 重装等）都要写在这里。

## 常见陷阱

- **run.js 被 npm 更新覆盖**：第 60 行 `if (false && ...)` 会被 `npm install -g cc-connect@beta` 重置。检查 `head -65 run.js` 看 `if (false` 是否还在；不在则需重新改。
- **Makefile 字母顺序**：未来加新 agent 时务必保持字母排序。冲突解决时也按字母排序。两者顺序不一致 = 下次同步 100% 撞。
- **mimocode agent 实现 vs opencode 模板不同步**：如果上游在某次同步后改了 `agent/opencode/session.go` 的内部逻辑，而你**没跟着改 `agent/mimocode/session.go`**，运行时 mimocode agent 会出现行为偏差（不一定报错，但表现不一致）。发现方法：`diff agent/opencode/session.go agent/mimocode/session.go`，除包名 + 标识符外应该完全相同。
- **config.toml 不会被 sync 影响**：cc-connect 配置目录是 `~/.cc-connect/config.toml`，在 fork 仓库之外。同步不会自动影响部署。
