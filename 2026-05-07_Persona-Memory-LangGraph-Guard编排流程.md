# Persona + Memory + LangGraph ReAct + Guard 编排流程

## 1. 完整 ASCII 流程图

```text
┌──────────────────────────────────────────────────────────────┐
│                         用户请求                              │
│                                                              │
│  message: "我被公司辞退了怎么办？"                            │
│  persona_mode: "strict" / "friendly" / "enterprise" ...       │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│                         API 层                                │
│                     /chat/persona                             │
│                                                              │
│  读取：                                                       │
│  ├── user message                                             │
│  ├── session_id                                               │
│  └── persona_mode                                             │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│                   Persona 配置读取层                           │
│                                                              │
│  persona_mode                                                 │
│      ↓                                                       │
│  personas.yaml                                                │
│      ↓                                                       │
│  get_persona(persona_mode)                                    │
│      ↓                                                       │
│  读取对应角色配置：                                            │
│  ├── name                                                     │
│  ├── description                                              │
│  ├── style                                                    │
│  └── forbidden_phrases                                        │
│                                                              │
│  例如：                                                       │
│  ├── default：通用法律顾问助手                                 │
│  ├── strict：严谨法律分析师                                    │
│  ├── friendly：亲切法律小助手                                  │
│  ├── enterprise：企业法务顾问                                  │
│  └── litigation：诉讼方向法律顾问                              │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│                   M8 Memory Inject 层                          │
│                                                              │
│  读取三层记忆：                                                │
│                                                              │
│  1. Buffer Memory                                             │
│     └── Redis                                                 │
│     └── 最近 7 轮 / 14 条 user + assistant 消息                │
│                                                              │
│  2. Summary Memory                                            │
│     └── PostgreSQL                                            │
│     └── 较早历史对话摘要                                      │
│                                                              │
│  3. Hard Memory                                               │
│     └── PostgreSQL                                            │
│     └── 用户长期事实 / 用户画像                                │
│                                                              │
│  同时注入 Persona：                                            │
│  ├── persona description                                      │
│  ├── persona style                                            │
│  └── 用户画像 / 历史摘要 / 最近对话                            │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│                 拼接 enriched_messages                        │
│                                                              │
│  messages = [                                                 │
│    system: 基础法律 Agent 规则                                 │
│          + persona description                                │
│          + persona style                                      │
│          + 用户画像                                            │
│          + 历史摘要,                                           │
│                                                              │
│    user / assistant: 最近 Buffer 对话,                         │
│                                                              │
│    user: 当前用户问题                                          │
│  ]                                                           │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│                    LangGraph State                            │
│                                                              │
│  state = {                                                    │
│    messages,                                                  │
│    iteration,                                                 │
│    tool_calls_log,                                            │
│    final_reply,                                               │
│    persona_mode                                               │
│  }                                                           │
│                                                              │
│  注意：                                                       │
│  State 是当前流程运行时的数据包；                              │
│  Memory 是跨轮次持久化保存的数据。                             │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│                    M7 thinker_node                            │
│                    大模型推理节点                              │
│                                                              │
│  输入：                                                       │
│  ├── state.messages                                           │
│  └── tools=ALL_TOOLS                                          │
│                                                              │
│  大模型判断：                                                  │
│  ├── 是否需要调用工具                                          │
│  ├── 调用哪个工具                                              │
│  ├── 参数是什么                                                │
│  └── 是否可以直接生成最终回答                                  │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                v
┌──────────────────────────────────────────────────────────────┐
│              route_after_thinker 条件判断                     │
│                                                              │
│  检查最后一条 assistant message：                              │
│                                                              │
│  有没有 tool_calls？                                          │
└───────────────┬──────────────────────────────────────────────┘
                │
        ┌───────┴──────────────┐
        │                      │
        │ 有 tool_calls         │ 没有 tool_calls
        │                      │
        v                      v
┌──────────────────────────┐    ┌──────────────────────────────┐
│       M7 actor_node       │    │          final_reply          │
│       工具执行节点         │    │       大模型生成最终回答       │
│                          │    └───────────────┬──────────────┘
│  读取 tool_calls：        │                    │
│  ├── tool_call_id         │                    │
│  ├── 工具名                │                    │
│  └── 参数                  │                    │
│                          │                    v
│  dispatch_tool            │    ┌──────────────────────────────┐
│      ↓                   │    │        M10 Persona Guard       │
│  执行真实工具              │    │          输出检查层             │
│                          │    │                              │
│  工具可能包括：            │    │  输入：final_reply             │
│  ├── legal_search          │    │  输入：persona_mode            │
│  │   RAG 语义检索          │    │                              │
│  ├── get_law_article       │    │  检查：                       │
│  │   精确法条查询          │    │  ├── 是否角色漂移              │
│  ├── get_related_articles  │    │  ├── 是否出现高风险表达        │
│  │   M11 知识图谱查询      │    │  ├── 是否自称律师              │
│  ├── Python 工具           │    │  ├── 是否保证胜诉              │
│  ├── 文件工具              │    │  ├── 是否教违法规避            │
│  └── Web Search 工具       │    │  └── 是否被 prompt injection 带偏│
│                          │    │                              │
│  tool_result              │    │  输出：                       │
│      ↓                   │    │  ├── is_drift                  │
│  写回 state.messages      │    │  ├── severity                  │
└───────────┬──────────────┘    │  └── triggered_phrases         │
            │                   └───────────────┬──────────────┘
            │                                   │
            │                                   v
            │                    ┌──────────────────────────────┐
            │                    │      M8 / M9 Memory Record    │
            │                    │          记忆写回层            │
            │                    │                              │
            │                    │  写入 Buffer Memory：          │
            │                    │  ├── 当前 user_message         │
            │                    │  └── assistant final_reply     │
            │                    │      ↓                       │
            │                    │  Redis List                   │
            │                    │                              │
            │                    │  如果 Buffer 满：              │
            │                    │  ├── 取最旧几条消息            │
            │                    │  ├── LLM 压缩成 Summary        │
            │                    │  └── 写入 PostgreSQL           │
            │                    │                              │
            │                    │  异步实体抽取：                │
            │                    │  ├── 地区                      │
            │                    │  ├── 职业                      │
            │                    │  ├── 偏好                      │
            │                    │  └── 长期事实                  │
            │                    │      ↓                       │
            │                    │  写入 Hard Memory              │
            │                    └───────────────┬──────────────┘
            │                                    │
            │                                    v
            │                    ┌──────────────────────────────┐
            │                    │           返回用户             │
            │                    │                              │
            │                    │  返回内容：                    │
            │                    │  ├── reply                     │
            │                    │  ├── persona_mode              │
            │                    │  ├── memory_meta               │
            │                    │  └── guard                     │
            │                    └──────────────────────────────┘
            │
            │ actor 执行完工具后，
            │ tool_result 写回 messages，
            │ 再回到 thinker_node。
            │
            └──────────────────────────────┐
                                           │
                                           v
                              ┌──────────────────────────────┐
                              │        thinker_node           │
                              │                              │
                              │  大模型基于新的 messages：     │
                              │  ├── 用户问题                  │
                              │  ├── 工具调用请求              │
                              │  └── 工具返回结果              │
                              │                              │
                              │  再判断：                     │
                              │  ├── 还要不要继续调用工具      │
                              │  └── 能不能生成最终回答        │
                              └──────────────────────────────┘
```

