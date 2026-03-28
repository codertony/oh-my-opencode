

先评价你现在的方案

你的主线是成立的，尤其适合做系列文章：

第一，以现成 harness 为学习入口是正确的。
复杂 Agent 的门槛不在“会不会调用 LLM API”，而在“怎么分层、怎么分工、怎么控制上下文、怎么保证任务收敛”。Oh My OpenAgent 的公开文档已经把问题暴露得很典型：它把系统拆成规划层、执行层、worker 层，并显式处理 context overload、cognitive drift、verification gaps，这正是复杂 Agent 工程的核心问题。

第二，把 OpenCode 当底座、把 OpenAgent 当业务编排层，这个定位也对。
OpenCode 更像“通用 agent runtime / tool substrate”，提供工具、规则、权限、插件、MCP、LSP、SDK 等基础设施；Oh My OpenAgent 则在这个之上叠加更强的角色分工、模型匹配、恢复与编排策略。换句话说，你不是在学一个“单点功能”，而是在学 runtime + orchestration + workflow policy 三层。

第三，你把“学习路径”和“做工具”绑定，是更高效的学习法。
因为真正的理解，不是把 agent 名字背下来，而是能回答：
“为什么要这样分工？”
“这个分工能不能换一种？”
“在我的开发流程里，哪个部分值得抽成插件/工具？”
只要你每篇文章最后都落到一个可运行增强点，这条路线就不只是可行，而是很强。

你当前方案的主要缺口
1. 少了“扩展点分析”这一大维度

你现在强调协作链路、调度、记忆、上下文、工程稳定性，但还缺一个非常实用的维度：
这个系统到底哪里能改，哪里不该改，哪里应该通过插件/MCP/custom tool 改。

这点非常关键，因为 OpenCode 已经明确提供了插件事件、project/global plugin 目录、npm plugin、custom tools、MCP server 接入和 SDK；如果你不单独分析“扩展边界”，文章最后会变成源码导览，而不是二次开发指南。

建议把这部分单列成一个核心维度：

扩展与产品化维度
看哪些能力应放在：

prompt / AGENTS.md 规则层
agent 编排层
plugin hook 层
custom tool 层
MCP integration 层
独立外部服务层

这是你从“读源码的人”走向“能做套件的人”的分水岭。

2. 少了“评测与验收”维度

很多 Agent 系统失败，不是因为设计得差，而是因为 没有稳定评测方法。
Oh My OpenAgent 已经很强调 verification、review、fallback、session recovery 这些机制，说明它面对的不是“能不能回答”，而是“能不能持续交付”。

所以你必须补一个维度：

Agent 评测维度

任务完成率
首轮成功率
回退率 / 重试率
token 成本
平均时延
变更正确率
回归风险
人工接管率

没有这套指标，你很难判断“你的开发效率 Agent 套件”到底比单 agent、比 Cursor/Codex/OpenCode 原生模式强在哪里。

3. 少了“安全与权限治理”维度

你现在提到工程稳定性，但还不够聚焦。
Agent 真正在企业或日常开发里变得危险的地方，不是“模型答错”，而是 bash/edit/write/webfetch/MCP 能力越界。OpenCode 明确支持全局权限、按 agent 覆盖权限、工具级 allow/ask/deny，以及规则系统和 AGENTS.md 指令注入。

这意味着你应该补一章专门讨论：

安全治理维度

谁能执行写操作
谁能跑 shell
谁能访问网络/MCP
谁能改配置/删文件
哪些步骤必须人工确认
如何防 prompt injection / tool misuse / runaway execution

这是“能 demo”与“能日常使用”的区别。

4. 少了“成本、性能、上下文经济学”维度

你已经提了 memory 和 context，但还没上升到工程账本。
Oh My OpenAgent 公开文档和特性说明里，已经把 context-window monitor、preemptive compaction、tool-output truncator、model fallback、runtime fallback 这些做成了显式机制；这说明复杂 Agent 的一个核心问题就是 token/上下文/模型成本管理。

因此建议单独补充：

性能经济学维度

