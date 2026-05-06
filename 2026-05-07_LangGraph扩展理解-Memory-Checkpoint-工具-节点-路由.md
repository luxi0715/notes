# LangGraph 扩展理解：Memory、Checkpoint、工具、节点、路由与状态持久化

## 1. 先把几个概念分清楚

```text
Memory ≠ Checkpoint
更多工具 ≠ 更多节点
复杂路由 ≠ 只是在入口选择大模型
状态持久化 ≠ 记录在大模型里
```

大模型本身不保存 Agent 状态。

状态一般保存在外部存储里，例如：

```text
Redis
PostgreSQL
SQLite
文件系统
向量库
图数据库
对象存储
```

大模型每次只是根据传入的上下文生成输出。

---

## 2. Memory 是什么？

Memory 是给 Agent 提供“历史上下文”和“用户长期信息”。

可以理解为：

```text
Memory
├── Buffer Memory
│   └── 短期记忆，存最近几轮对话，一般放 Redis
│
├── Summary Memory
│   └── 摘要记忆，把较长历史压缩成摘要，一般放 PostgreSQL
│
└── Hard Memory
    └── 用户长期事实，比如地点、身份、偏好，一般放 PostgreSQL
```

### 2.1 Buffer Memory

```text
Buffer Memory：
最近聊天原文，短期、快、会过期。
```

用途：

```text
保存最近几轮对话
解决短期上下文问题
适合放 Redis
```

### 2.2 Summary Memory

```text
Summary Memory：
长对话压缩摘要，避免 messages 无限变长。
```

用途：

```text
压缩较长历史对话
降低上下文长度
降低 token 成本
适合放 PostgreSQL
```

### 2.3 Hard Memory

```text
Hard Memory：
稳定事实，比如用户身份、地区、职业、长期偏好。
```

用途：

```text
保存用户长期事实
支持个性化回答
支持后续 Persona / 用户画像
适合放 PostgreSQL
```

### 2.4 Memory 的目标

```text
让 Agent 记住用户说过什么。
```

---

## 3. Checkpoint 是什么？

Checkpoint 不是普通 Memory。

Checkpoint 更像 Agent 运行过程中的“存档点”。

例如 LangGraph 运行到：

```text
thinker → actor → thinker
```

Checkpoint 可以保存：

```text
当前图运行到哪里
当前 state 是什么
messages 是什么
iteration 是多少
tool_calls_log 是什么
thread_id / session_id 是什么
```

它解决的是：

```text
服务重启后可以恢复
任务中断后可以继续
多轮会话状态不丢
调试时能看到执行轨迹
```

---

## 4. Memory 和 Checkpoint 的区别

| 概念 | 解决什么 | 保存什么 |
|---|---|---|
| Memory | 用户历史和长期信息 | 对话、摘要、用户事实 |
| Checkpoint | Agent 运行状态恢复 | 当前 state、节点进度、messages、iteration |
| Buffer Memory | 最近上下文 | 最近几轮原始对话 |
| Summary Memory | 长历史压缩 | 会话摘要 |
| Hard Memory | 用户长期事实 | 用户身份、地区、偏好 |
| Checkpoint | 图运行存档 | LangGraph state / thread 状态 |

一句话：

```text
Memory 是“用户历史记忆”，Checkpoint 是“Agent 运行状态存档”。
```

---

## 5. 更多工具是什么？

更多工具指的是：

```text
Agent 可以调用更多外部能力。
```

比如：

```text
更多工具
├── RAG 检索工具
├── 法条精确查询工具
├── 知识图谱工具
├── Python 计算工具
├── SQL 查询工具
├── Web Search 工具
├── 文件解析工具
├── 邮件 / 日历 API
├── Docker / Git / Shell 工具
└── 其他业务 API
```

工具的本质：

```text
大模型不会自己干活，它只会发 tool_call。
程序通过 dispatch_tool 调真实函数干活。
```

所以更多工具就是：

```text
扩大 Agent 能干的事。
```

---

## 6. 更多节点是什么？

节点是 LangGraph 里的一个执行步骤。

节点不一定都是大模型。

一个节点可以是：

```text
LLM 节点
工具执行节点
路由节点
记忆读取节点
安全检查节点
答案评估节点
摘要压缩节点
人工确认节点
```

当前 M7 已经有：

```text
thinker_node：大模型推理节点
actor_node：工具执行节点
route_after_thinker：路由函数 / 条件判断
```

后续可以继续加：

```text
memory_inject_node
  └── 读取 Buffer / Summary / Hard Memory，注入 messages

safety_guard_node
  └── 检查用户问题或模型回答是否越界

answer_evaluator_node
  └── 判断最终回答是否完整、有依据

summarizer_node
  └── 对长对话做摘要压缩

human_review_node
  └── 高风险任务交给人工确认

router_node
  └── 判断走法律 Agent、闲聊、拒答、转人工
```

---

## 7. 工具和节点的区别

| 概念 | 含义 | 例子 |
|---|---|---|
| 工具 | Agent 可以调用的外部能力 | RAG、SQL、Python、Web Search |
| 节点 | LangGraph 流程里的一个步骤 | thinker、actor、guard、summarizer |

