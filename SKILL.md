---
name: neat-freak
description: >
  End-of-session knowledge cleanup with OCD-level rigor — reconciles project docs
  (CLAUDE.md, README.md, docs/) and agent memory against the code so nothing rots.
  会话结束后对项目文档和记忆进行洁癖级审查与同步。MUST trigger when the user says:
  "sync up", "tidy up docs", "update memory", "clean up docs", "/sync", "/neat", "同步一下",
  "整理文档", "整理一下", "更新记忆", "梳理一下", "收尾", "定稿", "复盘", "这个阶段做完了",
  "新人能直接上手", or any phrase suggesting a dev milestone where knowledge needs
  reconciliation. "定稿" scenario = finish the user's primary task first, then append this
  skill's ritual (never skip the primary task). Also trigger when the user reports stale docs,
  conflicting memories, or wants a clean handoff to teammates or other agents. Bare "整理" /
  "tidy" with prior dev context counts — do not under-trigger. Cross-platform: works on
  Claude Code, OpenAI Codex, OpenCode, and OpenClaw.
---

# 洁癖 — Knowledge Base Neat-Freak

> **Cross-platform Agent Skill** — Claude Code · OpenAI Codex · OpenCode · OpenClaw 通用。
> 跨平台 SKILL.md，遵循开放 Agent Skill 规范。

你是一个**知识库编辑**，不是记录员。记录员只会往后追加，编辑会审查全局、合并重复、修正过期、删除废弃。你的工作是让整个项目的知识体系始终保持**干净、准确、对新人友好**的状态——像有洁癖一样。

## 为什么这件事重要

在 AI 协作开发中，代码可以随时重写，但**文档和记忆是跨会话、跨 Agent 的唯一桥梁**。如果记忆里有过期信息，下一个 Agent（无论它是 Claude、Codex 还是别的）会基于错误前提做决策。如果 docs/ 混乱或缺失，接手者（尤其是下游项目的同事）会浪费大量时间搞清楚这套系统怎么用。

这个 Skill 的价值就在于：**让知识体系的每一层都跟得上代码的变化。**

## 关键概念：三类知识，三种受众

**必须先理解这件事，否则你会只改 CLAUDE.md 就结束，把下游同事和其他 agent 晾在那儿。**

| 位置 | 受众 | 职责 | 不同步的代价 |
|------|------|------|--------------|
| **Agent 记忆系统**（若 agent 支持） | Agent 自己跨会话复用 | 个人偏好、非显而易见的项目事实、跨项目 reference | 下次会话 Agent 忘记历史决策 |
| 项目根 `CLAUDE.md` / `AGENTS.md` | 当前项目里的 AI（下次会话自己） | 项目约定、结构、红线、环境变量、路由清单 | 下次 AI 在这个项目里走弯路 |
| 项目 `docs/` + `README.md` | **其他人**（人类同事、下游开发者、未来接手的 AI） | 接入指南、架构图、运维手册、交接说明、API 参考 | **其他人或系统无法正确接入或运维** |

这三层**受众不同，职责不重叠**。CLAUDE.md 里写"新增了 device flow 五个路由" ≠ docs/integration-guide.md 里"下游怎么接这套 flow" —— 前者是提醒自己，后者是教别人。**两份都要写。**

> **Agent 记忆系统的具体位置因平台而异**（Claude Code 在 `~/.claude/projects/<...>/memory/`，Codex 用 `AGENTS.md`，OpenCode 用 `.opencode/`，OpenClaw 用 `~/.openclaw/`）。完整路径速查见 [references/agent-paths.md](references/agent-paths.md)。如果当前 agent 没有独立的记忆系统，直接跳过这一层，把功夫全花在 docs 和项目根 markdown 上。

### CLAUDE.md / AGENTS.md 是规则手册，不是变更日志（重要）

最常见的 skill 翻车模式：每次开发完都在 CLAUDE.md 顶部加一段 blockquote 历史叙事——"2026-05-08 X 功能上线，详见 docs/Y.md"。一次很爽，半年后顶部就是 200 行 blockquote 把真正的规则推到看不见。**这种叙事不属于 CLAUDE.md**，它的归宿是 git log / `/changelog` 页 / `docs/CHANGES.md`。

判断一条信息该不该进 CLAUDE.md，问一句：**下次 AI 写代码时如果没看到这条，会不会犯错？**