哪些任务值得多 agent
哪些任务单 agent 更划算
规划层和执行层如何用不同模型
深推理何时触发
compaction 何时触发
输出裁剪对正确率影响多大

否则你做出来的工具很可能“很聪明，但不够便宜、不够快”。

5. 少了“可观测性与调试”维度

你要做系列解析文章，更要做日常开发 Agent 套件，就不能只讲“流程图”，还要讲 怎么观察系统运行。
OpenCode 有日志、故障排查、server/SDK 能力；Oh My OpenAgent 也公开了大量 recovery / monitor / verification / session tracking 机制。

建议加一个维度：

可观测性维度

agent 间调用链
每步使用的模型
每步 token / latency
失败点与 fallback 原因
计划版本与执行偏差
最终交付证据

这一维度会直接提升你文章的含金量，因为大多数“源码解析”只讲静态结构，不讲运行态。

6. 少了“知识沉淀闭环”维度

你提到记忆管理，这是对的，但最好把“会话记忆”与“项目知识沉淀”拆开。
OpenCode 通过 AGENTS.md 注入项目规则；Oh My OpenAgent 社区也在讨论把 session learnings 自动沉淀到 AGENTS.md / CLAUDE.md / skills，说明这不是简单的“记忆”，而是 短期上下文 → 长期知识库 的闭环问题。

所以这里建议改成两层：

运行时记忆：当前会话、压缩、恢复、上下文裁剪
持久化知识：经验总结、反模式、团队规范、技能库、项目规则

这会比“记忆管理”更工程化。

我建议你把“五个方向”升级成“八个方向”

你原来的五个方向可以保留，但建议重组为下面这八个：

底座与边界
OpenCode 提供什么，OpenAgent 重写了什么，哪些是继承、哪些是增强。
协作与注入链路
规则注入、AGENTS.md、目录级上下文、工具调用前后钩子。
多 Agent 编排机制
Intent gate、planner、orchestrator、worker 的职责划分与委派策略。
上下文与记忆体系
上下文压缩、窗口监控、恢复策略、短期/长期知识边界。
工具与扩展体系
plugins、custom tools、MCP、SDK，哪些适合做你自己的效率套件。
安全与权限治理
tool permission、agent-specific permission、写操作守卫、人工确认机制。
评测、验证与稳定交付
review、verification、fallback、session recovery、成功率指标。
产品化与开发工作流适配
如何把它做成 PR 助手、重构助手、文档助手、排障助手、代码审查助手，而不是泛泛的“万能 agent”。
你的文章系列最好不要只做“源码分层”，而要做“双输出”

每一篇都同时回答两个问题：

学习侧问题
这个模块体现了什么 Agent 设计思想？

落地侧问题
如果我要做开发效率工具，这一层最值得抽成什么能力？

例如：

讲调度逻辑时，不只讲委派链路，还要落到“任务拆分器 + 执行监工”
讲上下文管理时，不只讲 compaction，还要落到“长任务不中断”
讲工具权限时，不只讲 config，还要落到“安全可控的自动改代码”

这样整套连载才会自然服务你的双目标。

还要补一个现实约束：版本漂移控制

这个项目现在变化很快。公开 issue 里已经有人指出文档分散、更新频繁且幅度大；项目本身在 2026 年 3 月刚完成从 oh-my-opencode 到 oh-my-openagent 的命名调整，并且 release 中还能看到 agent 命名、模型映射、验证策略都在快速演进。

这意味着你的系列文章必须做两件事：

第一，固定分析基线。
每一篇都注明基于哪个 tag / release / commit。

第二，区分“稳定机制”和“易变实现”。
比如：

稳定机制：分层编排、上下文压缩、fallback、权限隔离
易变实现：具体 agent 名称、默认模型、某个 hook 名称、某条 fallback chain

不这么做，系列会很快过期。

你可以直接采用的连载结构

比较适合的写法，不是先按源码目录写，而是按“工程问题”写：

