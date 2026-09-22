---
title: "Hermes Agent v0.21 Kanban Swarm：从进程内蜂群到持久化工作队列"
date: 2026-09-22
tags: ["Hermes Agent", "AI Agent", "多智能体", "Kanban", "工作流"]
author: "AlwaysWannaFly01"
---

先讲一个具体的失败现场。你写了一个 orchestrator agent，让它 spawn 三个 subagent 并行调研一个主题，调研完再汇总。它跑得挺好，直到你的上下文窗口被压缩了一次。三个子代理的返回结果、中间的讨论、谁负责哪一块，全没了。你重跑了一遍，结果和上次不一样，因为那些匿名子代理没有记忆，也没留下任何能事后查证的记录。

这不是玄学，这是"进程内蜂群"这个架构的固有毛病：子代理活在调用栈里，父任务一结束，它们存在的证据就跟着上下文一起蒸发。Hermes Agent v0.21 给出的答案是 Kanban：一块跨 profile 共享的持久化任务板，每个任务一行 SQLite 记录，每次交接一行任何人都可读可写的记录，每个 worker 是一个带独立身份的完整 OS 进程。`hermes kanban swarm` 再往上叠一层，用一条命令生成"root 黑板 + N 并行 worker + verifier 闸门 + synthesizer 闸门"的持久化蜂群拓扑。

本文所有命令都可 curl 官方文档核对，所有数字都来自文档或本机实测（Hermes Agent v0.21.3，2026.9.14，upstream 3917c7dd）。文末附来源清单。

## 一、地基：五个原语加一个调度循环

Kanban 没有发明花哨的新抽象。整个系统就五个名词：Board、Task、Link、Comment、Workspace，外面套一个 Dispatcher。

Board 是一个独立的任务队列，拥有自己的 SQLite 数据库、workspaces 目录和 dispatcher 循环。一个安装可以有多块 board，按项目或仓库或领域隔离。默认 board `default` 的数据库在 `~/.hermes/kanban.db`（向后兼容），新建的 board 在 `~/.hermes/kanban/boards/<slug>/kanban.db`。

Task 是一行记录：title、可选的 body、唯一的 assignee（一个 profile 名）、status（triage / todo / ready / running / blocked / review / done / archived 八个状态）、可选的 tenant 命名空间、可选的 idempotency key（给自动化和 webhook 去重用）。

Link 记录 parent 到 child 的依赖。dispatcher 在一个子卡的所有 parent 都 done 之后，把子卡从 todo 晋升到 ready。

Comment 是 agent 之间的协议通道。agent 和人都能追加，worker 被 spawn（或重新 spawn）时会把整条评论线程读进 context。

Workspace 有三种，选择直接决定任务结束后的数据去留：

- `scratch`（默认）：全新临时目录，任务完成即删。想留住某个文件，得在 `kanban_complete(artifacts=[...])` 里显式声明，系统会在清理前把它复制到持久附件存储。
- `dir:<path>`：一个已存在的共享目录，必须是绝对路径，完成后保留。
- `worktree`：git worktree，完成后保留，适合编码任务。

Dispatcher 是一个长驻循环，默认每 60 秒 tick 一次：回收 stale claim、回收 crashed worker、promote ready 任务、原子 claim、spawn 对应 profile。默认跑在 gateway 进程内。同一个任务连续 spawn 失败达到 `kanban.failure_limit`（默认 2）后自动 block，避免 profile 不存在时反复空转。

最后是隔离的两档。Tenant 是 board 内部的软过滤命名空间，一个专家舰队可以服务多个业务，靠 workspace 路径和 memory key 前缀做数据隔离。Board 才是硬隔离边界：每块 board 独立 SQLite，worker 只能看到自己 board 的任务，跨 board 的 link 直接禁止。

为什么重要：这套地基的每一个词都对应"持久化"这个核心承诺。Task 和 Link 让依赖关系可查询，Comment 让交接可追溯，Workspace 的三种形态让"易失"和"保留"成为显式选择而不是隐式行为。进程内蜂群这些东西全都不存在。

## 二、Swarm：一条命令建出一张持久化蜂群图

`hermes kanban swarm` 是唯一一处完整文档化的 Swarm 入口。命令形态逐字如下：

