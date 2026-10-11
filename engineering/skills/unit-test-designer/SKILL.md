
---

# unit-test-designer

## 0. 元信息

```yaml
name: unit-test-designer
version: 0.1
domain: 单元测试设计（Java / C#）
trigger:
  - 用户要求"写单测 / 补测试 / 提高覆盖率 / 评审测试用例"
  - 用户提交类、方法、接口或规约，并要求验证
  - CI 报红、覆盖率下降、mutation score 不达标时的修复请求
requires_tools:
  - 代码检索/切片（调用图、依赖图）
  - 构建与测试执行（mvn/gradle test、dotnet test）
  - 覆盖率（JaCoCo / coverlet）
  - 变异测试（PIT / Stryker.NET）
  - 可选：SMT/符号执行、差分比对
output_artifacts:
  - test-spec.json      # 用例规格（可机读、可评审）
  - *Test.java / *Tests.cs   # 可编译可运行的测试代码
  - verdict.md          # 结论 + 置信度 + 未覆盖风险
```

**一句话立场**：测试的目的是证伪，而证伪的前提是每条用例绑定一条可被判定的假设；不能执行的测试不算测试，没有预言的用例不算用例。

---

## 1. 七层理论 → 可执行规则映射（压缩版）

| 层 | 理论来源 | 落地为一条硬规则 |
|---|---|---|
| 认识论 | Popper 严峻检验、Lakatos 硬核/保护带、Duhem-Quine、可错论 | R1：每条用例必须声明 `falsifies`（若失败则说明什么）；断言失败只能证伪整体，故必须隔离依赖；输出给置信度而非"已通过" |
| 逻辑形式化 | Hoare 逻辑、最弱前置条件、DbC、代数规范、Curry-Howard | R2：先抽 `{P} S {Q}` 与不变式，再生成输入；断言骨架来自性质而非示例 |
| 测试学公理 | Weyuker 充分性公理、等价类+边界值、分类树、pairwise | R3：禁止跨层复用同一批输入；每层独立建模；用回报递减律作为停止信号 |
| 评估器 | 变异测试、CPH 耦合效应、覆盖率的局限性 | R4：主指标 = mutation score，分支覆盖率仅作下限约束；用例数不是指标 |
| 搜索优化 | SBST 适应度、多臂老虎机、子模去重、Delta Debugging | R5：生成 = 定义适应度函数 + 搜索算子；失败用例必须最小化 |
| 决策经济 | 运行剖面、风险预算、停止准则 | R6：按风险分配用例配额；边际收益<成本即停 |
| 人因与安全 | fault-error-failure 链、错误模式库 | R7：断言应插在错误传播路径上，不只比对外部输出 |

---

## 2. 工具链选型（默认，允许替换但须声明理由）

**Java**

| 职责 | 组件 |
|---|---|
| 运行器/参数化 | JUnit 5（`@ParameterizedTest`、`@TestTemplate`、`@Nested`） |
| 断言 | AssertJ（流式、可读、失败信息好） |
| 隔离 | Mockito（`mock`/`spy`/`ArgumentCaptor`）；构造注入优先于 `mockStatic` |
| 属性测试 | jqwik（首选）或 QuickTheories —— 边界与组合爆炸处 |
| 变异 | PIT（配合 pitest-maven/gradle 插件） |
| 覆盖率 | JaCoCo（行≥80%、分支≥70% 为下限） |
| 架构守门 | ArchUnit（防止测试越界依赖） |
| 备选增强 | EvoSuite/Randoop（回归兜底）、JQF（fuzz） |

**C#**

| 职责 | 组件 |
|---|---|
| 运行器/参数化 | xUnit.net（`[Theory]` + `[InlineData]`/`[MemberData]`）；异步原生支持最佳 |
| 断言 | FluentAssertions |
| 隔离 | NSubstitute（语法最接近自然语言）或 FakeItEasy |
| 属性测试 | FsCheck（含 xUnit 集成 `PropertiesAttribute`） |
| 变异 | Stryker.NET |
| 覆盖率 | coverlet + ReportGenerator |
| 架构守门 | NetArchTest |
| 备选 | TUnit（异步/并发场景）、NUnit（遗留项目兼容） |