| 例子 | 进 CLAUDE.md？ | 理由 |
|---|---|---|
| "Prisma 查询只写在 `modules/**/data/`" | ✅ | 违反就是边界破坏，AI 必须看到 |
| "rsync 单文件部署必须用完整 target 路径" | ✅ | 踩坑警示，会再次踩 |
| "禁止裸跑 systemctl stop my-worker" | ✅ | 红线，事故级 |
| "2026-05-08 timelineAt 上线，详见 docs/ARCHITECTURE.md §5.4" | ❌ | 详细机制在 docs；AI 改到这块自然会读 docs；「深入文档」指针表已做这件事 |
| "2026-04-30 起公网开放，匿名可访 /、/all" | ❌ | 既是历史也是事实，但事实归 docs/ARCHITECTURE.md §8 + 项目概览一句话足矣 |
| "5/8 修了 X bug 的复盘细节" | ❌ | 单次事故记忆，归 memory 或干脆删 |

✅ 该进 CLAUDE.md 的内容：硬边界规则、禁止事项、命令速查、权限模型、协作流程、深入文档指针表、踩坑警示。
❌ 不该进的：历史叙事（"X 时刻起 Y 上线"）、详细机制说明、单次事故复盘、bug fix 流水账、"详见 docs/Z.md" 的指针句子（这个角色已经被「深入文档」指针表占掉了）。

## 执行流程

### 第零步：尺寸体检（防膨胀）

任何同步动作之前，先 `wc -l` 关键文件：

| 文件 | Soft limit | 超过怎么办 |
|---|---|---|
| `CLAUDE.md` / `AGENTS.md` | ~300 行 / ~15KB | 先做精简：扫顶部 blockquote / 历史叙事段 → 删 / 迁 docs；项目概览只留 1-3 行 + 关键速查表，不要做"提醒下次会话"用 |
| 记忆索引（如 `MEMORY.md`） | ~150 行 | 找已被新版本取代的、单次事故复盘、详细机制可读代码代替的 → 删 |
| 单条 memory 文件 | ~100 行 | 通常说明在塞多件事 / 写成事故复盘 → 拆成几条独立记忆，或者直接删（很多事故复盘没复用价值） |
| `docs/<single>.md` | ~1500 行 | 切分成多文件，加目录索引 |

**超尺寸是这个 skill 的最高优先级，大于"补本次会话漏掉的同步"。** 原因：超尺寸的 CLAUDE.md 实际上让下次 AI 看不到真正重要的规则（被叙事段挤到 200 行外，进不了 prompt 重点段），同步再补也事倍功半。

**执行顺序**：先精简（破除膨胀）→ 再做本次会话增量同步（补漏）。两件事不能合并——精简时心态是"什么不该在这"，补漏时心态是"什么该补到这"，混着做会两头不到位。

### 第一步：盘点现状（强制机械式枚举，不能跳过）

**先做 ls，再做判断。**

1. 列出 agent 的记忆文件（如有）：
   - Claude Code：`ls ~/.claude/projects/<...>/memory/` 并读 `MEMORY.md` 及所有被引用的 `.md`
   - Codex / OpenCode / 其他：找该 agent 的等价位置（见 references/agent-paths.md）
2. 对本次对话涉及的**每一个**项目：
   - `ls <project-root>/` → 确认根目录结构
   - `ls <project-root>/docs/ 2>/dev/null` → **枚举所有 docs**（缺失也要确认）
   - `find <project-root> -maxdepth 2 -name "*.md" -not -path "*/node_modules/*" -not -path "*/.git/*"` → 兜底抓散落的 .md
   - 读 `README.md`、`CLAUDE.md` / `AGENTS.md`、每一个 `docs/*.md`
3. 读全局 agent 配置（若有，如 `~/.claude/CLAUDE.md`、`~/.codex/AGENTS.md`）
4. 回顾本次对话全部内容
5. **【secret 扫描·可选】** 若你的环境装有明文密钥清扫工具，顺手扫一遍当前会话记录与项目 memory：有命中 → 报告用户（类型 / 脱敏值 / 轮换入口），用户同意后再清理（先备份，确认无误后**删掉备份**，否则明文仍在）。没有这类工具就跳过，不阻塞收尾。

**输出一张文件清单**（内部用，不用给用户看），对每个文件标：「评估过 / 要改 / 不用改」。**漏一个不行**——这是这个 skill 最容易翻车的地方。

### 第二步：识别变更——用"变更影响矩阵"思考

**不要只看对话增量有什么新事实，要看新事实会波及哪些文档层级。**

常见模式速览：
- 新增 API / 路由 → CLAUDE.md 路由清单 + integration-guide + architecture 的 Routes
- 新增 / 改名 环境变量 → CLAUDE.md 环境变量表 + runbook + 下游 integration-guide
- 新增数据库表 → CLAUDE.md + architecture 的 Data Model
- 新增大特性（跨多文件） → 以上全部 + architecture 新章节 + handoff 已完成清单
- 跨项目改动 → 上下游两边的 docs **都要对齐**（最常见的漏改场景）
- 记忆层面：相对时间→绝对日期、过期事实→改、重复→合并、已完成待办→删

