# humanize 深入学习文档

> 研读对象：`jasonchen505/humanize`（fork 自 `humanfia/humanize`），研读时间 2026-10-07。
> 注意：这不是 Python 那个数字格式化库（`python-humanize`），而是 humanfia 的 **agent flow 系统**，
> slogan 是 "The agent flow system for token maxxing"。这是一份个人学习笔记，放在 fork 的
> `notes/learning` 分支上，不进入 upstream PR。

---

## 1. 一句话定位

humanize 定义并执行 **coding agent 的工作流（flow）**：你用 Python 写一个 flow（规定"谁被问什么、按什么顺序、何时停止"），
`hmz` 负责把它跑起来、把执行过程完整记录下来、事后可回放。它驱动的是你本机已登录的 coding agent CLI
（Claude Code、Codex、Kimi……），本身不实现模型调用。

## 2. 核心概念

### 2.1 flow vs run vs epic vs trace

- **flow**：一个 Python 函数，声明它需要的 agents（roles）、envs、params，规定循环逻辑。
- **run**：flow 的一次执行。run 在执行中被完整写下来，叫 **epic**（flow + agents + 每个 session）。
- **trace**：把整个 run 画成一条时间线（可在 Perfetto 里打开），用来找"哪一 turn 真正推动了任务"。

### 2.2 Session：flow 能做的最大的选择

Session 是与一个 agent 的"一次对话历史"（turn 的序列）。文档原话大意：**何时开一个新 session，是 flow 能做的最大的选择，因为它决定了 agent 记住什么**。
例如 `ralph_loop` 每轮开新 session，agent 每轮都从 task 重新开始，天然防"路径依赖"。

### 2.3 spawn / fork：把"对话历史"变成一等公民

- `Agent.spawn()`：开一个新 session（此时还不启动 CLI，第一个 turn 才启动）；
- `Agent.fork(session)`：把一个已有 session 在"它自己的第一个 turn 处"切出一个 fork——新 session 带着切点之前的对话继续。

这解决的问题：flow 能把一次 run 中途达到的对话状态快照下来，在本 run 或之后 `--resume` 的 run 里从断点继续。
这是 tree-search、多臂回放类 flow 的地基（upstream issue #172 的动机正是这个：`ctx.state` 里存 session，读出来是带着当时对话的新 session）。

关键解耦：**session 只带对话历史，turn 在哪台机器跑是 `run(env=...)` 的事**——"对话"和"执行环境"被拆开了，
大多数 agent 框架把这两者绑死。

### 2.4 FlowState 与 Outworlder

- **FlowState**（`ctx.state`）：dict-like，只有 `resumable=True` 的 flow 才有；`--resume` 会捡起本 workspace 该 flow 最新可恢复的 run。
- **Outworlder**：flow 眼中的"你"——prompt 前面的人。`/afk` 表示离开；从命令行 `hmz exec` 启动时 outworlder 恒为 away，默认空回答。

### 2.5 Mixin 能力声明（取代 harness 分支）

flow 不写 `if harness == "claude"`，而是声明能力：

```python
class MyAgent(Agent, GoalCommandAgentMixin, SteeringAgentMixin): ...
```

- 只有声明了 `GoalCommandAgentMixin` 才有 `/goal` 命令，"只声明普通 agent，就调不出 `/goal`，哪怕底层 harness 支持"；
- env 侧同理：`GitEnvMixin` / `RewindableEnvMixin` 把 snapshot/rewind 变成可声明、可拒绝（`CapabilityMissing`）的能力。

新增一个 harness 时，所有 flow 零改动即获得"能声明"的基础——一种"能力即类型"的设计。

## 3. 架构：`src/hmz` 七个包

职责划分来自 `specs/SPEC.md` 的 Packages 表（"answerable for"）：