**共性约束**：不得引入重量级容器/数据库做"单元测试"——那属于集成测试层，违反 R3。需要外部资源时用 Fake/In-Memory 实现，并在 `test-spec.json` 里标注隔离级别。

---

## 3. 工作流（阶段化，失败即回退）

### Phase 0｜意图锚定（不做这一步就生成的，直接判不合格）
输入：目标类/方法 + 可用规约（接口注释、需求、OpenAPI、领域文档）。
动作：
1. 识别被测单元的职责、可见输入域、可观察输出、可变状态。
2. 区分 **硬核**（业务不变式，失败→改产品代码）与 **保护带**（格式、日志、异常消息措辞，失败→可能改测试）。
3. 列出显式假设：前置条件、后置条件、异常契约、副作用、线程/时序约定。
产物：`assumptions.md`（≤10 条，每条可判定）。

### Phase 1｜输入建模（R3）
- 用等价类划分 + 边界值分析建立分类树：有效类、无效类、边界点、特殊值（null/空/极值/重复/有序逆序/并发交错）。
- 参数间存在组合时用 pairwise 求最小代表集。
- 标注每个代表元的 `coverage_of`（覆盖哪个等价类/哪条路径）。

### Phase 2｜预言设计（最关键，LLM 最易失分）
按优先级选择预言，**至少命中两条独立路径**：
1. **契约预言**：`{P}→{Q}`、不变式、异常类型与状态守恒（抛出异常后对象仍合法）。
2. **性质预言**（属性测试）：幂等、结合、交换、单调、往返同构（serialize↔deserialize）、缩放/平移不变、排序的排列守恒、逆元抵消。
3. **蜕变关系（MR）**：无需标准答案。例：输入排序后输出非降；输入放大 k 倍输出按比例变化；等价输入得等价输出；重试不改变结果。
4. **差分预言**：对照旧版本/另一实现/参考算法（慢但正确的暴力实现作 oracle）。
5. **状态/副作用预言**：协作对象被调用的次数、顺序、参数（用 captor 抓），且必须验证"不该发生的调用没发生"（`verifyNoMoreInteractions` / `ReceivedCalls`）。

> 禁止：只断言"不抛异常"、只断言返回非空、用 `assertTrue(true)` 式占位、让 LLM 自评断言正确性。

### Phase 3｜用例生成（双轨）
- **轨 A：意图性用例**（手写风格，数量少、可读、承载业务语义）——覆盖主成功场景、关键异常路径、业务边界。每个对应一条 `falsifies`。
- **轨 B：机械性用例**（属性测试/pairwise/演化生成，数量多）——覆盖输入空间与组合。由机器维护，人类只审性质。
命名规范：`should_<预期行为>_when_<条件>`（Java）／`Should_<预期行为>_When_<条件>`（C#），名称即断言的自然语言版。

### Phase 4｜隔离与可重复性
- 无共享可变状态、无真实时间（用可注入时钟）、无随机数（固定 seed 并可重放）、无网络/文件系统/DB。
- 每个用例独立、可任意顺序运行、幂等。
- 并发测试须声明调度模型（否则判为 flaky 风险）。

### Phase 5｜执行—反馈—收敛（R4/R5）
循环直到停止准则命中：
1. 运行测试 → 收集失败 → **Delta Debugging 最小化失败用例**（避免给出一百行的冗余反例）。
2. 测覆盖率（行/分支/MC/DC 按需）→ 缺口驱动补充用例。
3. 测 mutation score → 存活突变体驱动补充**能杀死它**的用例（这是唯一能证明"用例有判别力"的证据）。
4. 子模贪心去重：删除不降低充分性的用例。
5. 分类存活突变体：等价突变体 → 记录并豁免（需人工确认）；非等价 → 要么加强断言，要么承认产品缺陷。

### Phase 6｜裁决与交付
输出 `verdict.md`：通过/不通过、mutation score、覆盖率、剩余风险、flaky 标记、已知等价突变体清单、置信度（低/中/高，依据：预言强度×变异杀伤×覆盖完整性）。

---

## 4. 输出 Schema

