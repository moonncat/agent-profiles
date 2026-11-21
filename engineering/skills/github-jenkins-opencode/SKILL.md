---
name: github-jenkins-opencode
description: Gitea/GitHub + Jenkins + opencode 的 issue 驱动双仓交付流水线：评论状态机、闸门 lane、幂等派发、经验与教训库
---

# github-jenkins-opencode

## 0. 元信息

```yaml
name: github-jenkins-opencode
version: 0.1
domain: 工程流程（issue 驱动开发 + CI 闸门 + agent 执行器编排）
trigger:
  - 搭建或维护「issue 驱动 + CI 闸门 + AI 编码执行器」的交付流水线
  - 流水线排障：停摆、重复派发、闸门红、合并卡住、agent 只答不干活
  - 把 opencode 类执行器接入 git 托管 + Jenkins 的既有项目
requires_tools:
  - git 托管 REST API（Gitea/GitHub：issues、labels、PRs、comments）
  - Jenkins REST API（job 触发、config 管理、构建历史）
  - opencode serve API（session 创建 + 同步消息）
  - 轻量调度器（cron：watcher / dispatcher / delivery loop）
output_artifacts:
  - 工单机器指令评论（review:/rework:/dispatch:/decision:/gate-result:）
  - 闸门构建记录（l1/l2/l3 验证 lane）
  - 通过闸门合并的 PR
provenance: 实战于 Gitea + Jenkins + opencode 的双仓（上游服务仓 + 下游消费仓）交付流水线
```

**名称说明**：取「git 托管 + Jenkins + opencode」工具链三件套之意。托管侧以 Gitea 实测，GitHub 的差异点（标签 API、Actions 替代方案）单列在 §4.6。

**一句话立场**：工单评论区是唯一状态机，agent 只产不裁、CI 只裁不产；自动触发全部幂等可重入，人工裁决必须回写成机器可读指令——四条缺一，流水线必停摆。

---

## 1. 理论 → 可执行规则映射

| 层 | 理论来源 | 落地为一条硬规则 |
|---|---|---|
| 反馈控制 | MAPE-K、闭环控制 | R1：每个环节必须成环（感知→比较→决策→执行→再感知）；有监控无执行器 = 半截闭环，宁可不建 |
| 状态机 | 单一事实源、事件溯源 | R2：流程状态只存在工单的机器指令评论 + 标签里；工单正文与 chat 记录都不是状态，机器不读 |
| 幂等 | at-least-once + 去重（exactly-once 的工程退化） | R3：一切自动触发允许重复发生，判重键必须含「动作类型 + 对象锚点」，只比 SHA 会把推进误判成返工 |
| 职责分离 | SoD、双人复核 | R4：执行器只产出（改文件/提交/评论），裁决（verdict 标签、合并）只由 CI 写；产物方不得自证 |
| 控制论 | 时滞—增益、振荡 | R5：高频机械动作（轮询/触发/审批）交给无 LLM 轻脚本；低频判断动作才给 agent。对慢回路施高增益 = 振荡 |
| 传感器铁律 | Goodhart 定律 | R6：不得为过卡点而削弱卡点（跳测试、mock 真语义、降阈值）；HTTP 200 不是传感器，**回读确认**才是 |
| 故障隔离 | 最小权限、优雅降级 | R7：一次权限拒绝会报废执行器整个回合——审批只拒真正危险操作，命令必须保证退出码 0 后继续 |

---

## 2. 总体架构（组件地图）

```
工单评论区（状态机）
   ↑ 写指令/标记          ↓ 读指令
supervisor（agent，60m 级）→ 转译人工裁决、放行积压
dispatcher（no-agent，1-2m）→ 发现待派发 → trigger 编排 job
Jenkins orchestrator（无定时器，API 触发）
   → 派发 executor 会话（opencode serve）
   → 触发闸门 lane → 回写 gate-result: / dispatch: 标记
delivery loop（15m 级）→ 闸门全绿的 PR 自动合并
```

