# 5 Agent Skill Design Patterns Every ADK Developer Should Know

> 原文：Google Cloud Tech
> 来源：https://skillpkg.com/posts/five-agent-skill-design-patterns-every-adk-developer-should-know

---

说到 `SKILL.md`，很多开发者往往会把注意力集中在格式本身上——比如 YAML 写对没有、目录怎么组织、是否符合规范。

但随着 30 多种 Agent 工具（例如 Claude Code、Gemini CLI、Cursor）都开始采用同一套布局，这个"格式问题"其实已经基本不是问题了。

**现在真正的难点，变成了内容设计。**

规范告诉你该如何把一个 skill 打包起来，却几乎没有告诉你：**skill 内部的逻辑应该如何组织**。

举例来说，一个封装 FastAPI 规范的 skill，与一个四步式文档生成流水线 skill，虽然外部的 `SKILL.md` 看起来几乎一模一样，但它们实际的工作方式却完全不同。

通过研究整个生态中 skill 的构建方式——从 Anthropic 的代码仓库，到 Vercel，再到 Google 的内部指导原则——可以发现，有五种反复出现的设计模式，能够帮助开发者更好地构建 agent。

本文会结合可运行的 ADK 代码，依次讲解这五种模式：

- **Tool Wrapper**：让你的 agent 立刻成为某个库或框架的专家
- **Generator**：基于可复用模板生成结构化文档
- **Reviewer**：按照检查清单并基于严重程度对代码进行评审
- **Inversion**：在动手之前，先由 agent 来"采访"你
- **Pipeline**：通过检查点强制执行严格的多步骤流程

---

## 模式 1：Tool Wrapper（工具封装器）

**工具封装器（Tool Wrapper）**的作用，是为你的 agent 提供某个特定库或框架的按需上下文。

与其把 API 约定硬编码进 system prompt，不如把这些知识封装成一个 skill。只有当 agent 真正处理相关技术时，才加载这些上下文。

这是最容易实现的一种模式。

`SKILL.md` 文件会监听用户提示词里是否出现特定库的关键词；一旦匹配，就动态加载 `references/` 目录中的内部文档，并把这些规则当作绝对准则来执行。

这正是你把团队内部编码规范、某个框架的最佳实践，直接分发到开发者工作流中的方式。

下面是一个 Tool Wrapper 的例子，它教 agent 如何编写 FastAPI 代码。注意，其中的指令明确要求：只有在开始审查代码或编写代码时，才去加载 `conventions.md` 文件。

```markdown
# skills/api-expert/SKILL.md

---

name: api-expert
description: FastAPI development best practices and conventions. Use when building, reviewing, or debugging FastAPI applications, REST APIs, or Pydantic models.
metadata:
  pattern: tool-wrapper
  domain: fastapi
---

你是 FastAPI 开发方面的专家。请将以下规范应用到用户的代码或问题中。

## 核心规范

加载 `references/conventions.md`，获取完整的 FastAPI 最佳实践清单。

## 审查代码时

1. 加载规范参考文件
2. 逐条对照规范检查用户的代码
3. 对于发现的每一项违规：
   - 引用对应的具体规则
   - 给出修改建议

## 编写代码时

1. 加载规范参考文件
2. 严格遵循其中的每一条规范
3. 为所有函数签名添加类型注解
4. 使用 `Annotated` 风格进行依赖注入
```

---

## 模式 2：Generator（生成器）

如果说 Tool Wrapper 解决的是"知识应用"问题，那么**生成器（Generator）** 解决的就是"输出一致性"问题。

假如你经常遇到 agent 每次生成的文档结构都不一样，那么 Generator 就能通过一种"按模板填空"的流程，把输出稳定下来。

它依赖两个可选目录：

- `assets/`：存放输出模板
- `references/`：存放风格指南

这里的指令相当于一个项目经理。它会告诉 agent：

1. 加载模板
2. 阅读风格指南
3. 向用户补齐缺失变量
4. 把内容填充进文档

这非常适合用于生成格式稳定的 API 文档、统一 commit message，或为项目快速搭建架构骨架。

在下面这个"技术报告生成器"的例子里，skill 文件本身并不直接包含文档版式或语法规则；它只是负责协调这些资产的加载，并强制 agent 按步骤执行。

