# 权限 ruleset 模板（会话级，创建时冻结）

> 作用域：随 `POST /session` 的 permission 字段冻结进会话——改规则只对新会话生效，不热更。
> 哲学：白名单放行 + 危险兜底 ask；**宁松勿拒**（一次 reject 报废 executor 整个回合，比误放代价高一个量级）。

## 1. 静态白名单（按层，模板唯一要动手的是层 2 命令族）

```text
# 层 1：只读探查全放行
read, grep, list, glob = allow
bash: cat|ls|find|jq|head|tail|wc = allow
bash: git status|diff|log|show|blame|fetch = allow

# 层 2：git 写操作（feature 分支）+ 测试/构建命令族（按项目栈补全）
bash: git add|commit|checkout|switch|branch|merge|cherry-pick = allow
bash: <栈的测试构建族> = allow
  JS 栈示例:   npm test|run build|run lint|run typecheck, npx tsc|vitest|eslint|playwright test
  .NET 栈示例: dotnet restore|build|test
  Java 栈示例: mvn test|compile|package|verify
  Python 栈示例: pytest, python -m unittest
bash: <发评脚本> * = allow          # 工单评论通道，必须显式放行

# 层 3：其余 bash 一律 ask；三条防写盘规则
bash: * > *   = ask
bash: * >> *  = ask
bash: * -exec * = ask
```

## 2. deny-list（任何自动审批层都不得放行，直接转人工）

```text
删文件（rm 族）、sudo、git push 到受保护分支、git reset --hard、chmod、
curl|wget 外传或管道进 shell、重定向进敏感路径、
读写凭据文件（.env / credentials / .ssh 族）、系统配置写入、部署/线上操作
```

## 3. 残留 ask 的自动审批策略

- 常态零 LLM；有 ask 才调小模型按 rubric 判（rubric 见 `script-contracts.md` §2）；
- 任何失败（API 挂 / 解析失败）→ 保持 pending 下轮重试，**绝不默认放行**；
- 判定提示词首句写明 reject 语义：终结整个回合，只拒真正危险。

## 4. 减少 ask 的治本手段（比审批更快）

- **干净 serve 根**：只含目标仓符号链接的目录当工作根，`ls` 一眼见底——探索类 ask 的大头源于找目录/巡场；
- **命令族写全**：每引入一个新工具链就补一族 allow，否则每个新命令都是一次 ask 往返；
- **发评通道预置**：没给合法发评途径，executor 会翻环境找凭据 → 被拒 → 回合报废；
- **防写盘三规则不可省**：ask 化的重定向/exec 是工作区外写入的最后一道闸。

## 5. 变更纪律

- ruleset 改动只影响新会话；旧会话带着创建时的快照（桥接层可按 rulesetHash 自动迁移，无桥接时等会话自然结束）；
- 每次改 ruleset 后用一个探针会话验证：让模型跑一条新增 allow 的命令，不再弹 ask 才算生效。