| 层 | 组件 | 职责 | 节拍 | 验收传感器 |
|---|---|---|---|---|
| 状态机 | 工单评论区 + 标签 | 驱动阶段流转；`dispatch:` 标记 = 再入判重锚点 | — | 最新指令 vs 最新标记的时间序 |
| 监督 | supervisor（agent 型） | 人工裁决→机器指令转译、executor 卡死代执行、放行积压单 | 60m 级 | 调度记录里它每轮的实际动作 |
| 派发 | dispatcher（no-agent 轻脚本） | 扫 `ready` 工单：指令比标记新且编排 job 空闲 → trigger | 1-2m | 派发锚点文件出现 `<repo>#<n>: <指令id>` |
| 编排 | orchestrator job（Jenkins） | 参数化（ISSUE/STAGE/FORCE）派发 executor 会话，回写标记 | API 触发 | 构建历史参数 + console 派发行 |
| 执行 | opencode executor | 一次 run = 一个会话：改文件/commit/push/评论 | 按需 | serve `GET /session` 最新 `time.updated` |
| 闸门 | `<repo>-l1-unit` / `-l2-api` / `-l3-ui` | 验证 lane，全绿才合并 | PR 触发/链式 | 构建结果 + `gate-result:` 评论（带 SHA） |
| 交付 | delivery loop | 只做 PR 闸门验证 + 绿灯合并，不做阶段派发 | 15m 级 | `delivery:` 评论 + PR 已合并 |

**分层纪律**：latency 敏感的机械动作（触发、轮询、审批）一律 no-agent 高频轻脚本；agent 型 interval 任务的实测节拍是「上轮结束 + 3~40min 轮次」，即 60~90min 一跳，救不了延迟——把两者混在一个 agent cron 里是常见错误设计。

---

## 3. 工作流：工单生命周期

### 3.1 类型与阶段

| 标题前缀 | 流程 | 分支 | 阶段链 |
|---|---|---|---|
| `[REQ]` | 新功能 | `feat/issue-N` | 澄清 → REQ 文档 [review] → DES 文档 [review] → TDD+实现 → 闸门 → 合并 |
| `[BUG]` | 缺陷修复 | `fix/issue-N` | 根因分析 [review] → 测试+修复 → 闸门 → 合并 |
| 无前缀 | 询问用户 | — | 评论区澄清后再定型 |

REQ 实际是 bug → 按 REQ 走完不切换；BUG 需要改架构 → 评论说明、关单、建议转 REQ。每阶段产物落 `docs/` 并 commit，评论区发「请 review」即挂起，等下一条指令。

### 3.2 机器指令词汇（人读自然语言，机读前缀）

| 前缀 | 含义 | 谁写 |
|---|---|---|
| `review: passed <stage>` | 本阶段通过 → 推进下一阶段 | 人（或代转译的 supervisor） |
| `rework: <要点>` | 按意见修改当前产物 | 同上 |
| `decision: <裁决值+依据>` | 人工拍板记录 | supervisor 回写 |
| `dispatch: <kind> @<sha>` | 派发记录 = 再入判重锚点 | 编排 job |
| `gate-result: sha=<sha> <verdict>` | 闸门裁决 | Jenkins（唯一有权写 verdict） |
| `delivery: ...` | 交付环动作记录（触发/合并） | delivery loop |

标签词汇：`ready`（可派发）/ `blocked` / `needs-info` / `done`；裁决类 `ci:*` / `qa:*`。**dispatcher 只扫 `open + ready` 且不带 `done/blocked/needs-info` 的工单。**

### 3.3 决策两层制（review 的决策政策）

- **L1 自主**：依项目现有模式或行业惯例可定的 → 直接定，产物中标注「L1 决策：依据 \<模式/惯例\>」，不等人工。
- **L2 用户决策**：新增依赖 / 新增基础设施 / 影响既有功能（架构重构通常落入此项）→ 不得自决，评论区列**编号清单**请人拍板。
- 拿不准按 L2 处理；review 请求只圈 L2 项。

### 3.4 裁决回写闭环（chat 收裁决后必做，按序四件事）

状态机只读评论 + 标签——裁决只留在 chat 或正文等于没裁，流水线永远报「待裁决」。

1. 受影响工单各一条 `decision:` 记录评论；
2. `review: passed ...` 指令评论（机器触发锚）；
3. 标签翻转（`blocked`→`ready`；用 labels 专用端点，见 L10）；
4. 验证派发锚点落库（2~3 分钟后仍无 → 手动 trigger，判重键保证安全）。