| 包 | 职责 |
|---|---|
| `coganchor` | 关于"驱动 coding agent CLI"的一切知识（各 CLI 是什么、怎么驱动、以哪个账号跑、turn 落在哪台机器、token 价格）+ anchor（agent 在一台机器跑、在另一台机器干活）。子包：`agents`（驱动/会话/事件流）、`machines`、`providers`（账号）、`serve`、`linux` |
| `flows` | flow 能 import 的全部、**仅此而已**——纯 protocol 类型（`Agent`/`Env`/`Session`/`FlowContext`/异常体系），runtime 在运行时传入结构性实现。builtin flows 必须"像外部 flow 一样"只 against `flows` 写 |
| `runtime` | run 是什么：找 flow、给每个 role 发 driver、边跑边写 epic、事后读回。自己不驱动任何 agent |
| `cli` | 命令行。给人用的只有 `hmz exec` 一个命令；`hmz internal *`（anchor/cred/fence/hook/tools）是 humanize 为自己 spawn 的内部命令 |
| `daemon` | 一台机器上每个 workspace 的 runs 的宿主——终端关闭也不死；tui/sdk 都是它的前端 |
| `tui` | `hmz` 无命令时打开的终端界面（Textual）；不定义 flow、不驱动 agent、自己不存东西 |
| `sdk` | 外部工具进入 humanize 的路：`Hmz`（本进程的 runtime）、`Daemons`（机器上被 hold 的 runs）。"只透传、不组合" |

### 3.1 specs/：契约不是写在 wiki 里的

顶层 `SPEC.md` 规定整棵树的契约：每个包对什么负责、层之间怎么 import、顶层 API 长什么样；
每个包在自己路径下有专属 spec（`coganchor/SPEC.md`、`runtime/SPEC.md`、`cli.md`、`daemon.md`、`tui.md`、`sdk.md`、`flows.md`，
另有 `coganchor/agents.md`、`machines.md` 等子 spec），没被文件点名的包服从最近的上级。语言是 RFC 2119 式的 MUST/MUST NOT。

两个关键 MUST 例子（`specs/SPEC.md`）：
1. "`coganchor` MUST be the whole of what humanize knows about driving a coding agent CLI… No layer above it MUST reach past it to a driver."——所有"怎么驱动 CLI"的知识只能住在 coganchor，上层不许绕过它直连 driver。
2. 层间命名规则："Each layer MUST import only its own subtree… No two layers MUST name each other"（唯一例外：`flows` 可把 `flow`/`load`/`Outworlder.new` 递给 `runtime/flowing`，且只能在调用内部 import），并由 `tests/integration/layering/test_layering.py` 把层表写进测试**强制执行**。

CONTRIBUTING 原话："`specs/` is the contract. Change the code to match a spec."——先有契约，代码向契约看齐。

## 4. 内置 flows（`src/hmz/flows/builtin/`）

| flow | 行为 |
|---|---|
| `chat` | 单 agent 单 session 对话；TUI 默认打开的 flow；唯一没有自己 budget 的 flow（runtime 按 `Budget(cost=inf)` 跑）；session 不保留 |
| `ralph_loop` | 每轮开**全新** session（agent 记不住上一轮）；连续 3 轮空回答或 budget 耗尽则停；`--resume` 接着计数 |
| `stateful_ralph` | 同一个 session 跑全程，每轮重发 task；停止条件同上；resume 时开新 session 继续计数 |
| `goal` | 单 turn `/goal <task>`，model 自己干到说完成或 budget 耗尽；要求 harness 原生支持 `/goal` |
| `continue_loop` | 单 session，先发 task，之后每轮只发 "continue" 催；失败/空回答则重发；连续 3 次失败结束 |
| `flame_chase` | 两个 agent 轮流上阵，每轮都是 fresh session（两人只共享 repo）；某轮失败就换另一个 chaser；连续 3 次失败结束 |
| `rlar` | actor 在一个 session 里干活，每轮开一个全新的 reviewer session 审它的工作；reviewer 说 done 或 budget 耗尽结束 |

共同模式：连续 3 次失败熔断、budget 耗尽抛 `BudgetExceeded` 结束、`--resume` 恢复。

## 5. Backends：驱动哪些 CLI

`-a` 用的名字即 `HarnessKind`（`src/hmz/flows/agents.py`）：`claude`、`codex`、`cursor-agent`、`opencode`、`mimo`、
`mcode`、`qwen`、`kimi`、`grok`、`pi`、`agy`、`dsh`（DeepSeek Harness）、`litellm`（直连模型：每 turn 一次
chat completion，无工具/无文件系统/无 env）、`acp`（走 Agent Client Protocol 的 CLI）。