完整映射表（覆盖更多变更类型与对应文档）见 **[references/sync-matrix.md](references/sync-matrix.md)**——遇到不确定的改动先查这张表。

**关键检查**：这次对话是不是**跨项目**的？如果改了项目 A 且项目 B 依赖它（通过 SDK、API、子域、环境变量），**项目 B 的 docs 也要改**。这是我们实操里最常见的翻车点。

### 第三步：实际修改（用工具，不只是描述）

你必须**真的用 Edit 修改现有文件、用 Write 创建新文件、用删除命令清理废弃文件**。"我会怎么改"的描述不算完成。

**顺序建议**：先改 docs/（改错影响外部）→ 再改 CLAUDE.md/AGENTS.md → 最后理记忆。先动外部优先级最高的，即使中途被打断，读者看到的也是对齐的最新状态。

**编辑原则**：

- **减优于加**（最重要）：每次同步动作结束后，CLAUDE.md / AGENTS.md 净涨幅 > 30 行就是红灯——很可能在写历史叙事而不是补规则。回头审：这条加的是"下次 AI 写代码时必须看到"的规则，还是"上次会话告诉下次会话发生了什么"的便条？后者就是病。能删的先删，不能删的迁去 docs，最后剩下的才是规则。
- **合并优于追加**：新信息是对旧信息的更新，改旧条目；新加条目前先 grep 同关键字，看现有条目能不能并
- **删除优于保留**：完成的临时计划、推翻的决策、已被新版本取代的项目记忆、单次事故的流水账复盘——删
- **精确优于冗长**：一条记忆说清楚一件事，别塞三件
- **绝对时间**：永远 `2026-04-29`，不写"今天"、"最近"
- **面向读者**：docs/ 的读者是"第一次接触这个项目的外部人"，写的时候想象对方只有 5 分钟能看完
- **受众不混**：CLAUDE.md 里不抄 docs/ 的全文，docs/ 里不写"我记得上次……"——这是记忆的事
- **指针不重复**：同一条事实如果 docs/ 里已详写，CLAUDE.md 只在「深入文档」指针表里出现一次，不在概览段再叙事一次。「深入文档」指针表 = CLAUDE.md 内固定的『文档导航』小节（每行：文件 → 一句话职责），所有「详见 docs/X」只在表里出现一次；没有就先建表再迁移指针

**全局配置极度克制**：`~/.claude/CLAUDE.md` / `~/.codex/AGENTS.md` 只有用户在对话中明确表达了**跨项目的核心原则**才动。日常项目细节绝不进全局。

**docs/ 编辑要点**——新增一个能力的文档变更通常要四处都补：
1. **integration-guide** 或对应"外部视角"文档：加**怎么用**（curl / SDK 示例 / 错误码表）
2. **architecture**：加**怎么工作**（数据流、状态机、设计取舍）
3. **runbook**：加**怎么运维**（冒烟命令、故障排查、环境变量）
4. **handoff** 或 CHANGELOG：加**已完成**

API 速查表、环境变量表、术语表是高频查询的结构化信息，**必须保持"所见即最新"**。

### 第四步：自检清单（必须逐项过一遍）

这一步同时防止"漏改 docs" + "误把叙事塞进 CLAUDE.md"。改完后逐条检查：

**尺寸 / 反膨胀（先查这组，不达标的话回头先精简）**：
- [ ] CLAUDE.md / AGENTS.md 净涨幅 ≤ 30 行（超了先怀疑塞了历史叙事，回头审）
- [ ] 没新增 "X 起 Y 上线，详见 docs/Z.md" 这种 blockquote 历史叙事条目
- [ ] 没在 CLAUDE.md 里抄 docs/ 已有的详细机制说明
- [ ] 单条 memory 文件没超 ~100 行（超了拆 / 删 / 改成 reference）

**完整性 / 反漏改（再查这组）**：
- [ ] 第一步列出的每个文件，都判断了"不用改"或"已改"
- [ ] 记忆索引（若有）里的每个链接指向存在的文件
- [ ] 每个记忆文件的 description 和内容对得上
- [ ] 记忆之间没有互相矛盾
- [ ] CLAUDE.md / AGENTS.md 里提到的路径 / 命令 / 工具 / 环境变量在代码中真实存在
- [ ] README 的安装 / 运行步骤跟代码一致
- [ ] 新增 API 路由：**在 integration-guide 和 architecture 都出现了**
- [ ] 新增环境变量：**在 runbook 和项目根 markdown 都出现了**
- [ ] 新增数据库表：**在 architecture 的 Data Model 和项目根 markdown 都出现了**
- [ ] 跨项目影响：下游项目的 docs 也跟着改了
- [ ] 没有相对时间遗留（`grep -E "今天|昨天|刚刚|最近|上周|today|yesterday|recently"` 清零）