用户裁决「先排队」时只落 ①③，②④ 延后到开工令。

### 3.5 立项四件事（零评论新工单的特殊性）

判重逻辑是「最新指令比最新 `dispatch:` 标记新才触发」——**零评论的新工单永不自动派发**。立项按序落：① `decision:` 拍板记录 → ② `ready` 标签 → ③ 指令评论（机器触发锚）→ ④ 验证锚点。排队变体只落 ①②。

### 3.6 PR 闸门与合并规程

1. 开 PR 后**只触发一级闸门**（如 `-l1-unit`）；二级闸门（`-l2-api` 等）由一级链式触发，不要在开 PR 时双触发——重复触发的二级构建会占住 executor 道（实测单次占道 80min 级）。
2. 两绿自动合并，由 delivery loop 收口（`delivery:` 评论留痕）；`rework` 指令走正常返工循环，不影响合并判定。
3. executor 漏交 PR（分支 + 解决方案评论都在、PR 没开）→ 监督侧代补开 PR，不要空等。
4. 闸门红 → `gate-result: FAILED` + `dispatch: revise @<sha>` 派发返工；SHA 变化 = 修订循环重跑闸门。

---

## 4. Jenkins 侧约定

### 4.1 Pipeline 存放架构（分阶段，勿跳级）

- **P0**：各业务仓放自己的**薄 Jenkinsfile**（触发条件、agent、步骤）——代码与 pipeline 改动原子提交；
- **P1**：同类 stage 重复 ≥2 次才抽 shared library（早抽 = 伪有序）；
- **P2**：job 数 ≥3 或重建过一次 Jenkins 后，job 定义转 Job DSL 入独立 `ci-config` 仓（job 只存在于 UI = 无真相源的孤儿 artifact）。

否决过的方案（理由固定，别再论证）：单仓存全部 pipeline（爆炸半径 + 无法原子提交）；纯 UI 手建 job（重复率超标 + 无 git 真相源）。

### 4.2 闸门 lane 命名与 PIN 约定

- lane 命名 `<repo>-l1-unit` / `<repo>-l2-api` / `<repo>-l3-ui`，一仓可多条，只做验证；
- orchestrator 一仓一个（如 `<repo>-dispatch`），**不加定时器**——`abortPrevious` 语义下定时触发会把正在同步等执行器（可达 40min）的在途构建中途掐死；要高频轮询就在外面放轻脚本（先查 `lastBuild[building]` 再 trigger）；
- **PIN 约定**：pipeline 脚本从默认分支加载（CpsScmFlowDefinition 钉主干），参数 `BRANCH` 只移动 workspace checkout。推论：**改 Jenkinsfile 不能用本分支构建自验**——验收 = 合并后首次默认分支构建，合并前在本地按文件内命令等价模拟。

### 4.3 Pipeline 沙箱实测约束（Jenkins 容器）

- 容器常无 python3/jq → JSON 解析用 `groovy.json.JsonSlurper`；HTTP 载荷 `writeFile` + `curl --data-binary @file`（内联 JSON/换行会炸）。
- **沙箱白名单很窄**：`JsonOutput.toJson`、`new ArrayList`/`new LinkedHashMap` 一律被拒 → JSON 序列化手写递归字符串拼接，集合只用字面量 `[]`/`[:]`。
- **CPS 序列化**：`JsonSlurper` 实例与其 LazyMap 产物不能驻留 step 之间的帧（NotSerializableException）→ 「parse + deepPlain 转朴素 Map/List」收进一个辅助方法，帧销毁后只回传值。
- `disableConcurrentBuilds` 的参数名是 `abortPrevious`。
- Crumb 与会话 cookie 绑定：取 crumb 和 POST 必须同一个 cookie-aware 连接。
- job 建改：`POST /createItem?name=X` 带完整 config.xml（拿同类 job 的 config 改）；config.xml 直接 PUT 在新版 Jenkins 恒 400。改 job 配置删旧建新，原地编辑会漂移。
- 声明式 `parameters {}` 首次构建前可能未同步进 job（`params.*` 为 null）→ 代码里带默认值兜底。

### 4.4 编排 job 纪律