```markdown
# skills/report-generator/SKILL.md

---

name: report-generator
description: 用于生成结构化的 Markdown 技术报告。当用户请求撰写、创建或起草报告、总结或分析类文档时使用。
metadata:
  pattern: generator
  output-format: markdown
---

你是一个技术报告生成器。请严格按照以下步骤执行：

步骤 1：加载 `references/style-guide.md`，获取语气和格式规范。
步骤 2：加载 `assets/report-template.md`，获取所需的输出结构。
步骤 3：向用户询问所有用于填充模板但目前缺失的信息，包括：
- 主题或议题
- 关键发现或数据要点
- 目标读者（技术型、高管型、普通读者）

步骤 4：按照风格指南的要求填充模板。模板中的每一个部分都必须出现在最终输出中。
步骤 5：将完成后的报告作为一个完整的 Markdown 文档返回。
```

---

## 模式 3：Reviewer（审查器）

**审查器（Reviewer）**模式，把"检查什么"与"如何检查"分离开来。

你不需要在 system prompt 里塞进一长串代码异味、风格问题、漏洞类型，而是可以把这些评审标准模块化地存放在 `references/review-checklist.md` 文件中。

当用户提交代码时，agent 就加载这份检查清单，按照其中的规则逐项打分，并按严重程度汇总结果。

如果你把一份 Python 风格检查表，替换成一份 OWASP 安全检查表，那么在完全相同的 skill 基础设施之上，就能立刻得到一种全新的、面向安全审计的专业评审能力。

这是一种非常高效的方式，既可以自动化 PR review，也能在人工介入前提前发现漏洞。

下面这个代码审查 skill 就体现了这种"检查标准外置"的思路：指令本身保持不变，但 agent 会动态加载外部 checklist 中的具体评审标准，并强制输出结构化、按严重程度分组的结果。

```markdown
# skills/code-reviewer/SKILL.md

---

name: code-reviewer
description: 审查 Python 代码的质量、风格以及常见缺陷。当用户提交代码请求审查、希望获得代码反馈，或需要进行代码审计时使用。
metadata:
  pattern: reviewer
  severity-levels: error,warning,info
---

你是一名 Python 代码审查员。请严格遵循以下评审流程：

步骤 1：加载 `references/review-checklist.md`，获取完整的评审标准。
步骤 2：认真阅读用户的代码，在提出批评之前先理解它的用途。
步骤 3：将检查清单中的每一条规则应用到代码中。对于发现的每一个问题：
- 记录行号（或大致位置）
- 标注严重程度：`error`（必须修复）、`warning`（建议修复）、`info`（可酌情优化）
- 解释**为什么**这是个问题，而不仅仅是指出**哪里**有问题
- 给出具体的修复建议，并附上修正后的代码

步骤 4：输出一份结构化评审结果，包含以下部分：
- **Summary**：代码做了什么，以及整体质量评价
- **Findings**：按严重程度分组列出问题（先 error，再 warning，最后 info）
- **Score**：按 1–10 打分，并简要说明理由
- **Top 3 Recommendations**：最值得优先改进的三项建议
```

---

## 模式 4：Inversion（反转模式）

Agent 天生倾向于"先猜再生成"，一拿到任务就想立刻开始输出。

而**反转（Inversion）** 模式，就是要把这个默认倾向反过来：**不再是用户给出一个 prompt、agent 马上执行，而是由 agent 先充当"采访者"**。

Inversion 依赖一类明确且不可协商的"门禁式指令"——比如"在所有阶段完成之前，禁止开始构建"。

这样就能强制 agent 先收集上下文，再开始行动。它会按顺序提问，并在进入下一阶段前等待你的回答；在它完整掌握你的需求与部署约束之前，它不会生成最终结果。

下面这个项目规划 skill 就展示了这种方式。关键点在于：阶段划分非常严格，而且提示语里有明确的门禁，阻止 agent 在还没收齐用户回答前就抢先生成最终方案。