一句话：

```text
工具 = 能力
节点 = 流程步骤
```

一个节点可以调用工具，比如：

```text
actor_node 调用 dispatch_tool 执行工具。
```

一个节点也可以不调用工具，比如：

```text
route_after_thinker 只判断下一步去哪。
```

---

## 8. 复杂路由是什么？

复杂路由不只是选择哪个大模型。

选择模型只是其中一种路由，叫：

```text
Model Router
```

更完整的复杂路由包括：

```text
复杂路由
├── Query Router
│   └── 判断用户问题是法律、闲聊、无关问题
│
├── Tool Router
│   └── 判断该调用 RAG、SQL、Python、知识图谱
│
├── Model Router
│   └── 判断用小模型、大模型、推理模型
│
├── Persona Router
│   └── 判断用普通、严格、企业、诉讼模式
│
├── Safety Router
│   └── 判断是否拒答、降级、转人工
│
└── Workflow Router
    └── 判断走哪个流程分支
```

复杂路由的核心是：

```text
根据输入和状态，决定下一步走哪条路径。
```

---

## 9. M9 Cascade Router 怎么理解？

M9 的 Cascade Router 是一种典型路由。

它的思想是：

```text
规则能判断 → 直接返回
规则模糊 → 升级 LLM 判断
```

比如先判断用户问题属于：

```text
greeting
legal
off_topic
```

它不是只选模型。

它是在入口处先判断用户请求该走哪条路径。

---

## 10. 状态持久化是什么？

状态持久化不是“记录在大模型里”。

准确说是：

```text
把 Agent 运行状态保存到外部数据库或存储系统里。
```

可以保存到：

```text
PostgreSQL
Redis
SQLite
文件
对象存储
```

保存的内容可能包括：

```text
messages
iteration
tool_calls_log
thread_id
session_id
当前图运行状态
```

作用：

```text
服务重启后能恢复
任务中断后能继续
多轮会话状态不丢
调试时能看到执行轨迹
```

一句话：

```text
状态持久化 = 把 Agent 的运行 state 存进数据库，不是存进大模型。
```

---

## 11. 标准表达版

```text
M7 用 LangGraph 后，后续扩展更方便，主要体现在 Memory、Checkpoint、更多工具、更多节点、复杂路由和状态持久化几个方向。

Memory 是给 Agent 提供用户历史和长期信息，比如 Buffer Memory 存最近几轮对话，Summary Memory 存长对话摘要，Hard Memory 存用户长期事实。

Checkpoint 不是普通记忆，而是保存 LangGraph 的运行状态，比如当前 messages、iteration、tool_calls_log 和 thread_id，让服务重启或任务中断后可以恢复。

更多工具指的是给 Agent 增加外部能力，比如 RAG、SQL、Python、知识图谱、Web Search、文件解析等。

更多节点指的是在 LangGraph 流程里增加新的步骤，不一定都是大模型。节点可以是记忆注入节点、安全检查节点、答案评估节点、摘要节点、人工审核节点，也可以是 LLM 节点。

复杂路由指的是根据用户问题和当前状态决定走哪条路径，不只是选择哪个大模型，也包括 query 分类、工具选择、persona 选择、安全判断和工作流分支。

状态持久化是把 Agent 运行过程中的 state 保存到数据库里，比如 PostgreSQL，而不是保存到大模型里。大模型只根据每次传入的上下文生成输出，本身不负责保存运行状态。
```

---

## 12. 面试表达版

```text
LangGraph 相比手写循环的一个优势是扩展更自然。

比如 Memory、Checkpoint、更多工具、更多节点、复杂路由和状态持久化都可以比较清晰地接进去。

Memory 主要解决用户历史和长期事实的问题，比如最近几轮对话可以放到 Buffer Memory，长历史可以压缩成 Summary Memory，用户稳定事实可以放到 Hard Memory。

Checkpoint 解决的是 Agent 图运行状态恢复的问题，它保存的不是用户事实，而是 LangGraph 当前 state，比如 messages、iteration、tool_calls_log 和 thread_id。这样服务重启或者任务中断后可以恢复执行状态。

更多工具是扩大 Agent 的能力，比如接 RAG、SQL、Python、知识图谱、Web Search 或文件解析工具。

更多节点则是扩展工作流步骤，节点不一定是大模型，也可以是记忆注入节点、安全检查节点、答案评估节点、摘要节点或人工审核节点。

复杂路由也不只是选择模型，还包括 query 分类、工具选择、persona 选择、安全判断和工作流分支。

最后，状态持久化是把 Agent 的运行 state 存到 PostgreSQL、Redis 或其他外部存储里，不是存到大模型里。
```

---

## 13. 最短版

```text
Memory 是用户历史记忆，Checkpoint 是 Agent 运行状态存档。

工具是外部能力，比如 RAG、Python、SQL、知识图谱。

节点是流程步骤，可以是 LLM、工具执行、记忆注入、安全检查、答案评估，不一定都是大模型。

复杂路由不是只选模型，而是决定走哪条路径，比如法律、闲聊、拒答、工具选择、persona 选择。

状态持久化是把 state 存到数据库，不是存到大模型。
```