```bash
hermes kanban swarm "Design a multi-region failover plan" \
  --workers researcher,architect,sre \
  --verifier reviewer --synthesizer writer
```

一次调用生成五类卡片，构成一张完整的 DAG：

1. root / blackboard 卡片，立即完成，作为所有 worker 的共享父节点。
2. N 张并行 worker 卡片，`--workers` 逗号分隔，每个名字对应一个 profile。
3. 1 张 verifier 卡片，其 parent 是全部 N 个 worker（fan-in 门）。
4. 1 张 synthesizer 卡片，其 parent 是 verifier。

顺序语义很干净：root 先完成，N 个 worker 并行跑，verifier 等全部 worker 完成后才被 promote，synthesizer 等 verifier 判定通过后才被 promote。三个参数都可选，文档没有列其它 flag，别臆造。

这整张图是原子提交的。官方原话：dispatcher 和 dashboard reader 要么完全看不到新 swarm，要么看到完整拓扑，绝不会看到"root 建了但 worker 还没链上"的中间态。这是靠 kanban_db 层的单事务写入加 SQLite WAL 模式达成的。实际上原子性有三层：建图原子、complete 原子（summary 加 metadata 加状态翻转一次走完）、claim 原子（dispatcher 用 BEGIN IMMEDIATE 抢占 ready 任务，保证同一时刻只有一个 dispatcher 能 claim 到一张卡）。

黑板（blackboard）是 Swarm 在"无共享内存"约束下实现跨进程信息共享的机制。共享的 swarm context 以结构化 JSON comments 存在 root 卡上：

```json
{"key": "topology", "value": {
  "root_id": "t_076f3c3b",
  "worker_ids": ["t_38bd6f4e", "t_8dd02484", "t_2ef9824a"],
  "verifier_id": "t_191095b6",
  "synthesizer_id": "t_301589cc"
}}
```

任何 worker 调用 `kanban_show(root_id)` 或读评论线程都能拿到这份共享上下文。它不靠进程内消息传递，靠 SQLite 里的 durable 行。本文本身就是一次 `kanban_swarm_v1` 实例：root 卡上就挂着这样一份 topology JSON。

为什么重要：原子提交是 Swarm 区别于"进程内 subagent 蜂群"的核心工程保证。进程内蜂群一个子任务崩了，父任务可能停在半截；Swarm 的观察者（dispatcher 扫描、dashboard 的 WebSocket）在任何时刻看到的都是一个自洽的完整状态。

## 三、对比：delegate_task 是函数调用，Kanban 是工作队列

官方对比表逐字如下：

| 维度 | delegate_task | Kanban |
| --- | --- | --- |
| 形态 | RPC 调用（fork → join） | 持久消息队列 + 状态机 |
| 父任务 | 阻塞等子任务返回 | create 后 fire-and-forget |
| 子身份 | 匿名子代理 | 具名 profile + 持久记忆 |
| 可恢复性 | 无，失败即失败 | block→unblock→重跑；崩溃→reclaim |
| 人在环 | 不支持 | 任意时点 comment/unblock |
| 每任务 agent 数 | 一次调用 = 一个子代理 | 一个任务生命周期内 N 个 agent |
| 审计轨迹 | 上下文压缩即丢 | SQLite 中永久 durable 行 |
| 协调方式 | 层级（caller→callee） | 对等，任何 profile 读写任何任务 |

一句话区分：`delegate_task` 是函数调用；Kanban 是"每次交接都是一行任何 profile（或人）可读可改的记录"的工作队列。

什么时候用 delegate_task：父 agent 需要短期推理答案再继续、没有人类参与、结果要回到父 context。什么时候用 Kanban：工作跨 agent 边界、需要跨重启存活、可能需要人类输入、可能被别的角色接手、事后需要可发现。两者可以共存，一个 kanban worker 在 run 内仍能调用 delegate_task。文档还特别警告：delegate_task 的子进程携带 `HERMES_DELEGATED_CHILD_CONTEXT` 会被 fence 出 board，它的 auto-heartbeat 和 kanban_complete 都会被拒。

## 四、协作模式：文档说八种，表里列了九个

这里有一个必须先点破的事实核对结果：官方文档正文写的是 "the eight canonical collaboration patterns"（八种），但紧随其后的模式表实际列到了 P9。多出来的 P9（Triage specifier）是后来补进表里、计数没同步更新。所以严格说是九种，P1 到 P9。