```json
{
  "unit": "com.acme.BillingService.calculate",
  "language": "java",
  "framework": "junit5",
  "hardcore_invariants": ["应付金额非负", "折扣不超过上限"],
  "test_cases": [{
    "id": "TC-07",
    "name": "should_reject_negative_quantity_when_price_positive",
    "track": "A",
    "input_model": {"quantity": -1, "price": 10.0},
    "falsifies": "若通过则说明负数量未被拒绝（违反前置条件校验）",
    "oracle": ["contract:throws IllegalArgumentException","state:unchanged"],
    "covers": ["EC:quantity<0", "path:precondition-check"],
    "isolation": {"mocks": ["TaxCalculator"], "clock": "fixed"},
    "expected": {"throw": "IllegalArgumentException", "sideEffects": []}
  }],
  "properties": [{
    "name": "roundtrip_json",
    "generator": "jqwik",
    "statement": "forAll(dto) -> serialize(deserialize(s)) == s"
  }],
  "stop_criteria": {"timebox":"30min","mutation_floor":0.75,"branch_floor":0.7},
  "metrics": {"cases":14,"line_cov":0.86,"branch_cov":0.74,"mutation_score":0.78},
  "confidence": "mid",
  "residual_risks": ["浮点舍入在大额累加时未建模","并发重入未测"]
}
```

---

## 5. 停止准则（R6，写死，不许靠感觉）

满足任一组即可停：
- 时间盒耗尽，且 mutation score ≥ 阈值（默认 0.75，核心域 0.85）；
- 连续三轮新增用例未提升 mutation score（回报递减）；
- 单位时间缺陷检出率 < 预设阈值；
- 风险预算耗尽（按失效概率×影响分配的用例配额用完）。

**永不因"看起来够了"或"格式漂亮"而停止。**

---

## 6. 自检清单（生成后逐条过，任一否即退回）

- [ ] 每条用例有 `falsifies` 且非平凡（平凡=在任何合理实现下都通过）
- [ ] 断言至少两条独立预言路径，且不包含 LLM 自评
- [ ] 无真实 IO/时钟/随机/共享状态；用例可乱序重复运行
- [ ] 异常测试验证了"抛了正确的异常 + 状态未被污染"
- [ ] mock 验证了"不该发生的调用未发生"
- [ ] 边界值来自等价类建模，不是拍脑袋的 0/1/null 三连
- [ ] mutation score 已测且报告了存活突变体分类
- [ ] 失败用例已最小化
- [ ] 未在单元层复用集成层的输入集（R3）
- [ ] 用例数不作为任何汇报指标

---

## 7. 失败模式库（诊断表）

| 症状 | 最可能根因 | 修复 |
|---|---|---|
| 全绿但 mutation score 低 | 断言空洞/缺预言 | 补性质与 MR；删弱断言 |
| Flaky | 时间/随机/并发/共享状态 | 注入时钟、固定 seed、隔离 |
| 改产品代码就要改一堆测试 | 测了实现细节而非契约 | 断言移到 observable 行为 |
| 改了测试才能绿 | 保护带/硬核混淆或真缺陷 | 先判定是否真缺陷，禁止直接修断言 |
| 用例爆炸 | 缺 pairwise/属性测试 | 改用生成式 + 子模去重 |
| 失败无法定位 | 违反 Duhem-Quine 隔离 | 加 mock、拆用例、一次一假设 |
| 覆盖率高但线上仍漏 | 缺运行剖面与错误传播路径断言 | 按 operational profile 加权；在传播路径上加断言 |

---

## 8. 附录：可复用资产骨架

**性质库**：幂等、结合、交换、单调、往返、缩放/平移、排列守恒、逆元、德摩根、冪零、闭合性、三角不等、一致性（相等输入→相等输出）。
**MR 库**：排序/过滤/聚合的保序保量、缩放、归一化不变、重试幂等、对称性、局部修改的局部影响、子集一致性。
**变异算子库**：算术运算符替换、边界条件偏移（`<`→`<=`）、null 处理移除、异常吞掉、短路逻辑翻转、集合边界 off-by-one。
**风险预算模板**：核心计费/权限/并发 → 高配额 + mutation≥0.85；工具类纯函数 → 中配额；DTO/ getter → 低配额（甚至只跑属性测试）。

---