组织方式：
- `coganchor/backends.py` 的 `PROFILES` 表是 "every fact about a coding agent CLI that is not code"——每个 CLI 的调用方式（argv 模板）、登录方式、支持的 models 与 efforts；
- `coganchor/providers/` 是账号体系（named provider，登录/API key/gateway，与 CLI 自带登录隔离）；
- `coganchor/agents/` 下每个 harness 一个 driver，把各 CLI 的事件流统一成 turn/session 模型。

## 6. 安装与使用

```sh
uv tool install 'hmz>=0.1.0b1'            # 基础（claude/codex 等 CLI 型后端自带）
uv tool install 'hmz[dsh]>=0.1.0b1'       # DeepSeek Harness（走其 Python SDK）
uv tool install 'hmz[kimi]>=0.1.0b1'      # Kimi
uv tool install 'hmz[litellm]>=0.1.0b1'   # 直连模型
uv tool install 'hmz[all]>=0.1.0b1'       # 全要
```

需要 Python 3.12+ 和一个已登录的 coding agent CLI。**注意：agent 全部以 bypass approvals 运行，官方建议先在 scratch repo 里跑。**

```sh
hmz                                  # 不带命令 → 打开 TUI，prompt 即 chat
hmz exec -f rlar -a actor=claude/claude-opus-5:high -a reviewer=codex/gpt-5:high \
  -p budget.duration=2h,budget.cost=10 "fix the flaky payment test"
```

- `-a <role>=<cli>[@provider]/<model>[:effort]` 指定每个 role 用哪个后端；
- `-e` 指定 env（在哪台机器/目录跑）；
- budget 只能写成 `-p budget.<limit>=`（duration/cost/output_tokens/graceful），且 `budget` 这个名字不许任何 flow 声明为自己的 param。

## 7. 给外部贡献者的规则（`CONTRIBUTING.md` + `AGENTS.md`）

- 小修（bug/typo/缺测试）→ 直接开 PR；大改（feature/新后端/新 flag/改 flow API/破坏兼容）→ **先开 issue 与 maintainer 对齐方案**，否则"再好的大 PR 也可能被直接关闭"；
- `specs/` 是契约：改代码去符合 spec；改 spec 的 PR 必须 link 到 maintainer 要求改的 issue；AGENTS.md 补充：不许擅改 spec，代码要 minimal；
- PR 标题用 Conventional Commit（`type(scope): ...`），CI 校验标题；分支名 `<type>/<short-slug>`； squash 合并；**无需 sign-off/CLA**（按 Apache-2.0 §5）；
- 测试三级：`tests/unit/<package>`（只测本包 public 名，mock 其他包，不开子进程/网络）、`tests/integration`（按 topic）、`tests/system`（真 agent 真 token，不进 CI，别乱跑）；
- humanize 本身就是用 coding agent 建的，AGENTS.md 允许用 agent 写代码，但"你对你发的每一行负责：读过、跑过"。

## 8. 三个最值得注意的设计决策

1. **Spec-first 且可机检的层架构**：契约是 `specs/*.md` 的 MUST + `test_layering.py` 把"谁能 import 谁"写进测试强制执行。flow 只能 import `hmz.flows` 的 protocol，builtin flows 和外部 flow 走同一条路——这是插件生态（flowverse）的制度保障，不是口号。
2. **Session 作为一等公民 + fork 语义**：把"对话历史"从"执行环境"解耦，fork 让中断的对话能在任意 run 里续上。这是 ralph/stateful 差异的本质，也是 tree-search、多臂回放这类高级 flow 的地基。
3. **Mixin 能力声明取代 harness 分支**：flow 声明能力、runtime 按声明装配，新增 harness 时所有 flow 零改动。

## 9. 与 kda 的关系

kda（Kernel Design Agents）是**领域方法论层**：定义"做 CUDA kernel 优化"这个任务该怎么拆（任务契约、九步循环、evidence 规范）；
humanize 是**执行编排层**：把这类计划变成可运行、可恢复、可审计的 agent loop。kda 的 Minimal Flow 第 6 步明确写
"把 draft 转成可执行 plan，手动或用 Humanize 这样的 planning 工具"，README 还指引安装 humanize 的 Claude Code 插件。

---

*文件索引：`README.md`、`AGENTS.md`、`CONTRIBUTING.md`、`specs/SPEC.md`、`specs/flows.md`、`src/hmz/flows/builtin/`、`src/hmz/flows/agents.py`、`coganchor/backends.py`、`docs/user/concepts.md`*