- P1 Fan-out：N 个同角色兄弟卡，"并行研究 5 个角度"。这是最原始也最常被低估的用法。
- P2 Pipeline：角色链 scout → editor → writer，日报装配。用 `--parent` 表达前后依赖。
- P3 Voting / quorum：N 兄弟 + 1 聚合者，3 个研究员 → 1 个 reviewer 抉择。
- P4 Long-running journal：同 profile + 共享 dir + cron，Obsidian vault。
- P5 Human-in-the-loop：worker block → 用户 comment → unblock，模糊决策。
- P6 @mention：从 prose 内联路由，`@reviewer look at this`。
- P7 Thread-scoped workspace：线程内 `/kanban here`，per-project gateway threads。
- P8 Fleet farming：一个 profile 管 N 个主题，50 个社交账号。
- P9 Triage specifier：粗糙想法 → triage → `hermes kanban specify` 展开 body → todo。

用一个可复制的例子说明 P1 扇出。只有 dispatcher 能把 N 个 ready 任务真正并行 claim 时，扇出才成立。dispatcher 每次 tick（默认 60 秒）扫描所有 board，为每个 assignee 池各拉起一个 worker 进程，所以下面三个翻译任务由三个独立进程并行执行：

```bash
for lang in Spanish French German; do
    hermes kanban create "Translate homepage to $lang" \
        --assignee translator --tenant content-ops
done
```

P2 流水线用 `--parent` 串依赖。要注意 `--parent` 不只是调度门，它还是上下文交接通道：子任务 worker 调 `kanban_show()` 时，返回的 worker_context 里带一节 Parent task results，逐字携带父任务完成时的 summary 和 metadata。上游"为什么这么做"的决策随依赖边一起流动，下游不用重读长文档：

```bash
SCHEMA=$(hermes kanban create "Design auth schema" \
    --assignee backend-dev --tenant auth-project --priority 2 \
    --body "Design the user/session/token schema for the auth module." \
    --json | jq -r .id)

hermes kanban create "Implement auth API endpoints" \
    --assignee backend-dev --tenant auth-project \
    --parent $SCHEMA \
    --body "POST /register, POST /login, POST /refresh, POST /logout."
```

P5 人在环是 delegate_task 根本做不到的能力。worker 遇到无法自行决策的问题时 `kanban_block(reason=...)`，任务进 blocked，人类在评论区补上下文再 unblock。关键是 `/kanban` 斜杠命令豁免了"运行中 agent 守卫"，即使 worker 正在跑，你也能从手机端立刻 unblock 或 comment，因为看板数据在 SQLite 里，不在运行中的 agent 状态里。

这九种模式不是九个开关，而是五种原语的组合结果。真实系统里它们会叠加：P2 流水线的某一步再 P1 扇出，P3 聚合后再 P5 让人裁决。读懂它们的价值不在于背会表格，而在于知道：任何"多个 agent 如何交接"的问题，都能落到 Board、Task、Link、Comment、Workspace 这五个词上。

## 五、PR completion contracts：把"绿"变成可验证的闸门

创建任务时用 `--completion-contract OWNER/REPO`（或精确的 PR URL）声明这是 PR 工作。`kanban_create` 接受同名 `completion_contract` 参数。`local-only` 用于有意本地的工作，未声明的卡保留这个默认。

发布后向 completion 传 `metadata.published_pr`，第一个匹配的 URL 永久绑定该卡，重试不能用另一个"绿"兄弟 PR 顶替。共享的 complete_task 边界覆盖 worker 工具、CLI、review 审批和 dashboard 完成，它读经典 branch protection 与 ruleset 的 required contexts，分页检查 exact-head check runs 与 legacy statuses，再重读 PR head/base。

缺失、pending、failed、cancelled、timed-out、stale、skipped、neutral 的 required 证据都不能 complete 卡。零 run 的 acceptance、不可读的策略、GitHub API 失败同样不能。无 required checks 的仓库必须用 local-only contract。`gh` 需要仓库 checks/rules 的读权限认证，这个闸门不做 remote writes。

