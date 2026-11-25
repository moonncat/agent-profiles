# 无 Hermes 移植变体（opencode + Linux 调度器）

> 适用：环境里只有 opencode 与系统级调度器（cron / systemd timer），没有 Hermes 类 agent 平台的 cron 层。
> 原理：原设计里 agent 平台 cron 只提供两样东西——**独立于被看护进程的定时器** + **agent 会话宿主**。前者 Linux 原生有，后者用「定时器 → opencode API 起会话」替代。本文只写差异合同：触发语义、wrapper、看护、降级 delivery-loop；其余组件与主文 §2~§5 完全一致。
> 依据：调度器必须活在 opencode 之外——平台自带的进程内 cron 随 serve 存活，serve 死 = 调度死（已知限制）；系统级调度器恰好补上这一层。

## 1. 组件映射

| 回路 | 原设计 | 本变体 | 迁移量 |
|---|---|---|---|
| ticket-dispatcher (2min) | no-agent 脚本 | cron 直接跑同一脚本 | 零改动 |
| permission-reviewer (5min) | no-agent 脚本 | 同上（LLM 判定本就是裸 HTTP 调用） | 零改动 |
| supervisor (60m) | agent 会话 | cron → `agent-tick.sh`（wrapper 起 opencode 会话） | 新写 wrapper |
| delivery-loop (15m) | agent 会话 | **降级为纯脚本**（推荐，见 §5） | 新写脚本 |
| （新增）serve 看护 | 由平台兜底 | 系统 timer + 健康检查 | 必须新增 |

## 2. 触发语义差异（最大的坑）

- 原平台 interval 从上轮结束起算 → 天然不重叠；Linux cron 是**绝对时刻触发** → agent 轮次（10~30 分钟级）必然自重叠。
- **每条 agent tick 必须 `flock -n` 或忙则跳过**：双会话互踩是实测事故（共享 checkout 下并行会话互切分支、diff 变空）。
- 被跳过的一跳**不补跑**：巡检是幂等感知任务，下一跳自然覆盖；补跑只会制造追赶风暴。

## 3. agent-tick wrapper 合同（supervisor 轮次的宿主）

```text
agent-tick.sh <role>:
  flock -n /run/<role>.lock || exit 0                 # 防重叠
  pin 探针: 起一次性 "Reply: PONG" 会话验证模型可达
    空回合（HTTP 200 零事件）→ 告警退出，不发本轮巡检   # pin 错 = 静默空转
  sid = POST /session
        {metadata:{role,tick}, permission: <ruleset>}   # 权限创建时冻结
  POST /session/{sid}/message
        {prompt: <自包含巡检提示词>}                     # 同步长超时 + 容错
  验证: 会话消息出现新 assistant 条目                     # 无 → 告警（空回合防御）
  投递: 最终消息非「静」→ webhook / 工单评论 / 文件；否则零通知
  会话卫生: tick 会话用完即弃；定期清理 N 天前旧会话
```

要点：
- **prompt 必须自包含**：会话无跨会话记忆——规程写进 AGENTS.md 或独立指令文件，prompt 引用之 + 本轮上下文（主文的 cron 提示词模板即此用途）。
- 同步 message 调用**超时 ≠ 无产出**：prompt 在返回前已落服务端；超时被杀后勿重发（防双回合），产出以会话消息/评论状态机为准。
- supervisor 自己也要一份权限 ruleset（它要跑 curl/脚本），见 `permission-ruleset-template.md`。
- 「无事只回一个字」静语义保留：wrapper 把「静」翻译成零通知。

## 4. serve 看护（唯一站在 opencode 之外的组件，不能省）

- **为什么必须**：supervisor 自己也是 opencode 会话——serve 死 = 执行器与监督者一起死、无人报信 = 半截闭环。这是本变体唯一的新增组件，也是它比进程内 cron 变体更稳的全部原因。
- 合同：系统 timer 每 5 分钟 curl `/global/health`；连续 N 次失败 → 按部署形态拉起（init 脚本/systemd/s6）+ 告警到人工通道；恢复后补一条「serve 已恢复」通知。
- 看护零 LLM、只读探测 + 重启执行器——它是整个系统的最后一道闭环，不允许任何「顺手优化」逻辑混进来。

## 5. 降级版 delivery-loop 规格（纯脚本，每 15 分钟）

```text
cycle():
  prs = 拉全部 open PR；无 PR 且无待开分支 → 零输出退出
  for pr in prs:
    if 无 "delivery:" 评论:
      if 闸门 job 在跑: continue                        # 跳过本轮，不打断在途
      trigger(闸门, BRANCH=pr.head)                     # 成败按 HTTP 201 判
      评论 "delivery: triggered <job>#<构建号>"
    else if 最新 delivery: 是 triggered:
      核对闸门结果:
        两绿 → 合并 + 评论 "delivery: merged @<sha>"，合并后回读 PR 状态
        有红 → 只留一条 "delivery: gate <失败job#构建号>"，不合并
      合并 409 / 冲突 → 评论留证据，下轮再试
  扫尾: 带 ci:passed 标签的工单分支无 PR → 开 PR 入列（body 引用工单号）
```

- 唯一判断点是「两绿合并」，读构建结果即可判，**不需要 LLM**——这是它能降级成脚本的依据；
- 评论即状态机的全部规则（判重、幂等、回读）原样适用。

## 6. 落地清单（本变体新增项）

1. 脚本目录（如 `~/orch/`）：两个原样脚本 + `agent-tick.sh` + `delivery-loop.py` + `serve-watchdog.sh`
2. 调度表（`/etc/cron.d/orchestration`，或等价的 5 个 systemd timer）：

```text
*/2  * * * *  flock -n /run/od.lock  python3 ~/orch/ticket-dispatcher.py
*/5  * * * *  flock -n /run/op.lock  python3 ~/orch/permission-reviewer.py
*/15 * * * *  flock -n /run/ol.lock  python3 ~/orch/delivery-loop.py
0    * * * *  flock -n /run/os.lock  ~/orch/agent-tick.sh supervisor
*/5  * * * *  ~/orch/serve-watchdog.sh
```

3. AGENTS.md / 巡检指令文件（知识持久化——prompt 自包含的依托）
4. 投递通道一个（webhook / 工单评论 / 文件），并定义「静」= 零通知

## 7. Pitfalls（本变体专属）

| # | 症状 | 根因 | 规则 |
|---|---|---|---|
| N1 | agent 轮次互相打断 / 双会话互踩 | 绝对时刻触发 + 轮次超周期 | flock 必配，跳过不补跑（§2） |
| N2 | 某轮巡检静默消失 | 模型 pin 错 = 200 空回合 | PONG 探针 + 验证 assistant 条目（§3） |
| N3 | serve 挂了全链静默停摆 | 执行器与监督者同宿主 | opencode 外看护（§4） |
| N4 | 规程「配了没生效」 | 会话无跨会话记忆、prompt 不自包含 | 规程进指令文件（§3） |
| N5 | tick 会话无限堆积 | 每 tick 新会话不清理 | 会话卫生：定期清理（§3） |