为什么 OMY OpenAgent 值得读：不是插件集合，而是 Agent orchestration harness
OpenCode 底座：tool、rule、plugin、MCP、LSP、SDK 提供了什么能力
协作与注入链路：AGENTS.md、规则系统、目录上下文怎么进 prompt
规划层：Prometheus/Plan 类能力如何把模糊需求转成可执行计划
执行层：Orchestrator 怎样分派任务、控制收敛
Worker 层：专职 agent 如何分工，何时专用、何时通用
Context/Memory：压缩、恢复、知识沉淀三件事怎么分开
Verification/Fallback：为什么复杂 Agent 不能靠“我觉得完成了”
扩展实战：用 plugin/custom tool/MCP 做一个开发效率增强点
产品化总结：怎样从源码理解走到自己的 Agent 套件

这会比“第一篇讲 A 文件，第二篇讲 B 文件”更有读者价值。

最后给你的结论

结论很明确：

这条学习与研究路线是可行的，而且是高质量路线。
但要把目标从“源码解析”提升为“成为 Agent 开发工程师”，你必须把当前方案从“架构理解导向”扩成“架构 + 扩展 + 评测 + 治理 + 产品化”五位一体。

一句话概括，就是：

现在的方案适合成为“优秀的源码解析系列”；补上我上面这几层后，它才会真正变成“Agent 工程师成长路线图”。

你这套计划下一步最该做的，不是继续补零散分析点，而是先把系列总纲改成“八个核心维度 + 每篇一个可落地增强实验”。

重新梳理核心要学习的问题。帮我重新生成一份学习研究的路线图和文章架构

我先把你的原始目标拆成两层：一层是“要学会什么问题”，一层是“怎么把这些问题组织成连载文章”。我会直接给你一版可执行的学习研究路线图，并把文章架构按由浅入深重新编排。

我已经把它重新收束成一个更清晰的问题域了：不是“我要把源码都看完”，而是“我要弄清楚复杂 Agent 为什么能稳定工作，以及哪些机制能迁移到自己的开发套件里”。接下来我会把路线图改写成“问题驱动 + 文章驱动”的双层结构。

Thought for 36s

可以。你现在最需要的，不是继续加“分析点”，而是把学习对象从“源码模块”改成“核心问题”。因为复杂 Agent 真正难的不是目录结构，而是这几个问题：任务如何拆、上下文如何控、工具如何接、权限如何收、结果如何验、系统如何稳。而 OMY OpenAgent 本身正是构建在 OpenCode 之上的多模型编排层，文档也明确把重点放在 planning/execution 分离、专职 agent 分工、上下文注入、恢复机制、权限与扩展上，所以很适合按“问题驱动”来学，而不是按文件夹顺序来学。

一、重新梳理后的核心学习问题

我建议把“要学什么”收敛成 8 个问题。之后所有学习动作、源码阅读、文章撰写，都围绕这 8 个问题展开。

1. 一个复杂 Agent 系统到底在解决什么问题？

你要先搞清楚：OMY OpenAgent 不是“多几个 prompt”，而是把单个 coding agent 提升为一个可协作的开发团队，核心是让复杂任务能被规划、委派、执行、验证和恢复。官方 overview 和 orchestration guide 都把它定义为 orchestration harness，并强调 planning / execution separation。

你真正要学的问题不是“它有哪些 agent”，而是：
为什么复杂开发任务不能只靠一个 agent 一把梭？

2. 任务是如何从“用户需求”变成“可执行工作流”的？

这是最核心的一层。你需要研究：

什么情况下只需要直接 prompt
什么情况下要先 plan
plan 如何变成待执行任务
任务粒度如何控制
谁负责验收标准

OMY OpenAgent 的 orchestration 文档已经给出“简单任务直接做、复杂任务走 plan / work”的分层方式，并且有专门的 planner、reviewer、executor 角色。

3. 多 Agent 为什么能比单 Agent 更稳定？

这里不要停留在“有很多 agent 很酷”，而要追问：

这些 agent 的职责边界是什么
哪些是只读顾问，哪些能写代码
哪些能继续委派，哪些不能
为什么 orchestrator 和 worker 不能混成一个

features 文档已经把角色分成 orchestrator、planning agents、specialists，并对不同 agent 的工具权限做了限制，例如 Oracle/Librarian 只读、Atlas 不能再委派。

4. 上下文为什么会失控，系统如何控制它？

