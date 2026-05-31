# oh-my-openagent 学习地图总结

> 源文件：`oh_my_openagent_learning_v3.html`  
> 面向 OpenCode 的多代理研发编排插件

---

## 一、核心判断

**oh-my-openagent 不是一个独立 IDE，也不是单纯的提示词包。** 它是运行在 OpenCode 上的多模型、多代理编排 harness。OpenCode 提供终端/IDE/桌面里的代码执行环境、LSP、多会话等基础能力；oh-my-openagent 借助插件机制扩展出代理、任务路由、后台任务、技能、MCP、命令和 hooks。

| 对象 | 真实含义 | 理解方式 |
|------|----------|----------|
| OpenCode | 开源 AI coding agent | 执行环境，负责让 AI 看代码、改代码、跑命令、接 LSP |
| oh-my-openagent | 多模型 agent orchestration harness | 编排层，把一个 coding agent 拆成一个有岗位分工的研发小队 |
| oh-my-openagent.json | 插件配置文件 | 规则表，控制模型、代理、category、并发、权限、fallback、skills |
| Agent | 岗位化角色 | 不是越多越好，每个代理必须有清晰边界 |
| Category | 任务类型路由 | 代理委派任务时不直接选模型，而是选任务类型 |
| Skill | 专业上下文 + 工具注入 | 给代理临时加载某个场景的 SOP、MCP、操作方法 |

> 学习重点：不要放在"有哪些神话代理"。重点放在"一个研发任务从需求到代码提交，中间每个节点谁负责、谁不能越权、用什么证据判断完成"。

---

## 二、系统模型

本质是把单一 Agent 的长任务压力拆掉：计划交给计划者，执行交给执行者，审查交给审查者，搜索交给搜索者。

### 四个关键环节

| 环节 | 说明 |
|------|------|
| **输入** | 来自用户、Git Issue、Bug、需求文档、UI 截图、测试失败日志 |
| **判断** | 简单任务直接做；复杂任务走 `ulw`；精确任务走 `@plan → /start-work` |
| **路由** | Sisyphus 或 Atlas 把不同任务派给合适的代理或 category |
| **闭环** | 运行测试、检查实现、拆分提交、记录失败归因和项目上下文 |

### 一句话模型

```
用户目标 → 任务判断 → 计划拆解 → 计划审查 → 分工执行 → 测试验证 → 原子提交 → 经验沉淀
```

---

## 三、三层架构

| 层级 | 核心角色 | 解决的问题 | 失败时的典型表现 |
|------|----------|------------|------------------|
| **计划层** | Prometheus / Metis / Momus | 需求不清、边界不清、验收不清 | Agent 自己脑补业务逻辑，改了一堆不该改的文件 |
| **执行层** | Atlas | 任务拆分、并行调度、状态连续性 | 多任务互相踩踏，前一个结论没传给后一个执行者 |
| **工人层** | Junior / Oracle / Explore / Librarian / Looker | 具体编码、代码库侦察、文档检索、架构咨询、视觉分析 | 高级模型被浪费在 grep；低级模型被迫做架构判断 |

---

## 四、代理分工

代理不是"越多越智能"。代理的价值来自边界：谁负责判断，谁负责执行，谁只读，谁不能乱改。

### 十大代理速查

| 代理 | 角色 | 适合场景 | 不要做的事 |
|------|------|----------|------------|
| **Sisyphus** | 主协调者 / 默认入口 | 大部分中等复杂任务 | 让它无限自由地重构全项目 |
| **Hephaestus** | 深度自主工程师 | 难以定位的 bug、复杂技术方案 | 拿它做简单 grep 或文案修补 |
| **Prometheus** | 战略计划者 (Plan Builder) | Git Issue 转执行计划、需求澄清、边界定义 | 让它直接写业务代码 |
| **Atlas** | 执行调度器 (Plan Executor) | `/start-work` 后的计划执行 | 让它重新发明计划 |
| **Sisyphus-Junior** | 具体任务执行者 | 单个明确子任务、局部改动 | 让它做架构裁决 |
| **Oracle** | 只读架构顾问 | 计划评审、方案取舍、复杂问题诊断 | 让它改文件 |
| **Explore** | 代码库侦察员 | repo 初探、定位函数、找调用链 | 让它写复杂实现 |
| **Librarian** | 资料研究员 | 技术栈文档、第三方库用法 | 用它代替本地代码理解 |
| **Metis** | Plan Consultant（缺口分析顾问） | 计划定稿前找遗漏、歧义、隐藏前提 | 把它当成最终审批者 |
| **Momus** | Plan Critic（严格计划批评者） | 高风险任务开工前的最后审查 | 让它替 Prometheus 写完整计划 |