- 再入安全：状态放评论标记（`dispatch: <kind> @<sha>`、`gate-result: sha=`），按「分支 head vs 已判 SHA」决定派发/跑闸门/收工；**判重键必须含 dispatch kind**（见 L1）。
- 触发器脚本的 trigger 成败按返回的 HTTP 状态判（201 = 已入队），**退出码恒 0 的封装很常见**，shell 里别只看 return code。
- 长同步调用（executor 会话 POST 可达 40min）加容错（`|| true`）——超时 ≠ 无产出，产出看分支/评论状态机，别让非零退出拖垮构建。

### 4.5 命名与变更纪律

新增/改名 job、常驻脚本等命名资源前，把确切名称清单报用户确认；命名全词不缩写、cron job 与其脚本同名、沿用组件前缀与 lane 编号约定。

### 4.6 GitHub 差异点

GitHub 侧等价替换：labels/issue API 语义相近但字段与分页不同；CI 若用 GitHub Actions 替代 Jenkins，闸门 lane 对应 required checks，orchestrator 对应 repository_dispatch/workflow_dispatch + 外部 dispatcher，判重与状态机规则原样适用。

---

## 5. opencode executor 约定

- **会话模型**：一次 run = 一个会话（`POST /session` + 同步 message）；跨任务续聊用固定 session_id 复用上下文。
- **只产不裁**：executor 改文件/commit/push/发评论，verdict 标签与合并只由 CI/交付环写。
- **权限审批**：一次拒绝 = 整个回合报废（模型不再被调用直接收工）。只拒真正危险操作（删数据/外传/改系统/碰凭据），只读探查一律放行；派发提示要求命令保证退出码 0（`|| true`）、报错换路径继续。
- **进度传感器**：会话消息里的 tool 部分 = 实际跑的命令，最后一条 text = 阶段自述；对照工单锚点评论确认阶段，别只看进程存活（进程活着 ≠ 在干活）。
- **发布去重**：要求执行器发评论前先读已有评论，相同则跳过（防重复播报）。
- **手尾补救**：分支 + 解决方案评论在档而 PR 未开 = 漏 hand-off，监督侧代补开，不留悬案。

---

## 6. 双仓契约（上游服务仓 + 下游消费仓）

来源实践：上游服务仓（后端 + 题库/契约）与下游消费仓（小程序前端）各自独立仓库、独立闸门，契约先行。

1. **契约事实源唯一**：跨仓契约（接口、字段、语义）以上游 `docs/` 的 REQ/DES 文档为准；下游按契约消费（如专用查询端点），契约变更先改上游文档再动两侧代码。
2. **元数据多处同源的同步铁律**：领域表/字段元数据常同时存在于「契约文档 → 后端注册表 → 前端表定义」三处——增表/改字段必须三处同改互核，单改一处必漂移。两处白名单（控制器 + 服务层）同理。
3. **拆分 vs 合并用探针决策**（勿拍脑袋）：
   - 提交共现率（近 30 天 FE-only/BE-only/both）：单侧改动占绝对多数（实测 71%~93%）→ 必须拆，合并 = 大部分构建是无效触发；
   - 测试自包含性（逐文件 grep 网络/DB 调用）：自包含的一侧先上 CI；**把坏传感器绑在好传感器上 = 两者一起失去信号**（外部依赖超时会把稳定绿测试拖死）。
4. **测试外部依赖的处置阶梯**：A. 指向可达外部服务 + filter 隔离不兼容用例（零新依赖，先上线）→ B. 嵌入式真实 DB（新增第三方依赖 = 需用户批准）→ C. Mock/Fake——**当测试本身在验证真实存储语义时否决**（mock 真语义 = 测试 mock 自己，闸门虚设）。
5. **老版本基础设施的语义坑**：旧 DB/运行时不支持新语义（如副本集事务）时，测试卡住后失败不是网络故障——用 filter 显式排除并标注升级路径。

---

## 7. 经验与教训

### 7.1 成功经验（照做）