复杂 Agent 的关键难点不是“会不会说”，而是“做长任务时会不会漂”。这里要重点学习：

AGENTS.md / README / rules 是怎么注入的
目录级上下文如何叠加
context window 如何监控
compaction 如何做
长会话如何恢复

OpenCode 和 OMY OpenAgent 都明确支持 AGENTS.md 规则体系、目录上下文注入、context-window monitor、preemptive compaction、session recovery。

5. “记忆”到底应该分成哪几层？

你原来把“记忆管理”放成一个点，但建议拆成三层：

运行时记忆：当前任务执行上下文
阶段性沉淀：learnings / decisions / issues / verification
项目长期知识：AGENTS.md、skills、规则、约定

在 orchestration 和 features 文档里，已经能看到 notepad 体系、verification 记录、层级 AGENTS.md、skills 这些机制。

6. 工具体系是如何决定 Agent 上限的？

想做开发效率套件，这一层必须学透。你要研究：

原生工具够不够
什么能力适合做 custom tool
什么能力适合做 plugin
什么能力适合接 MCP
哪些能力不该塞进主编排 prompt

OpenCode 官方文档明确提供 plugin hooks、custom tools 和 MCP servers，这正是你以后做效率增强套件的主要落点。

7. Agent 为什么“能跑起来”不等于“能交付”？

这一层要学的是工程化，而不是提示词。关键问题包括：

验收标准怎么定义
verification 怎么嵌进流程
review 在哪一步做
fallback 何时触发
会话坏了怎么恢复

OMY OpenAgent 的 reviewer 机制会检查 clarity、verification、context、big picture；同时还有模型 fallback、session recovery、context-window-limit recovery 等稳定性机制。

8. 安全和权限为什么是 Agent 工程的硬门槛？

如果你的目标是做日常开发 Agent 套件，这个问题必须前置：

哪些动作可以自动执行
哪些必须 ask
哪些必须 deny
哪些 agent 只能读不能写
如何防止越权修改

OpenCode 的 permissions 机制就是为此设计的，OMY OpenAgent 也对不同 agent 设定了不同工具限制。

二、学习研究路线图

下面这版路线图不是“把源码读完”，而是“每个阶段解决一类问题，并产出一篇或两篇文章 + 一个小实验”。

阶段 0：建立坐标系

目标不是深入源码，而是先把系统地图画出来。

你要完成三件事：

搞清楚 OpenCode 是底座，OMY OpenAgent 是编排增强层。OpenCode 负责 agent、rules、plugins、tools、permissions、MCP、初始化项目规则；OMY OpenAgent 在这之上增强编排、角色分工、上下文控制和稳定机制。
画出系统分层图：底座能力层、编排层、执行层、扩展层、治理层。
固定研究基线：每篇文章都注明基于哪个 repo 分支 / 文档版本，否则项目演化会让系列迅速失效。近期 issue 里也明确提到文档升级和增强文档这件事本身正在推进。

阶段产出：

1 张系统架构图
1 篇总览文章
1 份术语表
阶段 1：先学“编排”，不要先学“实现细节”

这一阶段只研究任务如何流动。

核心问题：

用户请求如何进入系统
谁判断复杂度
谁做规划
谁做审查
谁做执行
谁做收尾验证

你要把 orchestration guide 读成“工作流状态机”，而不是读成“agent 名单”。因为官方文档已经明确给出了 planning / execution 分离，以及 planner、reviewer、executor 的协作方式。

阶段产出：

任务流转图
一篇“复杂 Agent 为什么必须分层”的文章
一个最小版任务分派流程图
阶段 2：研究“角色边界”和“委派原则”

你接下来要回答的是：为什么有些 agent 只读，有些 agent 可写，有些 agent 不能再委派。

这一阶段重点读：

各 agent 的职责定义
model matching 背后的角色假设
工具权限限制

features 和 agent-model-matching 文档已经明确说明，不同 agent 的职责不是平铺的，而是按 orchestration、analysis、deep work、lookup、review 拆分；有的模型适合 orchestrator，有的更适合 deep specialist。

阶段产出：