哪条打不了勾，**回去补**。不要因为"差不多了"就跳过这一步——这是这个 skill 的灵魂。

### 第五步：变更摘要

在所有文件修改完之后（不是之前），给用户简洁摘要：

```
## 同步完成

### 记忆变更
- 更新：xxx（原因）
- 新增：xxx
- 删除：xxx（原因）

### 文档变更（按项目分组，每个项目列全改动的文件）
- <项目 A>/CLAUDE.md — xxx
- <项目 A>/docs/integration-guide.md — xxx
- <项目 A>/docs/architecture.md — xxx
- <项目 B>/docs/<integration>.md — xxx

### 未处理
- xxx（为什么没处理，比如需要用户确认）
```

只列有实际变更的条目。没改的不写。

### 第六步：经验沉淀与打分（对话复盘 · 防"价值蒸发"）

文件同步只保证"文档不过期"；本步保证"经验不蒸发"。审查自身不产生复利，复盘才产生。两个动作：

**A. 经验沉淀**

1. **复发问题**：本次对话遇到的问题，检索记忆目录与项目知识库找同主题历史记录。命中 = 复发 → 把「问题 + 本次有效解法」写入 memory（通用教训）；已有同类记录则**更新旧条目而非新增**。未命中但值得记的新坑 → 新增一条。
2. **高频动作（≥3 次门槛）**：某操作模式跨会话累计出现 ≥3 次（本次会话 + 历史留痕合并计数）→ **必须询问用户是否 skill 化**，不得擅自创建。用户同意后：创建 skill（按你的项目规则确定存放位置）→ 若项目有能力分工 / 角色体系，挂载到对应角色，把能力与整改要点一并写进去，使其成为体系的一部分。
3. **留痕**：本次识别到但未达 3 次门槛的动作模式，在 memory 留一行记录，供下次累计。没有留痕，跨会话计数就是空谈。

**B. 打分板写入**（可选：项目若有任务打分 / 复盘机制则执行，无则跳过）

往项目的复盘记录（数据库 / 日志表皆可）写一行：任务一句话描述 + **客观证据**（一次成 / 返工 N 次、测试或压测通过与否、审查发现的问题数——客观信号优先，LLM 自评易失真）+ 综合分 1~5（AI 自评仅作初审）。

- **用户快评（收尾必问）**：收尾时问一句「本次 1~5 打几分？」——用户分数覆盖 AI 自评分，以用户为准。
- **述职触发**：累计 ≥20 条 → 主动提议跑一轮统计复盘（哪类任务返工多 / 哪类任务分低 / 哪类问题重复出现 → 整改建议回流到规则与 skill）。只记录不回流不叫迭代。

**触发方式**：`/neat-freak` 命令；或会话中出现收尾意图关键词（「收尾」「定稿」「复盘」「总结一下」）且当前任务已完结时执行本 skill。「定稿」场景 = 先完成用户主任务的定稿动作，**再追加本仪式**，不跳过主任务。

## 特殊情况

**项目还没有 README 或 CLAUDE.md/AGENTS.md**：判断项目是不是到了"有可运行代码"的阶段。是 → 创建。还在 vibe 阶段 → 跳过，但在摘要里提一句。

**对话没有产生新事实**：审查现有记忆和文档有没有过期 / 冲突 / 相对时间——审查本身就有价值。

**记忆之间出现无法自动判断的矛盾**：列在「未处理」让用户决定。常规修改自己拍板，只有这几处回来找用户：①记忆矛盾裁决 ②skill 化同意（见第六步）③敏感信息清理同意（见第一步）④收尾打分（见第六步）。

**跨项目改动**：本次对话改了多个项目，每个项目都要跑一次完整的第一步（ls + 读 docs）。不要假设一个项目的 docs 改了，另一个就不用。尤其是上游-下游对接文档（集成指南 / SDK 说明 / API 协议），两边都要对齐。

**发现之前的同步漏了东西**：修掉。不要说"那不是这次对话的事"——你就是这个项目的持续编辑，过去的漏洞也归你管。

**用户只要求某一层**（如"更新记忆"、"看看新人能不能上手"）：以该层为重点，但流程不裁剪——第一步盘点、第三步修改、第四步自检、第五步摘要照走，只是修改与自检的重心放在指定层。

## 参考资料

- **[references/sync-matrix.md](references/sync-matrix.md)** — 完整的"变更类型 → 要改哪些文件"映射表
- **[references/agent-paths.md](references/agent-paths.md)** — Claude Code / Codex / OpenCode / OpenClaw 各自的记忆与配置路径速查
