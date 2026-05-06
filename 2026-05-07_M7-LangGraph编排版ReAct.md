# M7：LangGraph 编排版 ReAct

## 1. M7 的核心变化

M6 是手写循环：

```text
for 循环
  ↓
调大模型
  ↓
判断 tool_calls
  ↓
执行工具
  ↓
工具结果写回 messages
  ↓
下一轮继续 / 返回最终答案
```

M7 把这套流程改成 LangGraph 图结构：

```text
START
  ↓
thinker_node
  ↓
route_after_thinker
  ├── 有 tool_calls → actor_node → thinker_node
  └── 没有 tool_calls → END → final_reply
```

一句话：

```text
M7 没有改变 ReAct 的逻辑，而是把 M6 的手写循环拆成了 LangGraph 的节点、状态和边。
```

---

## 2. M7 的完整流程图

```text
START
  ↓
thinker_node
  ├── 调大模型
  ├── 传 messages
  ├── 传 tools=ALL_TOOLS
  └── 得到 assistant message
        ↓
route_after_thinker
  ├── 有 tool_calls
  │     ↓
  │   actor_node
  │     ├── 解析工具名
  │     ├── 解析工具参数
  │     ├── dispatch_tool 执行工具
  │     ├── tool_result 写回 messages
  │     └── 记录 tool_calls_log
  │          ↓
  │       回到 thinker_node
  │
  └── 没有 tool_calls
        ↓
       END
        ↓
     final_reply
```

---

## 3. 各部分分别做什么

| 部分 | 作用 | 对应 ReAct |
|---|---|---|
| `ReactState` | 保存 `messages`、`iteration`、`tool_calls_log` | 状态管理 |
| `thinker_node` | 调用大模型，让模型判断下一步 | Reasoning |
| `route_after_thinker` | 判断有没有 `tool_calls`，决定去 `actor` 还是 `END` | 路由判断 |
| `actor_node` | 执行工具，把工具结果写回 `messages` | Acting + Observation |
| `StateGraph` | 用节点和边组织整个流程 | 流程编排 |
| `END` | 没有工具调用时结束流程 | Final Answer |

---

## 4. M7 的拆分方式

M6 的逻辑原来都在一个循环里：

```text
调模型
判断 tool_calls
执行工具
写回 messages
记录日志
判断是否结束
```

M7 拆成：

```text
thinker_node
  └── 只负责调大模型

route_after_thinker
  └── 只负责判断下一步走 actor 还是 END

actor_node
  └── 只负责执行工具、写回结果、记录日志

ReactState
  └── 只负责保存状态

StateGraph
  └── 只负责控制节点流转
```

这样每一块职责更清楚。

---

## 5. 拆分体现在哪里

### 5.1 推理和工具执行分开

M6 里，调模型和执行工具都写在同一个循环里。

M7 拆成：

```text
thinker_node：负责推理
actor_node：负责执行工具
```

也就是：

```text
模型负责想
程序负责干
```

---

### 5.2 路由判断单独拆出来

M6 里是直接写：

```python
if not msg.tool_calls:
    return trace
```

M7 拆成：

```text
route_after_thinker
```

它只判断：

```text
最后一条 assistant message 有没有 tool_calls？
```

结果：

```text
有 tool_calls → actor_node
没有 tool_calls → END
```

---

### 5.3 状态集中管理

M6 里状态散落在变量中：

```text
messages
iteration
trace.tool_calls
```

M7 统一放进：

```text
ReactState
```

也就是：

```text
messages：上下文
iteration：当前轮数
tool_calls_log：工具调用日志
```

节点只读取和更新状态，不再自己到处维护变量。

---

### 5.4 控制流显式化

M6 的控制流藏在循环和 if 里。

M7 用 LangGraph 显式写出来：

```text
START → thinker
thinker → actor 或 END
actor → thinker
```

这样流程不是靠读一整段循环代码猜出来，而是直接由图结构表达出来。

---

### 5.5 后续扩展更方便

M7 拆开后，后面接新能力更容易：

```text
接 Memory：
  在 run_react_agent 里通过 initial_messages 注入上下文

接 Checkpoint：
  给 graph.compile 加 checkpointer

加新工具：
  改 ALL_TOOLS 和 dispatcher

加新节点：
  在 StateGraph 里继续 add_node / add_edge
```

如果还停留在 M6 的大循环里，这些东西都会继续往一个函数里塞，最后会变成难维护的大函数。

---

## 6. 面试表达版

```text
M7 的核心是把 M6 手写循环版 ReAct 重构成 LangGraph 图结构。

M6 里调模型、判断 tool_calls、执行工具、写回 messages、记录日志和判断结束都写在一个 for 循环里。M7 我把这套流程拆成 thinker_node、actor_node 和 route_after_thinker。

thinker_node 负责调用大模型做 Reasoning，把 messages 和 tools=ALL_TOOLS 传给模型，让模型判断是否需要调用工具。

route_after_thinker 负责检查最后一条 assistant message 里有没有 tool_calls。如果有，就进入 actor_node；如果没有，就进入 END。

actor_node 负责解析 tool_calls 里的工具名和参数，通过 dispatch_tool 执行真实工具，并把 tool_result 作为 role="tool" 的消息写回 messages，同时记录 tool_calls_log。actor 执行完后再回到 thinker_node，让模型基于新的上下文继续推理。

所以 M7 的价值不是改变 ReAct 思想，而是把原来写在循环里的控制流升级成 LangGraph 的状态图编排，让推理、工具执行、路由判断、状态管理和流程控制分开，后续接 Memory、Checkpoint 和更多工具更清晰。
```

---

## 7. 最短版

```text
M7 就是把 M6 的手写 ReAct 循环拆成 LangGraph 图结构：thinker 负责调模型推理，route 判断有没有 tool_calls，actor 负责执行工具并把结果写回 messages。actor 执行完再回 thinker，没有 tool_calls 就进入 END 返回 final_reply。
```
