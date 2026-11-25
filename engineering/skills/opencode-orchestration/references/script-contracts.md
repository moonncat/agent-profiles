# 脚本合同（伪代码级，按此重写即可复现）

> 三条公共纪律：①凭据从环境/凭据文件读取，不上命令行不进输出；②任何网络失败 = 跳过本轮，绝不误判为「空/无」；③无事零输出（watchdog 语义：空输出=不投递）。

## 1. ticket-dispatcher（每 1-2 分钟，零 LLM）

职责：把「已定指令」机械转成派发触发。

```text
cycle():
  token = read_cred()                          # 缺失则打印一行并退出
  for repo, job in REPO_MAP:                   # repo -> 编排 job 名
    issues = GET /issues?state=open&labels=ready
    if fetch_failed: log + continue            # 网络抖动下轮再扫
    for iss in issues:
      if labels(iss) ∩ SKIP_LABELS: continue   # done/blocked/needs-info
      comments = GET /issues/{n}/comments
      last_dispatch  = 最后一条以 "dispatch:" 开头的评论下标
      last_directive = 最后一条以 "review:"/"rework:" 开头的评论下标
      终阶豁免: "review: passed" 且首行含终阶名（如 TDD）→ 跳过
                                             # 终阶后走 PR 收口，再派=零产出会话
      if last_directive <= last_dispatch: continue   # 指令已被消费
      if state["{repo}#{n}"] == directive_comment_id: continue   # 判重
      if job_building(job): continue           # 查不到状态按「在跑」处理
      ok = trigger(job, ISSUE=n)               # 成败按 HTTP 201 判，不看 exit code
      if ok: state["{repo}#{n}"] = directive_comment_id; save_state
      else: log, 下轮重试
```

要点（每条都是实测教训的凝结）：
- **判重键 = 指令评论 id**：只比 SHA 会把推进（next）误判成返工（revise）；只比时间序会重复触发。
- **构建中跳过**：编排 job 多为 abortPrevious 并发策略，新触发会把在途派发构建整个顶掉。
- **顶掉 = 静默停摆陷阱**：判重锚点已写、构建被顶掉 → `dispatch:` 标记和会话都不会出现 → 锚点让脚本不再重试。故**触发后双验**（锚点评论 + 新会话），缺一即补触发一次（会话与消息在同步调用返回前已落服务端，补触发安全）。
- 触发封装常恒 exit 0：按输出的 HTTP 状态判成败。

## 2. permission-reviewer（每 3-5 分钟）

职责：executor 权限 ask 的快车道审批。常态零 token，有 ask 才调小模型判定。

```text
cycle():
  pending = GET /permission
  if empty or unreachable: return             # 静默；fail-safe 绝不默认放行
  for ask in pending:
    if ask.id in state.decided: continue
    session = GET /session/{ask.sessionID}     # metadata: issue/repo/stage
    verdict = judge(ask, session)              # 小模型 temperature=0，输出 JSON
    if verdict is None: audit(skip); continue  # 判定失败保持 pending，下轮重试
    POST /session/{sid}/permissions/{rid} {"response": once|reject}
    state.decided.append(rid)                  # 定长截断（如留最近 200 条）
    audit(jsonl)；工单留 evidence(permission) 评论（用会话 metadata 定位归属）
    if verdict == deny: 告警到人工通道
```

判定 rubric（写进 judge 的 system 提示，首句先立吓阻）：
1. 首句：「reject 会直接终止 executor 的当前回合，代价极高——只在真正危险时才 reject。」
2. 只读探查（ls/cat/grep，含工作区外普通目录）→ approve；
3. 工作区/临时目录内文件读写、git 操作（feature 分支）、测试/构建命令族 → approve；
4. 真正危险才 reject：删文件（rm）、外传（curl/wget）、改系统配置、装包、访问凭据文件、部署操作、工作区外写入；
5. 发评通道脚本 → approve。
输出仅 JSON：`{"verdict":"approve|deny","reason":"≤40字"}`。

## 3. state 文件约定

| 文件 | 键 | 值 | 语义 |
|---|---|---|---|
| dispatch state | `<repo>#<n>` | 指令评论 id | 该指令已触发过 |
| permission state | `decided[]` | ask id（定长截断） | 该 ask 已裁决 |
| 审计日志 | jsonl 行 | ts / ask / verdict / reason / 归属 | 全量留痕，供事后审计 |

state 文件是**判重缓存不是状态机**——状态机永远在工单评论里；state 丢失最多导致一次重复触发（被双验和锚点兜住），不可反向依赖。

## 4. 应急派发器（编排 job / serve 故障时手动解楔）

逐条复刻编排 job 的 dispatch() 合同（见 `dispatch-job-contract.md` §3），在 CI 之外手工派单：
1. 先发 `dispatch: <kind> @<sha>` 锚点评论（防后续构建重复派发）；
2. 建会话：POST /session，metadata 带 issue/repo/stage/build，权限表随创建冻结；
3. 发任务：POST /session/{id}/message（同步长超时；调用方超时被杀不影响——prompt 在返回前已落服务端）；
4. 验证：status busy + 会话 updated 时间推进（**新鲜会话的消息列表可为空，勿据此判未投递**）；
5. 判投递以 busy + updated 为准，勿重发 prompt（防双回合）。

## 5. 两个小工具的约定

- **发评脚本**：`post-comment.sh <owner/repo> <issue> <正文文件>`——凭据脚本内自取、只回显评论 id；**必须写进权限白名单**（否则 executor 自寻凭据 → 被拒 → 回合报废）。
- **CI 触发封装**：恒 exit 0、输出 HTTP 状态（201=入队）——调用方按输出判成败；长同步调用一律 `|| true` 容错，产出看状态机不看退出码。