为什么重要：这一层把"我觉得 PR 可以合并了"替换成"CI 的实际证据说它可以"。拒绝时卡和 workspace 都保留，`pr_acceptance` 事件持久化 URL、SHA、required contexts、check IDs、分类和恢复指引，`last_failure_error` 给出下一步。失败是常态，但失败必须是可解释、可恢复的。

## 六、worker 怎么碰板子：14 个工具，不 shell out

这是 AI 开发者最易误解的一点。文档明确：worker 不 shell out 到 `hermes kanban`。dispatcher spawn worker 时在子进程环境里设 `HERMES_KANBAN_TASK=t_abcd`，这个环境变量翻转模型 schema 里的专用 kanban toolset，工具通过 Python 的 kanban_db 层直接读写板，和 CLI 是同一层。

这么设计有三个理由。一是后端可移植性：远程 terminal 后端（Docker/Modal/SSH）里没有 hermes 二进制，也没挂载 `~/.hermes/kanban.db`，工具在 agent 自己的 Python 进程里跑，总能触达数据库。二是没有 shell quoting 的脆弱性，避免把 `--metadata '{"files": [...]}'` 塞进 shlex/argparse。三是更好的错误：工具结果是结构化 JSON，模型可直接推理，不必解析 stderr 字符串。

工具集共 14 个，分四组。读取：`kanban_show`（默认取 env 里的 task_id，返回 title、body、worker_context、parents、prior attempts、comments）、`kanban_list`。终结：`kanban_complete`、`kanban_request_review`、`kanban_request_changes`、`kanban_block`。协作：`kanban_heartbeat`、`kanban_comment`、`kanban_attach`、`kanban_attach_url`、`kanban_attachments`。编排：`kanban_create`、`kanban_link`、`kanban_unblock`。

spawn 时注入 9 个环境变量：`HERMES_KANBAN_TASK`（任务 id）、`HERMES_KANBAN_DB`（board SQLite 绝对路径）、`HERMES_KANBAN_BOARD`（slug）、`HERMES_KANBAN_WORKSPACES_ROOT`、`HERMES_KANBAN_WORKSPACE`（本任务 workspace 绝对路径）、`HERMES_KANBAN_RUN_ID`、`HERMES_KANBAN_CLAIM_LOCK`（`<host>:<pid>:<uuid>`）、`HERMES_PROFILE`、`HERMES_TENANT`。

一个典型 worker 回合长这样：

```text
kanban_show()                                    # 无参，用 HERMES_KANBAN_TASK
# 模型读 worker_context，用 terminal/file 工具做实际工作
kanban_heartbeat(note="halfway through, 4 of 8 files transformed")
# 更多工作
kanban_complete(
    summary="migrated limiter.py to token-bucket; added 14 tests, all pass",
    metadata={"changed_files": ["limiter.py", "tests/test_limiter.py"], "tests_run": 14},
)
```

生命周期强制"一次 run 恰好一个终结调用"：`kanban_complete`、`kanban_request_review`、`kanban_block` 三选一。否则 worker 进程退出时任务仍 running，kernel 判定 crashed 或 protocol_violation。

exit code 是有语义的。`1` 是普通失败。`75`（EX_TEMPFAIL，rate-limited/overloaded/5xx/timeout/quota）记 rate_limited 且不计失败地 requeue。`78`（EX_CONFIG，credential/model/TLS 这类不可重试修复的错误）首次即触发 circuit breaker，park 到 blocked（sticky），`last_failure_error` 带 provider 原话。连续 spawn 失败达到失败上限会自动 block，避免看板永久空转。

结构化 handoff 的约定：`kanban_complete(summary=..., metadata=...)`，summary 是人读的收尾，metadata 是下游 agent、reviewer、dashboard 复用的机器可读交接。机密、原始日志、token 不进 metadata。

## 七、真实场景：从四个用户故事到五类工作负载

官方 tutorial 用四个带 dashboard 截图的故事讲清这套系统到底解决什么。

故事一，单人开发发功能。设计 schema → 实现 API → 写测试，三个任务用 `--parent` 串成链。只有 schema 一开始处于 ready，后两个在 todo 等父卡完成。worker 被 spawn 时做的第一件事是 `kanban_show()`，干完活后：

