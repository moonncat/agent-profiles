# agent-profiles

[中文](./README.md)

A collection of domain-specific `AGENT.md` files and `skills` (software engineering, business data analysis, and more) for building AI Agents grounded in **professional domain theory**.

---

## Background

Generic LLM agents are usually given nothing but a one-line persona — "you are a domain expert". The resulting behavior has no stable foundation: there is no reviewable theoretical framework and no decidable standard of correctness, so output depends heavily on whatever the model improvises.

Real professional work, however, is theory-driven. Software engineering rests on entropy growth, closed-loop control, and test-sufficiency theory; business data analysis rests on causal inference, metric design, and experimental design. These theories only shape an agent's behavior once they are written down as **executable rules, gates, and workflows** — at which point the agent can be constrained, reviewed, and iterated like a professional.

This repository captures two kinds of artifacts per domain:

- **`AGENT.md`**: the system prompt of a domain agent — identity, principles, execution model, and hard constraints. It defines *how the agent works*.
- **`skills/`**: reusable, task-scoped skills that break domain theory into phased workflows and checklists. It defines *what the agent does when a specific task arrives*.

The files themselves are platform-agnostic and can be plugged into opencode, Claude Code, or any tool that supports custom agents and skills.

---

## Goals

1. **Accumulate domain assets**: capture `AGENT.md` and skills per domain, gradually covering software engineering, business data analysis, and beyond.
2. **Turn theory into rules**: no vague "expert persona" — domain theory must become decidable hard rules, thresholds, gates, and phased workflows.
3. **Platform-agnostic and portable**: the same file works across agent platforms; the only integration cost is adding a frontmatter block.
4. **Reviewable and evolvable**: everything lives as Markdown under version control, so it can be reviewed via PR and tracked over time.

---

## Repository Structure

Organized as "one directory per domain"; each domain holds that domain's agent definition and its skill set:

```
agent-profiles/
├── README.md                          # Chinese version
├── README.en.md                       # this document (English)
├── engineering/                       # domain: software engineering
│   ├── AGENT.md                       #   domain agent: engineering control agent
│   │                                  #   (entropy assessment → MAPE-K loop → entropy ledger → hard gates)
│   └── skills/                        #   skill set of this domain
│       └── unit-test-designer/        #     skill: unit test design
│           └── SKILL.md               #     (theory→rules mapping, phased workflow, stop criteria)
└── <domain>/                          # future domains (e.g. business-data-analysis)
    ├── AGENT.md                       #   agent definition of that domain
    └── skills/
        └── <skill-name>/SKILL.md      #   skill definitions of that domain
```

Conventions:

- `<domain>/AGENT.md`: the domain agent's system prompt, one per domain.
- `<domain>/skills/<skill-name>/SKILL.md`: a skill definition; the directory name is the skill name, one skill per directory.
- Adding a domain means adding a sibling directory; domains never interfere with each other.

---

## Usage (with opencode as an example)

opencode loads custom agents and skills from fixed locations, so using this repository is a matter of copying files into place. Both are placed under the project-level `.opencode/` directory (effective for the current project); for global setup see the end of this section.

### 0. Prerequisites

[opencode](https://opencode.ai/docs/) is installed, and your project is initialized (a `.opencode/` directory or `opencode.json` exists).

```bash
git clone https://github.com/moonncat/agent-profiles.git ~/agent-profiles
cd your-project
```

### 1. Install an AGENT.md

opencode reads markdown agents from `.opencode/agents/<name>.md`; **the file name becomes the agent name**.

```bash
mkdir -p .opencode/agents
cp ~/agent-profiles/engineering/AGENT.md .opencode/agents/engineering.md
```

`AGENT.md` is platform-agnostic body text, so add a **frontmatter block at the top** when installing it (`description` is required):

```yaml
---
description: Engineering control agent for software engineering, constraining code changes via entropy assessment and the MAPE-K loop
mode: all
---
(the original AGENT.md body stays unchanged...)
```

Optional fields: `mode` (`primary` / `subagent` / `all`), `model`, `temperature`, `permission`, etc. See the [opencode Agents docs](https://opencode.ai/docs/agents/).

### 2. Install skills

Skills go in `.opencode/skills/<skill-name>/SKILL.md`:

```bash
mkdir -p .opencode/skills
cp -r ~/agent-profiles/engineering/skills/* .opencode/skills/
```

Likewise, each `SKILL.md` must start with YAML frontmatter in which `name` and `description` are required, and `name` must match the directory name:

```yaml
---
name: unit-test-designer
description: Design and review Java/C# unit tests, using mutation score as the primary sufficiency metric
---
(the original SKILL.md body stays unchanged...)
```

Optional: add `metadata: { opencode/slash: "true" }` to the frontmatter to expose the skill as a `/unit-test-designer` command in the TUI. See the [opencode Agent Skills docs](https://opencode.ai/docs/skills/).

### 3. Verify and use

```bash
opencode
```

- **agent**: cycle primary agents with **Tab**; invoke a subagent via `@engineering`.
- **skill**: the model loads it on demand based on the `description`; you can also trigger it manually with a `/` command (if `opencode/slash` is configured).

### 4. Global installation (optional)

Put the files in the global directories to use them in every project:

| Type | Project-level | Global |
|---|---|---|
| agent | `.opencode/agents/<name>.md` | `~/.config/opencode/agents/<name>.md` |
| skill | `.opencode/skills/<name>/SKILL.md` | `~/.config/opencode/skills/<name>/SKILL.md` |

> On Windows, the global directory is `C:\Users\<you>\.config\opencode\`.

---

## Contributing

To add a domain or a skill: create (or extend) a `<domain>/` directory at the repository root and place `AGENT.md` and `skills/<skill-name>/SKILL.md` there following the conventions above, then open a PR. Content requirements: **theory with a source, decidable rules, executable workflows**.