---

## 五、Plan 系列代理详解

Prometheus → Metis → Momus → Atlas 是一条完整的计划链路，不是同一种东西。

### 四者区别

| 代理 | 角色 | 主要问题 | 使用时机 | 错误用法 |
|------|------|----------|----------|----------|
| Prometheus | Plan Builder | 这件事到底要怎么做？边界是什么？ | 复杂任务动代码前 | 让它直接写业务代码 |
| Metis | Plan Consultant | 这个计划漏了什么？哪里会导致 AI 自作主张？ | Prometheus 计划初稿出来后 | 把它当最终审批 |
| Momus | Plan Critic | 计划是否足够清晰、可执行、可验证？ | 计划准备进入 `/start-work` 前 | 没有计划就直接找 Momus |
| Atlas | Plan Executor | 如何按计划拆分任务、调度 worker？ | `/start-work` 后 | 用它消化模糊需求 |

### 完整计划链路

```
@plan / Prometheus → Metis 找缺口 → 修订计划 → Momus 严格审查 → /start-work / Atlas → Verification
```

### Metis vs Momus 核心差异

- **Metis**（计划还没定稿时用）：偏顾问，偏补洞，偏前置发现问题——暴露"你以为清楚、其实不清楚"的地方
- **Momus**（计划准备执行前用）：偏审判，偏 gate，偏判断是否能交给执行系统——挡住烂计划

> 只要任务涉及多文件、多模块、权限、数据结构、支付、账务、架构或重构，就应该让它们做计划 gate。

---

## 六、oh-my-openagent.json 配置

可以把配置文件理解为"多代理研发系统的控制台"。

| 配置区域 | 负责内容 | 什么时候需要改 |
|----------|----------|----------------|
| `agents` | 为具体代理设置模型、权限、prompt、fallback | 某个代理效果长期不稳定 |
| `categories` | 定义任务类型对应的模型、温度、工具、附加提示词 | 需要针对 UI、深推理、快修做不同路由 |
| `background_task` | 控制后台任务并发、provider 并发、model 并发 | 出现限流、费用失控、并行任务抢资源 |
| `sisyphus_agent` | 控制主编排系统、Prometheus planner | 想恢复原生 OpenCode 或调整计划代理行为 |
| `skills` | 配置内置/自定义 skills | 要加入 git、playwright、设计走查等专业 SOP |
| `team_mode` | 实验性并行多代理团队模式 | 普通编排跑稳后再考虑，默认不该打开 |

> 建议：先用默认配置跑固定任务集，记录失败类型，再改模型、权限、category。没有基线就调配置，本质是在蒙。

---

## 七、三种使用模式

### 模式一：简单任务（直接 prompt）

适合拼写修复、单文件小改、明确 bug、小范围样式。不需要探索大量上下文，不需要拆计划。

### 模式二：复杂但懒得解释（ulw / ultrawork）

适合你知道目标但不想详细解释项目结构。风险是更自主，适合实验但不适合高风险业务逻辑。

### 模式三：复杂且要求可控（@plan → /start-work）

适合 Git Issue、跨文件功能、架构改造、支付/权限/数据迁移等高风险任务。**这是自驾代码库最应该采用的主流程。**

```
@plan "Implement issue #42. Inspect the codebase first..."
/start-work
```

---

## 八、命令体系

| 命令 | 核心作用 | 适合场景 | 主要风险 |
|------|----------|----------|----------|
| `ulw ...` | ultrawork 模式 | 复杂但边界清楚的修复、探索 | 可能扩大范围 |
| `/ralph-loop "..."` | 自引用开发循环 | 长任务、需要自动续跑 | 完成检测依赖机制 |
| `/ulw-loop "..."` | ultrawork + loop | 大型、多步骤自动执行 | 最容易失控 |
| `/cancel-ralph` | 取消 Ralph Loop | 发现 loop 方向错了 | 只能止损 |
| `/stop-continuation` | 停止续跑机制 | agent 一直续跑、不停重复 | 可能中断未完成任务 |
| `/init-deep` | 生成分层 AGENTS.md | 老项目、复杂项目 | 不维护会过期 |
| `/refactor` | LSP + AST-grep 重构 | 命名迁移、模块重整 | 不适合无测试保护的核心逻辑 |
| `/handoff` | 生成上下文交接文档 | 换会话时交接 | 没真实验证时只是漂亮总结 |

### /ulw-loop 使用原则

