# legal-agent 分层架构与 M8-M11 功能标注

## 1. 系统分层图

```text
legal-agent
├── API 层
│   └── 接收用户请求，返回回答
│       └── M9：接入服务启动 / 关闭时的工程增强
│           ├── Redis 连接池预热
│           └── Checkpoint 初始化 / 关闭
│
├── Router 层
│   └── 判断 greeting / legal / off_topic
│       └── M9：Cascade Router
│           ├── 规则高置信先判
│           ├── 模糊 query 升级 LLM Router
│           └── 避免所有请求都进入完整法律 Agent
│
├── Memory 层
│   ├── Buffer Memory：Redis，短期上下文
│   │   └── M8：短期记忆
│   │       ├── Redis LIST
│   │       ├── 保存最近 7 轮对话
│   │       └── TTL 7 天
│   │
│   ├── Summary Memory：PostgreSQL，中期摘要
│   │   └── M8：摘要记忆
│   │       ├── session_summaries 表
│   │       ├── Buffer 满后触发压缩
│   │       └── LLM 滚动摘要压缩
│   │
│   ├── Hard Memory：PostgreSQL，长期用户事实
│   │   └── M8：长期事实记忆
│   │       ├── user_facts 表
│   │       ├── user_id + key + value + confidence
│   │       └── 保存地区、职业、偏好等稳定事实
│   │
│   └── Memory Manager：读写和注入记忆
│       ├── M8：三层记忆调度
│       │   ├── 推理前 inject_memory_into_messages
│       │   └── 推理后 record_turn
│       │
│       ├── M9：异步实体抽取
│       │   └── 把 extract_and_persist 放到后台任务
│       │
│       └── M10：Persona 结合 Memory
│           ├── persona_mode
│           ├── 用户画像文本
│           └── persona system prompt + memory context
│
├── Agent 编排层
│   ├── M6：手写 ReAct Loop
│   │   ├── for 循环
│   │   ├── 调 LLM
│   │   ├── 判断 tool_calls
│   │   ├── dispatch_tool
│   │   └── tool_result 写回 messages
│   │
│   ├── M7：LangGraph StateGraph 编排
│   │   ├── ReactState
│   │   ├── thinker_node
│   │   ├── actor_node
│   │   ├── route_after_thinker
│   │   └── START → thinker → actor / END
│   │
│   └── M9：Checkpoint 状态保存
│       ├── 保存 LangGraph state
│       ├── thread_id / session_id 关联
│       └── 服务重启后可恢复图状态
│
├── Tool 层
│   ├── legal_search：RAG 语义检索
│   │   └── M6 / M7：ReAct 可调用工具
│   │
│   ├── get_law_article：精确法条查询
│   │   └── M6 / M7：ReAct 可调用工具
│   │
│   └── get_related_articles：知识图谱关联法条
│       └── M11：知识图谱工具
│           ├── 查询法条关联关系
│           ├── 补足 RAG 对结构关系的不擅长
│           └── 用于“哪些法条相关 / 引用 / 被引用”类问题
│
├── Dispatcher 层
│   └── 根据工具名调用真实 Python 函数
│       ├── M6：手写 ReAct 里 dispatch_tool
│       ├── M7：actor_node 里 dispatch_tool
│       └── M11：新增 get_related_articles 后扩展注册表
│
├── 存储层
│   ├── Redis：Buffer Memory / 短期缓存
│   │   ├── M8：Buffer Memory
│   │   └── M9：Redis 连接池预热
│   │
│   ├── PostgreSQL：Summary / Hard Memory / 法条数据
│   │   ├── M8：session_summaries
│   │   ├── M8：user_facts
│   │   └── M9：Checkpoint 可接持久化存储
│   │
│   ├── 向量库：RAG 检索
│   │   └── M6 / M7：legal_search 使用
│   │
│   └── Neo4j：法条关系图谱
│       └── M11：知识图谱关系存储
│           ├── 法条节点
│           ├── 引用关系
│           └── 关联法条查询
│
└── Guard / Persona 层
    ├── Persona
    │   └── M10：多 Persona 模式
    │       ├── default
    │       ├── strict
    │       ├── concise
    │       └── 其他角色风格
    │
    └── Guard
        └── M10：Persona Guard
            ├── 检查角色漂移
            ├── 检查高风险表达
            ├── 例如“我是律师”“保证胜诉”
            └── 记录 drift 日志
```

