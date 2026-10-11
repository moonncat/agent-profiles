# agent-profiles

[English](./README.en.md)

收录各领域（软件工程、业务数据分析等）的 `AGENT.md` 与 `skills`，为用户提供**有专业领域理论指导**的 AI Agents。

---

## 背景

通用大模型 Agent 往往只靠一句「你是一个领域专家」来设定角色，行为缺乏稳定依据：既没有可复核的理论框架，也没有可判定的执行标准，结果高度依赖模型的即兴发挥。

而真实的专业工作是有理论支撑的——软件工程有熵增、闭环控制、测试充分性理论；业务数据分析有因果推断、指标体系、实验设计。这些理论只有被写成**可执行的规则、卡点与工作流**，Agent 才能像专业人员一样被约束、被评审、被迭代。

本仓库按领域沉淀两类产物：

- **`AGENT.md`**：领域 Agent 的系统提示词，定义身份、原则、执行模型与硬约束，决定 Agent「以什么方式工作」；
- **`skills/`**：面向具体任务的可复用技能，把领域理论拆成阶段化流程与自检清单，决定 Agent「接到某类任务时怎么做」。

文件本身与具体 Agent 平台解耦，可接入 opencode、Claude Code 等支持自定义 agent / skill 的工具。

---

## 目标

1. **沉淀领域资产**：按领域沉淀 `AGENT.md` 与 skills，逐步覆盖软件工程、业务数据分析等方向。
2. **理论落到规则**：不写空泛的「专家人设」，而是把领域理论转成可判定的硬规则、阈值、卡点与阶段化流程。
3. **平台无关、随处可用**：同一份文件可接入不同 Agent 平台，接入成本只在补一段 frontmatter。
4. **可评审、可演进**：所有内容以 Markdown 入库、走版本管理，可被 PR 评审、可追踪演进。

---

## 仓库目录结构

按「一个领域一个目录」组织，领域内包含该领域的 Agent 定义与其技能集：

```
agent-profiles/
├── README.md                          # 本文档（中文）
├── README.en.md                       # English version
├── engineering/                       # 领域：软件工程
│   ├── AGENT.md                       #   领域 Agent：工程控制代理
│   │                                  #   （熵评估 → MAPE-K 闭环 → 熵账本 → 硬卡点）
│   └── skills/                        #   该领域的技能集
│       └── unit-test-designer/        #     技能：单元测试设计
│           └── SKILL.md               #     （理论→规则映射、阶段化工作流、停止准则）
└── <domain>/                          # 未来领域（如 business-data-analysis）
    ├── AGENT.md                       #   该领域的 Agent 定义
    └── skills/
        └── <skill-name>/SKILL.md      #   该领域的技能定义
```

约定：

- `<domain>/AGENT.md`：领域 Agent 系统提示词，一个领域一份。
- `<domain>/skills/<skill-name>/SKILL.md`：技能定义，目录名即技能名，一个技能一个目录。
- 新增领域直接新增同级目录即可，互不影响。

---

## 使用方法（以 opencode 为例）

opencode 从固定位置加载自定义 agent 与 skill，因此只需把本仓库的文件复制到对应目录。agent 与 skill 都放在项目级 `.opencode/` 下（当前项目生效）；若想全局生效，见文末说明。

### 0. 前置条件

已安装 [opencode](https://opencode.ai/docs/)，并在你的项目中初始化（存在 `.opencode/` 目录或 `opencode.json`）。

```bash
git clone https://github.com/moonncat/agent-profiles.git ~/agent-profiles
cd your-project
```

### 1. 接入 AGENT.md

opencode 的 markdown agent 放在 `.opencode/agents/<name>.md`，**文件名即 agent 名称**。

```bash
mkdir -p .opencode/agents
cp ~/agent-profiles/engineering/AGENT.md .opencode/agents/engineering.md
```

`AGENT.md` 是平台无关的正文，接入时需在文件**顶部补充 frontmatter**（`description` 为必填项）：

```yaml
---
description: 软件工程领域的工程控制代理，以熵评估与 MAPE-K 闭环约束代码变更
mode: all
---
（原 AGENT.md 正文保持不变……）
```

可选字段：`mode`（`primary` / `subagent` / `all`）、`model`、`temperature`、`permission` 等，详见 [opencode Agents 文档](https://opencode.ai/docs/agents/)。

### 2. 接入 skills

skill 放在 `.opencode/skills/<skill-name>/SKILL.md`：

```bash
mkdir -p .opencode/skills
cp -r ~/agent-profiles/engineering/skills/* .opencode/skills/
```

同样，`SKILL.md` 需以 YAML frontmatter 开头，`name` 与 `description` 必填，且 `name` 必须与目录名一致：

```yaml
---
name: unit-test-designer
description: 设计与评审 Java/C# 单元测试，以变异得分作为测试充分性的主要指标
---
（原 SKILL.md 正文保持不变……）
```

可选：在 frontmatter 中加 `metadata: { opencode/slash: "true" }`，可将该技能暴露为 TUI 中的 `/unit-test-designer` 命令。详见 [opencode Agent Skills 文档](https://opencode.ai/docs/skills/)。

### 3. 验证与使用

```bash
opencode
```

- **agent**：主 agent（`mode: primary`）用 **Tab** 键切换；子 agent（`mode: subagent`）用 `@engineering` 调用。
- **skill**：模型会根据 `description` 在需要时自动加载；也可通过 `/` 命令手动触发（若已配置 `opencode/slash`）。

### 4. 全局接入（可选）

把文件放到全局目录，即可在所有项目中使用：

| 类型 | 项目级 | 全局 |
|---|---|---|
| agent | `.opencode/agents/<name>.md` | `~/.config/opencode/agents/<name>.md` |
| skill | `.opencode/skills/<name>/SKILL.md` | `~/.config/opencode/skills/<name>/SKILL.md` |

> Windows 下全局目录为 `C:\Users\<你>\.config\opencode\`。

---

## 贡献

新增领域或技能：在根目录新建（或扩展）`<domain>/` 目录，按上述约定放置 `AGENT.md` 与 `skills/<skill-name>/SKILL.md`，提交 PR 即可。内容要求：**理论有出处、规则可判定、流程可执行**。