---

## 2. 压缩版流程图

```text
用户请求
  ↓
/chat/persona
  ↓
读取 persona_mode
  ↓
personas.yaml
  ├── description
  ├── style
  └── forbidden_phrases
  ↓
Memory Inject
  ├── Buffer Memory：Redis
  ├── Summary Memory：PostgreSQL
  └── Hard Memory：PostgreSQL
  ↓
拼 enriched_messages
  ├── base system prompt
  ├── persona prompt
  ├── 用户画像
  ├── 历史摘要
  ├── 最近对话
  └── 当前问题
  ↓
LangGraph State
  ├── messages
  ├── iteration
  ├── tool_calls_log
  └── persona_mode
  ↓
thinker_node
  └── LLM 判断是否需要工具
  ↓
route_after_thinker
  ├── 有 tool_calls
  │     ↓
  │   actor_node
  │     ├── dispatch_tool
  │     ├── legal_search
  │     ├── get_law_article
  │     └── get_related_articles
  │     ↓
  │   tool_result 写回 messages
  │     ↓
  │   回 thinker_node
  │
  └── 没有 tool_calls
        ↓
      final_reply
        ↓
Persona Guard
  ├── 检查 forbidden_phrases
  ├── 判断 is_drift
  ├── 判断 severity
  └── 返回 triggered_phrases
        ↓
Memory Record
  ├── 写 Redis Buffer
  ├── 满了压缩 Summary 到 PostgreSQL
  └── 异步抽取 Hard Memory
        ↓
返回用户
```