- `ulw` 是"这次回答进入 ultrawork 模式"，仍然是一次任务请求
- `/ulw-loop` 是"不要停，直到任务完成"，会在 ultrawork 模式下持续推进
- `/ulw-loop` 适合"已经定义清楚目标、边界、验收标准、最大轮数、验证命令"的任务
- **不要用** `/ulw-loop` 的场景：需求还没想清、没有测试保护、高风险核心链路

---

## 九、Category 与 Skill

这两个概念最容易混淆。

| 概念 | 决定什么 | 例子 | 错误用法 |
|------|----------|------|----------|
| **Category** | 模型、温度、思维方式、工具开关 | `visual-engineering`、`deep`、`quick`、`writing` | 把所有任务都塞进 `deep` |
| **Skill** | 专业上下文、流程、MCP、操作规范 | `git-master`、`playwright`、`review-work` | 做一个万能大 skill 塞进所有规则 |

### 内置 Category

- **visual-engineering**：前端 / UI / 样式 / 动效
- **ultrabrain**：极深推理（架构、高风险技术判断）
- **deep**：深度问题解决（一个清晰目标、一个明确交付物）
- **quick**：小任务快修（单文件、低风险）
- **writing**：文档写作
- **unspecified-high / low**：兜底任务

### 面向自驾代码库的 Skill 建议

```
issue-intake           # Git Issue 转任务输入
plan-review            # 检查计划是否可执行
implementation-boundary # 限制改动范围
verification           # 定义 build/test/lint/手工 QA
failure-analysis       # 失败归因
context-update         # 经验沉淀进项目 harness
handoff                # 交付总结
```

---

## 十、推荐工作流（Git Issue 驱动）

```
Issue Intake → @plan → Plan Gate (Metis/Momus/Oracle) → /start-work → Review → Commit
```

### 判断公式

```
任务越复杂，越不能直接执行。
任务越高风险，越要先写验收标准。
代码库越老，越要先探索现有模式。
上下文越多，越要把计划和验证写进文件。
Agent 越自主，权限边界越要硬。
```

---

## 十一、实战提示词模板

### Git Issue 转计划
```
@plan "Analyze GitHub issue #XX and create an executable implementation plan.
1. Inspect the existing codebase before proposing changes.
2. Identify affected files and existing conventions.
3. Define clear acceptance criteria + verification commands.
4. Explicitly list MUST DO and MUST NOT DO constraints.
5. Do not edit code yet."
```

### 执行计划
```
/start-work
```

### 完成后审查
```
Use review-work to verify the implementation against the original issue,
code quality, security, hands-on QA, and hidden context problems.
```

### 原子提交
```
Use git-master to split changes into atomic commits.
Each commit must have a clear purpose.
```

---

## 十二、常见六大误区

| 误区 | 说明 |
|------|------|
| 所有任务都用 ultrawork | 等于把计划、判断、执行、验收全交给一个黑盒 |
| 把所有代理都配最强模型 | Explore/Librarian 做搜索不需要最强模型 |
| 没有验收标准就开干 | Agent 会用"看起来完成"代替"真的完成" |
| 只写提示词不做权限约束 | 边界要靠系统配置，不靠自觉 |
| 计划评审只是走形式 | Metis/Momus 的意义是挡住烂计划 |
| 跳过失败归因 | 失败不沉淀，下一次还会失败 |

---

## 十三、验证清单

1. **安装验证** — 运行 `bunx oh-my-openagent doctor`
2. **代理可见性** — 确认核心代理在 OpenCode 里可调用
3. **计划质量** — 计划是否包含目标、范围、文件、验收、验证
4. **计划 Gate** — Metis/Momus/Oracle 是否能发现模糊点
5. **执行边界** — 是否按计划执行，不扩大范围
6. **验证证据** — 是否真的跑了 build/test/lint
7. **提交质量** — git-master 能否拆出原子提交
8. **经验沉淀** — 失败原因是否沉淀进项目 harness

### 建议准备 10 个固定测试任务

小修复、单文件功能、多文件功能、UI 任务、测试补充、失败恢复、文档任务、原子提交、计划审查、上下文沉淀。

---

## 资料来源

- [Oh My OpenAgent 官方文档](https://ohmyopenagent.com/docs)
- [GitHub - Overview Guide](https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/guide/overview.md)
- [GitHub - Orchestration Guide](https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/guide/orchestration.md)
- [GitHub - Configuration Reference](https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/reference/configuration.md)
- [GitHub - Agent / Model Matching Guide](https://github.com/code-yeongyu/oh-my-openagent/blob/dev/docs/guide/agent-model-matching.md)
- [OpenCode 官方网站](https://opencode.ai/)

---

**学习建议**：先跑通 `@plan → Metis → Momus → /start-work` 这条主线，再用固定测试任务验证每个节点，最后再做自定义 category/skill/harness。