1. **评论区即 review 机制**：全部评审、裁决、派发留痕在工单评论，天然可审计、可重放；chat 只做交互，收口必回写（§3.4）。
2. **fire-and-forget + 判重幂等**：调度器只负责「检测 + 触发」，不等执行器；重复触发被判重键免疫——这让每个组件都可以放心重试，整条链在 at-least-once 语义下稳定。
3. **机器指令前缀**：人读自然语言 + 机读前缀，人不必记语法、机器不必懂自然语言；supervisor 负责转译。
4. **决策两层制**：L1 标注依据自主推进、L2 编号清单上抛——既不把人拖进每个取舍，也不让 agent 自决依赖/架构。
5. **闸门分层 + 两绿自动合并**：l1 快闸做主验证、慢闸链式跟随；绿灯合并由 delivery loop 收口，人只介入红灯与 rework。
6. **只产不裁的职责分离**：执行器无法自证，verdict 单一来源——CI 报告可信度由此保住。

### 7.2 教训库（症状 → 根因 → 规则）

**状态机与幂等**

| # | 症状 | 根因 | 规则 |
|---|---|---|---|
| L1 | 闸门红后返工永不派发，issue 停摆 | 判重只比 `@sha == head`，把 `dispatch: next`（推进）误当成 `dispatch: revise`（返工） | 判重键 = **kind + sha**；外部触发器用「指令评论 id」判重天然免疫 |
| L2 | 裁决后流程仍报「待裁决」僵死 | 裁决只写进工单正文或留在 chat，而状态机只读评论指令 + 标签 | 裁决必走 §3.4 四件事；正文只作人读摘要 |
| L3 | 新工单永远不被自动派发 | 判重逻辑是「指令比标记新」，零评论工单恒不满足 | 立项四件事（§3.5）；「先排队」变体只落 decision + 标签 |
| L4 | 同一结论重复播报 N 条 | 执行器每轮独立会话、发布不去重 | 发布前读已有评论去重；会话用固定 session_id 复用 |

**派发与时滞**

| # | 症状 | 根因 | 规则 |
|---|---|---|---|
| L5 | 在途 40min 的派发被中途掐死 | orchestrator 挂了定时器 + `abortPrevious`，新触发 abort 旧构建 | 编排 job 无定时器；外部轻脚本轮询（先查 building 再 trigger） |
| L6 | 指令发出后 60~90min 才动 | 把触发/轮询放进 agent 型 cron（轮次 3~40min + interval 从上轮结束起算） | 机械动作 no-agent 1-2min 轻脚本；agent 留给判断 |
| L7 | 重复触发二级闸门，单次占道 80min 级 | 开 PR 时同时手动 trigger 多条 lane，又叠加链式触发 | 开 PR 只 trigger 一级；二级走链式；出 PR 留 `delivery:triggered` 标记防双触发 |

**执行器与权限**

| # | 症状 | 根因 | 规则 |
|---|---|---|---|
| L8 | 执行器回合秒收工、零产出 | 权限拒绝/工具报错会终结整个回合（模型不再被调用） | 审批只拒真正危险操作；派发提示要求命令 `|| true` 保证退出码 0 |
| L9 | 日志 0 字节、结果丢失，gateway 侧却跑完了 | 启动包装的 curl 超时 < 实际运行时长 + `set -e` 在打印前终止脚本 | 超时上限 > 最坏 run 时长；curl 段包 `set +e` 并必达打印 rc/状态；**超时 ≠ 无产出**，产出看状态机 |

**托管 API 陷阱**

| # | 症状 | 根因 | 规则 |
|---|---|---|---|
| L10 | 改标签的请求 200 但标签纹丝不动 | 通用 issue 更新端点带 labels 字段是静默 no-op | 用 labels 专用端点（增/删/替换），改完 **GET 回读**——请求成功 ≠ 状态已变 |
| L11 | issue 列表翻页只见 PR，过滤后全空 | issue/PR 共享编号空间挤占分页；issue 对象恒带 `pull_request` 键（纯 issue 为 null），`in` 判断恒真 | 真值过滤 `if d.get('pull_request')`；清单核对以逐号 GET 兜底枚举为准 |
| L12 | 中文评论被误分类为「关闭」 | 关键词匹配把「……就完成」类指令句命中 CLOSE | 忽略自动分类结果，直接读评论语义定意图 |

**CI 与环境**