```python
kanban_complete(
    summary="users(id, email, pw_hash), sessions(id, user_id, jti, expires_at); "
            "refresh tokens stored as sessions with type='refresh'",
    metadata={"changed_files": ["migrations/001_users.sql", "migrations/002_sessions.sql"],
              "decisions": ["bcrypt for hashing", "JWT for session tokens",
                            "7-day refresh, 15-min access"]},
)
```

schema 一 done，API 卡自动晋升。故事二，舰队 farming。三个 profile（translator/transcriber/copywriter）加一堆互不依赖的任务，`hermes gateway start` 之后人就走了，gateway 内嵌 dispatcher，整条队列在无人干预下被清空。故事三是角色流水线加重试，这是最挣分的地方，完整 review 循环用三个工具调用表达：

```python
kanban_request_review(summary="implemented reset flow",
                      metadata={"changed_files": ["auth/reset.py"], "tests_run": 8},
                      reviewer="reviewer")
kanban_request_changes(reason="Add password-strength validation and make reset tokens single-use.")
kanban_request_review(summary="added zxcvbn validation and single-use reset tokens",
                      metadata={"changed_files": ["auth/reset.py", "auth/tests/test_reset.py",
                                                   "migrations/003_single_use_reset_tokens.sql"],
                                "tests_run": 11, "review_iteration": 2},
                      reviewer="reviewer")
kanban_complete(summary="review passed; acceptance criteria verified")
```

任务 run 历史记下 review_requested → changes_requested → review_requested → completed，每次尝试都有各自的 actor、summary、metadata。故事四讲熔断器和崩溃恢复：连续 spawn 失败后自动 block（缺 AWS creds 的 deploy 任务三次失败进 blocked），以及进程中途死亡（OOM 杀掉的迁移任务）由 dispatcher 轮询 pid 发现后 reclaim，重试 worker 在上下文里看到上一次 crashed，改用分块策略成功完成。

最后一个高频场景：实现卡 done 两小时后 CI 挂了。不要重开 done 卡，完成卡是不可变历史。用 done 卡当 `--parent` 建一张修复卡，父卡已 done，所以子卡直接建进 ready，下一 tick 就能被 claim：

```bash
hermes kanban create "Fix CI: test_backoff_jitter flakes on 3.11" \
    --assignee backend-dev \
    --parent t_impl \
    --workspace worktree --branch wt/ci-fix-backoff \
    --body "CI run #4812 failed after t_impl completed.
FAILED tests/test_retry.py::test_backoff_jitter - TimeoutError
Acceptance: tests/test_retry.py green on 3.11 and 3.12."
```

修复 worker 的上下文里带着 t_impl 的完成 summary 和 metadata，而 CI 日志（父卡完成时尚不存在）放在新卡 body 里。优先给修复卡开新 worktree/branch，checkout 原分支给 worker 的是代码状态而非决策理由，后者靠 parent handoff 携带。

这四类故事合起来覆盖文档开头点明的五类工作负载：research triage（并行研究员 + 分析师 + 写作者，人在环）、scheduled ops（跨周累积成日志的循环日报）、digital twins（带持久记忆的具名助手）、engineering pipelines（分解 → 并行 worktree 实现 → review → 迭代 → PR）、fleet work（一个专家管 N 个主题）。

## 收尾

Kanban 解决的不是"让 agent 跑起来"，而是"让 agent 跑起来之后，交接、失败、重试、审查、追溯这些事怎么办"。进程内蜂群把这些问题全压在调用栈上，栈一退就什么都不剩。Kanban 把它们落成 SQLite 里的 durable 行，任何 profile 或人都能读能写。`hermes kanban swarm` 再把最常用的蜂群拓扑固化下来，一条命令生成一块黑板、N 个并行 worker、一道 verifier 闸门、一道 synthesizer 闸门，整张图原子提交。

它的边界也值得记清楚：Kanban 有意单主机，`~/.hermes/kanban.db` 是本地 SQLite，dispatcher 同机 spawn worker，跨主机共享板不支持。多主机就每台主机独立 board，再用 delegate_task 或消息队列桥接。知道它在哪里停，才知道它在哪里真正有用。

来源：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban（重点 kanban-swarm-topology-helper、how-workers-interact-with-the-board、kanban-vs-delegate_task、core-concepts、collaboration-patterns 章节）、/kanban-tutorial、/kanban-worker-lanes、/kanban-multi-gateway。