一张“agent 职责边界表”
一篇“复杂 Agent 不是堆人头，而是职责隔离”的文章
一个你自己的“开发效率套件角色设计草案”
阶段 3：研究“上下文系统”

这是你从“看懂架构”走向“能自己做”的关键阶段。

重点问题：

AGENTS.md 为什么重要
项目规则和个人规则怎么叠加
README 与目录上下文的注入时机是什么
长任务怎么不丢上下文
compaction 会损失什么

OpenCode 规则系统支持 project/global AGENTS.md，并规定了查找优先级；OMY OpenAgent 进一步提供目录级 AGENTS.md 注入、README 注入、上下文监控和预压缩机制。

阶段产出：

一篇“Agent 的上下文不是 prompt，而是分层知识系统”
一个示例项目的 AGENTS.md 设计模板
一个“长任务上下文漂移排查清单”
阶段 4：研究“记忆、沉淀与恢复”

这一阶段要把“记忆”从概念拉回工程。

重点问题：

哪些信息只应留在会话里
哪些应该写入 notepad / verification / decisions
哪些应该回写项目知识
会话中断后如何恢复

orchestration guide 里有 learnings、decisions、issues、verification 等工作沉淀结构，features 里也有 session-recovery 和 handoff / init-deep 一类机制。

阶段产出：

一篇“Agent 记忆不等于向量库”的文章
一个“短期记忆 / 长期知识”分层模型
一个 session handoff 模板
阶段 5：研究“扩展能力”，开始做自己的开发套件

到这一步才开始真正做东西。

你要把需求拆成三类：

通过规则能解决的
通过 plugin hook 能解决的
通过 custom tool / MCP 能解决的

OpenCode 的 plugins 支持按事件扩展行为，custom tools 可定义可调用函数，MCP servers 用于接外部服务。这些就是你开发“日常开发 Agent 套件”的主战场。

建议你优先做三个小插件/工具：

PR 评审助手
代码库检索与模式归纳助手
任务分解与验收清单生成器

阶段产出：

一篇“哪些能力该放在 plugin，哪些该放在 tool”
1～2 个最小可用增强模块
一个你的 Agent 套件 v0.1
阶段 6：研究“验证、评测与稳定性”

这一步是把“能跑”变成“能用”。

重点问题：

一个任务什么时候算完成
如何自动检查“不完整完成”
fallback 的代价和收益是什么
稳定性如何观测

OMY OpenAgent 已把 verification、lsp diagnostics、todo continuation、fallback 和 recovery 作为系统机制的一部分，这说明“稳定交付”是核心设计目标之一。

你要建立一套自己的评测指标：

任务完成率
首轮成功率
平均修正轮次
失败原因分类
token / latency / cost
人工接管率

阶段产出：

一篇“Agent 系统为什么必须有验收闭环”
一套评测表
你的 Agent 套件 v0.2 评测报告
阶段 7：研究“权限治理与产品化落地”

最后一阶段才谈“日常使用”。

OpenCode 的 permissions 支持自动、询问、阻止三类控制，OMY OpenAgent 也把不同角色的工具能力限制得很明确。做日常开发 Agent 套件时，这一层决定了你敢不敢真的开自动化。

你要输出的是：

哪些任务可自动执行
哪些任务必须审批
哪些任务只能建议不能落地
团队协作时如何共享规则和能力

阶段产出：

一篇“开发 Agent 从 demo 到生产，差在权限和治理”
你的 Agent 套件 v1.0 使用规范
三、对应的文章架构

下面这版更适合做系列连载。每篇文章都同时服务两个目标：
一边帮你学 Agent，一边帮你沉淀自己的开发效率工具。

第一部分：建立认知框架
第 1 篇：为什么要读 OMY OpenAgent，而不是再看一遍普通 AI coding agent

回答的问题：

它解决的到底是什么问题
它和 OpenCode 的关系是什么
为什么它值得作为复杂 Agent 学习样本
第 2 篇：复杂 Agent 系统的五层结构

建议写成：

底座层
编排层
执行层
扩展层
治理层

这篇是全系列的总纲。

第二部分：吃透任务编排
第 3 篇：从用户需求到执行计划，任务是怎么被拆开的

