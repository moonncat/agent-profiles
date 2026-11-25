# 编排 job 合同（参数化派发 job，如 Jenkinsfile.dispatch）

> 角色：读工单状态机 → 决定派发 kind → 落锚点 → 建 executor 会话。**无定时器**（由 ticket-dispatcher 轮询触发）。
> PIN 约定：job 脚本从默认分支加载；参数只收窄扫描范围，不换脚本——改脚本的验收 = 合并后首次默认分支构建。

## 1. 参数

| 参数 | 语义 |
|---|---|
| ISSUE | 只处理该工单号（空 = 扫描全部 ready） |
| FORCE | 忽略已判 SHA，强制重跑闸门 |
| STAGE | 仅写进会话 metadata 的阶段名（默认实现阶段） |

## 2. 扫描状态机（每单读评论流后决策）

| 条件（评论流 + 分支） | 动作 |
|---|---|
| review:/rework: 指令新于最近 dispatch: | `dispatch(kind=next或rework, @head, 指令全文)` |
| 无 fix-solution 且无 dispatch: 标记 | `dispatch(kind=implement, @-)` |
| 有 dispatch: 但无工单分支 | 等待（澄清中/已停），不重复派 |
| head == gate-result SHA 且 FAILED | `dispatch(kind=revise, @head)`（标记防循环） |
| head == gate-result SHA 且 PASSED | 收口（PR 合并由 delivery-loop 做） |
| dispatch: 已含当前 head | 等待 |

判重锚点就是 `dispatch: <kind> @<sha>` 评论本身——评论即状态，天然再入安全；判重键必须含 kind（推进与返工同 SHA 时语义不同）。

## 3. dispatch() 合同（顺序不可换）

1. **落锚点**：POST 评论 `dispatch: <kind> @<sha>`（先于一切副作用）；
2. **建会话**：POST /session，body 带 `metadata: {issue, repo, stage, build}` + 权限表（**创建时冻结**，改规则不热更）；
3. **发任务**：POST /session/{id}/message，prompt 自包含（见下），同步调用长超时（可达数十分钟）+ `|| true`；
4. **回显**：会话 id 打进构建日志。

**prompt 自包含要素**（executor 不知任何聊天历史）：
- 工单号 + 标题 + 为什么做；
- 分支约定：**显式从 origin 主干起分支**（共享 checkout 下默认 HEAD 起分支会叠上他单提交）；
- 精确文件清单——清单外不得改动（改形状前先全仓 grep 旧形状断言补全清单）；
- 已实测引证（file:line）——**引证先行**：缺引证会让 executor 自行开探索子代理，实测楔死 30 分钟级；
- 阶段流转说明（review: passed 进阶 / rework: 按意见改，保留正确部分）；
- 命令保证退出码 0（必要时 `|| true`）、报错换路径继续，绝不终止回合；
- 产出回评要求（fix-solution/evidence）+ 发评通道脚本用法（并声明不允许自寻凭据）；
- 返回格式：分支、改动文件、闸门输出摘要。

## 4. 运行约束

- **无定时器**：定时触发 + abortPrevious 并发策略会掐死在途派发构建（可达几十分钟）；高频轮询放外面的轻脚本（先查 building 再 trigger）。
- **模型 pin 先探针**：未知模型 id / 端点不匹配的 key 会被接受后**静默空回合**（HTTP 200、零事件、零产出）——派发前用一次性「Reply: PONG」会话验证 pin，空回合即查 serve 日志与端点。
- 同步 message 超时 ≠ 无产出：产出以评论状态机为准，绝不重发 prompt。
- 同机共享 checkout 时**禁止并发双会话**（互踩实测事故：A 单提交后 B 单切分支，A 的 diff 变空）：一次只派一单，或每单独立 worktree。
- 同签名基础设施红（DB 超时类）判环境瞬态：原地重触发不返修，判据写 decision: 留档；只有代码语义失败走 rework。