```markdown
# skills/project-planner/SKILL.md

---

name: project-planner
description: 通过结构化提问收集需求，再为新的软件项目制定计划。当用户说"我想做一个……""帮我规划一下""设计一个系统"或"开启一个新项目"时使用。
metadata:
  pattern: inversion
  interaction: multi-turn
---

你正在进行一场结构化需求访谈。在所有阶段完成之前，**不要**开始构建或设计。

## 阶段 1 —— 问题发现
（一次只问一个问题，并等待用户回答）

请按顺序提出以下问题，不要跳过：

- Q1：这个项目要为用户解决什么问题？
- Q2：核心用户是谁？他们的技术水平如何？
- Q3：预期规模是多少？（例如：每日用户数、数据量、请求速率）

## 阶段 2 —— 技术约束
（仅在阶段 1 完整回答后进入）

- Q4：你计划部署在什么环境中？
- Q5：你对技术栈有没有明确要求或偏好？
- Q6：有哪些不可妥协的要求？（例如延迟、可用性、合规、预算）

## 阶段 3 —— 方案整合
（仅在所有问题都回答完毕后进入）

1. 加载 `assets/plan-template.md`，获取输出格式
2. 使用已收集到的需求信息填满模板中的每一个部分
3. 将完成后的计划呈现给用户
4. 询问："这份方案是否准确反映了你的需求？你希望调整哪些部分？"
5. 根据反馈继续迭代，直到用户确认
```

---

## 模式 5：Pipeline（流水线）

对于复杂任务来说，最怕的就是步骤被跳过、指令被忽视。

**流水线（Pipeline）** 模式，就是通过明确的检查点，强制执行严格的顺序化工作流。

在这种模式中，指令本身就充当了工作流定义。

通过设置显式的"菱形门禁条件"——比如要求用户先确认 docstring 生成结果，才能进入最终组装阶段——Pipeline 可以确保 agent 无法跳过复杂流程，直接给出一个未经验证的最终结果。

这种模式会用到所有可选目录：在不同步骤中，只在真正需要时加载对应的参考文件和模板，从而保持上下文窗口尽量干净。

下面这个文档流水线示例就非常典型。注意其中的门禁条件：在上一步生成的 docstring 没有得到用户确认之前，agent 被明确禁止进入文档组装阶段。

```markdown
# skills/doc-pipeline/SKILL.md

---

name: doc-pipeline
description: 通过多步骤流水线，从 Python 源代码生成 API 文档。当用户要求为模块编写文档、生成 API 文档，或根据代码创建文档时使用。
metadata:
  pattern: pipeline
  steps: "4"
---

你正在执行一条文档生成流水线。请按顺序执行每一步。不要跳过步骤；如果某一步失败，也不要继续往下执行。

## 步骤 1 —— 解析与清点

分析用户提供的 Python 代码，提取其中所有公开的类、函数和常量。

将提取结果以检查清单的形式呈现。

然后询问用户：

"这就是你希望写入文档的完整公开 API 吗？"

## 步骤 2 —— 生成 Docstring

对于每一个缺少 docstring 的函数：

- 加载 `references/docstring-style.md`，获取所要求的格式规范
- 严格按照风格指南生成 docstring
- 将每一条生成的 docstring 展示给用户审核

在用户确认之前，**不要**进入步骤 3。

## 步骤 3 —— 组装文档

加载 `assets/api-doc-template.md`，获取输出结构。

将所有类、函数和 docstring 汇编成一份完整的 API 参考文档。

## 步骤 4 —— 质量检查

对照 `references/quality-checklist.md` 进行审查，检查以下内容：

- 每一个公开符号都已被记录
- 每一个参数都有类型和说明
- 每一个函数至少包含一个使用示例

汇报检查结果。
在向用户展示最终文档之前，先修复发现的问题。
```

---

## 如何选择正确的模式？

每一种模式，回答的其实都是不同的问题。

你可以用下面这棵"决策树"来判断，哪一种最适合你的使用场景：

- 你的 skill 主要解决什么问题？
  - 知识应用？ → **Tool Wrapper**
  - 输出一致性？ → **Generator**
  - 检查标准化？ → **Reviewer**
  - 需求理解？ → **Inversion**
  - 流程控制？ → **Pipeline**

---

## 模式可以组合

这些模式并不是彼此排斥的。恰恰相反，它们是可以自由组合的。

- **Pipeline + Reviewer**：流水线末尾加审查步骤，对自己的结果再做一次检查
- **Inversion + Generator**：先套用 Inversion，通过提问收集必要变量，再进入模板填充流程

借助 ADK 的 `SkillToolset` 和渐进式披露（progressive disclosure）机制，你的 agent 只会在运行时，为当前真正需要的模式消耗上下文 token。

---

**核心结论：**

> 不要再试图把复杂、脆弱、难维护的指令一股脑塞进一个 system prompt 里。
> 把工作流拆开，给不同问题匹配合适的结构化模式，才能构建出更可靠的 agent。

---

原文："[5 Agent Skill design patterns every ADK developer should know](https://x.com/googlecloudtech/status/2033953579824758855)", Google Cloud Tech