重点写：

复杂度判断
规划入口
任务拆分
验收条件
第 4 篇：Planner、Reviewer、Executor 为什么不能合并

重点写：

planner 负责什么
reviewer 为什么必要
executor 为什么必须受约束
intelligence 在系统里，而不是在某个 agent 里

这一点在 orchestration guide 中讲得非常清楚：执行稳定性来自清晰约束、todo 机制和验证要求，而不是单个模型足够聪明。

第 5 篇：多 Agent 委派的真正价值是什么

重点写：

并行的价值
专职角色的价值
什么时候不该多 agent
第三部分：吃透上下文、记忆与恢复
第 6 篇：AGENTS.md、README、rules 是如何成为 Agent 的外部脑

重点写：

project / global 规则
目录级上下文
规则优先级
为什么要 commit AGENTS.md

OpenCode 文档明确建议初始化后创建并提交项目的 AGENTS.md。

第 7 篇：长任务为什么会漂，OMY OpenAgent 如何处理 context overload

重点写：

context monitor
preemptive compaction
compaction 时保留什么
什么信息不该靠上下文硬扛
第 8 篇：Agent 的记忆体系应该怎么设计

建议你把这篇写成：

运行时上下文
过程沉淀
长期知识
第四部分：吃透工具与扩展
第 9 篇：从源码阅读走到能力扩展，plugin、tool、MCP 该怎么选

这是你做套件最重要的一篇。
重点写：

规则层 vs plugin 层 vs tool 层 vs MCP 层
哪类需求适合放哪一层
第 10 篇：基于 OMY OpenAgent 做一个开发效率增强模块

选一个最贴近日常开发的场景，比如：

PR review
重构辅助
Bug 排查
文档生成

写成完整实战。

第五部分：吃透工程化交付
第 11 篇：为什么复杂 Agent 不能没有 verification

重点写：

clarity / verification / context / big picture
验收标准写法
自动验证和人工复核的边界
第 12 篇：权限、回退与恢复，Agent 系统怎么才能日常可用

重点写：

permissions
tool restriction
fallback
session recovery
第 13 篇：我的 OMY OpenAgent 学习总结与 Agent 套件路线图

这是收官篇。
内容包括：

学到的系统设计原则
做出来的能力清单
下一步如何扩展成自己的 Agent 工程体系
四、我建议你现在就采用的“每篇文章固定模板”

这样能保证系列稳定输出，不会越写越散。

每篇都按这 6 段写：

这篇要回答什么问题
源码里这个问题出现在哪里
它当前的设计方案是什么
这套方案解决了什么，牺牲了什么
如果我要做自己的开发 Agent 套件，我会怎么迁移
一个最小实验 / 小改造

这样写，文章就不会沦为“源码导览”，而会变成“工程理解 + 方法迁移”。

五、你现在最该优先学习的顺序

如果只讲最实际的优先级，我建议是：

先学 任务编排，再学 上下文系统，再学 角色边界，然后进入 扩展能力，最后补 验证、权限和稳定性。
原因很简单：你如果先看插件和工具，很容易陷入“会接东西”；但如果没理解编排、上下文和验证，你做出来的只是能调用工具的 prompt 工程，不是复杂 Agent 系统。

六、给你一版可直接开写的目录

你可以直接把系列名定成：

《OMY OpenAgent 源码解析：从复杂 Agent 学习到开发效率套件落地》

子标题顺序建议：

为什么 OMY OpenAgent 值得作为复杂 Agent 学习样本
OpenCode 底座与 OMY OpenAgent 编排层：边界在哪
复杂任务为什么必须先规划再执行
Planner / Reviewer / Executor 的角色分工设计
多 Agent 委派不是炫技，而是稳定性交换
AGENTS.md、rules 与目录上下文注入机制
Context overload、压缩与会话恢复
复杂 Agent 的记忆体系：运行时、沉淀层、长期知识
Plugin、custom tool、MCP：能力扩展的三种路线
做一个开发效率增强模块：从源码理解到实战改造
Verification、fallback 与稳定交付闭环
权限治理与产品化：让 Agent 真正进入日常开发