| # | 症状 | 根因 | 规则 |
|---|---|---|---|
| L13 | `git commit` 在钩子下必挂（spawn EINVAL / rc 234） | 某些 FS/容器环境下钩子内 spawn 子进程必炸，与消息格式无关 | 闸门不跳过：改用纯 lint 库离线校验消息后 `--no-verify` 提交（守恒：验了格式，换了实现） |
| L14 | agent 本地合并失败、流程停摆 | worktree 长期占用受保护分支，主工作区无法 checkout | worktree 用完即删；长期独立目录用独立 clone，不占受保护分支 |
| L15 | clone/push 偶发 401，脚本静默空转 | remote URL 内嵌明文 token 过期；HTTP 错误被吞 | 凭据走 credential.helper 指向凭据文件（不进代码不进 URL）；脚本启动先做认证探活，401 显式告警 |
| L16 | 插件/技能「配了却没生效」 | 加载失败零报错、列表里有 ≠ 已加载 | 唯一可信传感器 = **功能探针**（调用插件专有工具看是否 NO_TOOL） |
| L17 | 三处元数据静默漂移，跨仓契约对不上 | 表/字段定义散落文档、后端、前端三处，只改一处 | 增改三处同改互核（§6.2）；把「三处 diff」做成闸门检查更好 |
| L18 | 报「请求成功」但事没办成 | 触发封装恒 exit 0 / HTTP 200 不代表状态翻转 | 一切外部写操作**回读验证**后才报成功（L10 的推广） |

### 7.3 三条元教训

1. **所有停摆最后归结为两类**：判重键设计错（重复/漏触发），或状态没回写（机器读不到人的裁决）。排障先查这两处，再查网络。
2. **时滞问题不能靠加 agent 解决**——agent 是慢回路，把它放进快回路只会放大延迟与成本；拆成「no-agent 快回路 + agent 慢回路」。
3. **规则也是熵源**：教训条目多了以后要合并、提炼、删过时项（如 L12 的根因修掉后规则可退役），否则规则库本身变成下一处停摆点。

---

## 8. 新项目落地清单

1. **前置**：git 仓 + Jenkins 可达其 remote；凭据入 Jenkins Credentials / 凭据文件（不进代码）；runner 工具链就绪。
2. **P0 最小闭环**（先跑通一条）：
   - 业务仓薄 Jenkinsfile + 一条 `<repo>-l1-unit` 闸门 lane；
   - 一个 orchestrator job（无定时器，参数 ISSUE）+ 一个 no-agent dispatcher 脚本（1-2m）；
   - 一个 executor（opencode serve）+ 权限策略（只拒危险）；
   - 工单模板 + 指令词汇（§3.2）+ 标签集。
3. **验证闭环**（每个组件一个传感器，§2 表末列）：建一张试运行工单，走完「立项四件事 → 派发 → 产物 → 闸门 → 合并」，逐层核对传感器读数。
4. **P1+**：重复 stage 抽 shared lib；二级闸门链式；delivery loop 自动合并；supervisor 转译层。
5. **红线**：状态机只读评论+标签；verdict 只有 CI 写；编排 job 无定时器；判重键含 kind——这四条落地时就要有，不能后补。

---

## 9. 停摆快速诊断表（自上而下，每层一个传感器）

| 顺序 | 查什么 | 待派发 / 在途的判据 |
|---|---|---|
| 1 | 工单评论 | 最新机器指令 vs 最新 `dispatch:` 标记：指令新 = 待派发；标记新 = 在途 |
| 2 | 编排 job 构建历史 | 参数（ISSUE/STAGE）与起止时间 |
| 3 | executor serve 会话列表 | `time.updated` 最新会话 = 当前/最近 run；无新会话 = 未派发 |
| 4 | 调度器执行记录 | supervisor/dispatcher 每轮实际做了什么 |
| 5 | 执行器会话消息尾部 | 最近命令 + 阶段自述，定位卡在哪一步 |

| 症状 | 先查 |
|---|---|
| 闸门红后无人返工 | L1（判重键）+ 诊断表 1 |
| 裁决后仍「待裁决」 | L2（回写闭环）|
| 新工单不动 | L3（立项四件事）|
| 有构建无产出 | L9（启动包装超时）+ 诊断表 3 |
| 有产出无评论 | 执行器提示是否含「必须回评」块（L4 同源）|
| 「请求成功」但状态没变 | L10/L18（回读验证）|