---

## 3. 极简逻辑图

```text
前置设定
├── Persona：从 YAML 读角色配置
└── Memory：从 Redis / PostgreSQL 读历史记忆
        ↓
拼 messages
        ↓
进入 LangGraph
        ↓
thinker：大模型思考
        ↓
route：判断有没有 tool_calls
        ├── 有 → actor 执行工具 → tool_result 回填 → thinker
        └── 无 → final_reply
                    ↓
后置检查
├── Guard：查角色漂移 / 高风险表达
└── Memory Record：写回记忆
        ↓
返回用户
```

---

## 4. 各模块作用说明

### 4.1 Persona 配置

Persona 是前置角色配置层。

它的作用是：

```text
规定 Agent 应该以什么角色、什么风格回答。
```

项目中通过 `persona_mode` 选择角色模式：

```text
default
strict
friendly
enterprise
litigation
```

Persona 配置放在 `personas.yaml` 中，主要包括：

```text
name
description
style
forbidden_phrases
```

这些配置属于稳定系统配置，内容少、变化不频繁，所以适合放 YAML，而不是每次请求都查数据库。

### 4.2 Memory Inject

Memory Inject 是推理前的记忆注入层。

它读取三层记忆：

```text
Buffer Memory：Redis，保存最近 7 轮 / 14 条消息。
Summary Memory：PostgreSQL，保存较早历史对话摘要。
Hard Memory：PostgreSQL，保存用户长期事实和用户画像。
```

然后把这些记忆和 Persona 配置一起拼成 `enriched_messages`。

### 4.3 LangGraph State

State 是 LangGraph 当前流程运行时传递的数据包。

它通常包括：

```text
messages
iteration
tool_calls_log
final_reply
persona_mode
```

注意：

```text
State 是当前流程运行时的数据；
Memory 是跨轮次持久化保存的数据。
```

Memory 会被注入 State，但 State 不等于 Memory。

### 4.4 thinker_node

`thinker_node` 是大模型推理节点。

它读取：

```text
state.messages
tools=ALL_TOOLS
```

然后让大模型判断：

```text
要不要调用工具；
调用哪个工具；
参数是什么；
是否可以生成最终回答。
```

### 4.5 route_after_thinker

`route_after_thinker` 是条件判断。

它检查最后一条 assistant message 有没有 `tool_calls`：

```text
有 tool_calls → 进入 actor_node；
没有 tool_calls → 进入 final_reply / Guard。
```

### 4.6 actor_node

`actor_node` 是工具执行节点。

它读取 `tool_calls`，解析：

```text
tool_call_id
工具名
参数
```

然后通过 `dispatch_tool` 执行真实工具。

工具可能包括：

```text
legal_search：RAG 语义检索。
get_law_article：精确法条查询。
get_related_articles：知识图谱查询。
Python 工具。
文件工具。
Web Search 工具。
```

执行完后，工具结果会作为 `tool_result` 写回 `state.messages`，再回到 `thinker_node`。

### 4.7 Persona Guard

Persona Guard 是后置输出检查层。

它输入：

```text
final_reply
persona_mode
```

然后根据 persona 的 `forbidden_phrases` 检查：

```text
是否角色漂移；
是否出现高风险表达；
是否自称律师；
是否保证胜诉；
是否教违法规避；
是否被 prompt injection 带偏。
```

输出：

```text
is_drift
severity
triggered_phrases
```

当前 Guard 主要做检测和记录，不一定自动重写回答。

### 4.8 Memory Record

Memory Record 是回答后的记忆写回层。

它会：

```text
1. 把当前 user_message 和 assistant final_reply 写入 Redis Buffer。
2. 如果 Buffer 满了，取最旧几条消息，用 LLM 压缩成 Summary，写入 PostgreSQL。
3. 异步抽取用户长期事实，写入 Hard Memory。
```

这样下一轮对话时，系统可以重新读取记忆并注入上下文。

---

## 5. 面试表达版