---

## 2. 阶段主线

```text
M6：Agent 会用工具
  └── 手写 ReAct Loop：LLM 判断 tool_calls，程序 dispatch_tool，结果写回 messages

M7：Agent 流程图结构化
  └── LangGraph：thinker / actor / route / ReactState / StateGraph

M8：Agent 有记忆
  └── Buffer Memory + Summary Memory + Hard Memory + Memory Manager

M9：Agent 更工程化
  ├── Cascade Router
  ├── 异步实体抽取
  ├── Checkpoint
  └── Redis 连接池预热

M10：Agent 有角色控制和输出约束
  ├── Persona
  ├── 用户画像
  └── Persona Guard

M11：Agent 增强结构化法律关系能力
  └── Knowledge Graph Tool：get_related_articles + Neo4j
```

---

## 3. M8-M11 分布位置总结

```text
M8 主要落在 Memory 层；
M9 落在 Router、Checkpoint、异步任务和 Redis 预热；
M10 落在 Persona / Guard 层，并和 Memory Manager 结合；
M11 落在 Tool 层和 Neo4j 存储层，增强法条关系查询。
```

---

## 4. 面试表达版

```text
这个项目可以按后端系统分层来理解。

API 层负责接收用户请求并返回回答；Router 层负责先判断请求是 greeting、legal 还是 off_topic，M9 的 Cascade Router 就是在这里做的，避免所有请求都进入完整法律 Agent。

Memory 层是 M8 的核心，包括 Redis 里的 Buffer Memory、PostgreSQL 里的 Summary Memory 和 Hard Memory，以及 Memory Manager。推理前，Memory Manager 负责读取三层记忆并拼成 messages；推理后，负责写入 Buffer、异步抽取用户事实，并在 Buffer 满时触发 Summary 压缩。M9 在这里把实体抽取改成异步，M10 又把 Persona 和用户画像结合进 Memory Manager。

Agent 编排层从 M6 的手写 ReAct Loop 演进到 M7 的 LangGraph StateGraph。M6 里是一个 for 循环完成调模型、判断 tool_calls、执行 dispatch_tool、写回 messages；M7 把它拆成 thinker_node、actor_node、route_after_thinker 和 ReactState。M9 又在这一层加入 Checkpoint，用于保存 LangGraph 的运行状态。

Tool 层包含 legal_search、get_law_article 和 get_related_articles。前两个主要服务于 RAG 检索和精确法条查询，M11 新增 get_related_articles，用 Neo4j 法条关系图谱补足 RAG 对结构化关系查询的不擅长。

Dispatcher 层负责根据模型返回的工具名调用真实 Python 函数。模型只生成 tool_calls，真正执行工具的是 dispatch_tool。

存储层包括 Redis、PostgreSQL、向量库和 Neo4j。Redis 主要服务于 Buffer Memory，PostgreSQL 存 Summary、Hard Memory 和法条数据，向量库服务 RAG 检索，Neo4j 服务 M11 的知识图谱关系查询。

Guard / Persona 层是 M10 的核心，用于控制回答风格和输出边界。Persona 负责不同角色模式，Guard 负责检查角色漂移和高风险表达，比如“我是律师”“保证胜诉”等。
```

---

## 5. 最短版

```text
M6 解决工具调用；
M7 解决流程编排；
M8 解决记忆；
M9 解决性能和状态恢复；
M10 解决角色一致性和输出约束；
M11 解决法条关系推理。
```