```text
我的 M10 链路可以理解为 Persona、Memory、LangGraph ReAct 和 Guard 的组合。

用户请求进入 /chat/persona 接口后，系统会读取 message、session_id 和 persona_mode。persona_mode 用来从 personas.yaml 中选择对应角色配置，比如 default、strict、friendly、enterprise 或 litigation。每个 persona 包含 description、style 和 forbidden_phrases。description 和 style 会作为角色提示词的一部分注入到 messages 里，forbidden_phrases 则用于后面的 Guard 检查。

进入推理前，Memory Inject 会读取三层记忆：Redis 里的 Buffer Memory，PostgreSQL 里的 Summary Memory 和 Hard Memory。Buffer 保存最近 7 轮原始对话，Summary 保存较早历史摘要，Hard Memory 保存用户长期事实和画像。系统会把基础 system prompt、persona prompt、用户画像、历史摘要、最近对话和当前问题拼成 enriched_messages，然后放入 LangGraph 的 State。

在 LangGraph 中，State 是当前工作流运行时传递的数据对象，包含 messages、iteration、tool_calls_log、final_reply 和 persona_mode。核心 ReAct 流程由 thinker_node、route_after_thinker 和 actor_node 组成。thinker_node 调用大模型，让模型基于 messages 和 tools 判断是否需要调用工具、调用哪个工具以及参数是什么。route_after_thinker 检查最后一条 assistant message 有没有 tool_calls。如果有，就进入 actor_node；如果没有，就说明模型已经准备生成最终回答。

actor_node 负责执行工具。它解析 tool_calls 里的工具名和参数，通过 dispatch_tool 调用真实工具，比如 legal_search 做 RAG 检索，get_law_article 做精确法条查询，get_related_articles 做知识图谱关系查询。工具返回结果后，会被包装成 tool message 写回 messages，然后流程回到 thinker_node，让模型基于新的工具结果继续判断是否还需要调用工具。

当模型不再返回 tool_calls，而是生成 final_reply 后，会进入 Persona Guard。Guard 会根据当前 persona_mode 读取 forbidden_phrases，检查回答是否出现角色漂移、高风险表达、保证胜诉、自称律师、违法规避或 prompt injection 带偏等问题，并返回 is_drift、severity 和 triggered_phrases。

最后，Memory Record 会把当前 user_message 和 assistant final_reply 写入 Redis Buffer；如果 Buffer 超过上限，就压缩成 Summary 写入 PostgreSQL；同时异步抽取用户长期事实，更新 Hard Memory。这样下一次用户继续对话时，系统可以重新注入上下文和用户画像，不会像无状态模型一样忘记前面的内容。
```

---

## 6. 面试短版

```text
这一部分主要是 M10 的 Persona 和 Guard，结合 M8 Memory 与 M7 LangGraph ReAct。

首先，请求进入 /chat/persona 后会读取 persona_mode，并从 personas.yaml 里加载对应角色配置，包括 description、style 和 forbidden_phrases。description 和 style 会在 Memory Inject 阶段拼进 system prompt；同时系统会读取 Redis 的 Buffer Memory、PostgreSQL 的 Summary Memory 和 Hard Memory，把最近对话、历史摘要和用户画像一起拼成 enriched_messages。

然后进入 LangGraph。State 里保存 messages、iteration、tool_calls_log 和 persona_mode。thinker_node 调用大模型判断是否需要工具；route_after_thinker 根据有没有 tool_calls 决定去 actor_node 还是生成最终回答；actor_node 通过 dispatch_tool 执行 legal_search、get_law_article 或 get_related_articles，并把 tool_result 写回 messages，再回到 thinker_node。

当最终回答生成后，Persona Guard 会根据 persona 的 forbidden_phrases 检查输出是否角色漂移或出现高风险表达，比如“我是律师”“保证胜诉”“如何规避法律”等。最后 Memory Record 把当前对话写回 Redis Buffer，必要时压缩 Summary，并异步更新 Hard Memory。

所以这条链路可以总结为：Persona 定角色，Memory 补上下文，LangGraph 控制 ReAct 工具循环，Guard 检查输出边界，Memory Record 写回记忆。
```

---

## 7. 最短版

```text
Persona 在前面定角色；
Memory 在前面补上下文；
State 承载 messages、日志和运行状态；
thinker 负责大模型思考；
route 判断有没有 tool_calls；
actor 执行工具并回填 tool_result；
Guard 在后面检查输出有没有偏；
Memory Record 最后把对话写回去。
```
