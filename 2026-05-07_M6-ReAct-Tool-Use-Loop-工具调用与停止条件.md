# M6 ReAct / Tool Use Loop：工具暴露、工具调用与停止条件

## 0. 一句话总览

```text
这版 ReAct 的核心是：程序把 tools=ALL_TOOLS 提供给大模型，大模型基于 messages 上下文判断要不要调用工具；如果返回 tool_calls，程序执行工具并把 tool_result 写回 messages；如果不再返回 tool_calls，而是返回普通 content，程序就认为任务完成并返回最终答案。
```

---

## 1. 简单 ASCII 树状图：先看主干

```text
M6 ReAct / Tool Use Loop
├── 1. 工具怎么给模型？
│   └── tools=ALL_TOOLS
│       ├── 工具名 name
│       ├── 工具说明 description
│       └── 工具参数 parameters
│
├── 2. 模型怎么选工具？
│   └── 根据两类信息判断
│       ├── 当前上下文 messages
│       └── 工具 schema：name / description / parameters
│
├── 3. 程序怎么执行工具？
│   └── dispatch_tool(fn_name, fn_args)
│       ├── fn_name 决定调用哪个工具
│       └── fn_args 决定工具具体查什么 / 算什么
│
├── 4. 工具结果怎么回到模型？
│   └── messages.append(role="tool")
│       ├── tool_call_id 对应前面的工具调用
│       └── content 保存 tool_result
│
└── 5. 怎么判断结束？
    └── 看 msg.tool_calls
        ├── 有 tool_calls → 继续执行工具
        └── 没有 tool_calls → 返回 msg.content 作为最终答案
```

---

## 2. 复杂 ASCII 树状图：完整运行链路

```text
用户问题
  ↓
初始化 messages
  ├── system：告诉模型角色、规则、工具使用方式
  └── user：用户问题
      ↓
进入 for iteration in range(MAX_TOOL_ITERATIONS)
      ↓
调用大模型 API
  ├── model=settings.deepseek_model
  ├── messages=messages
  ├── tools=ALL_TOOLS
  ├── tool_choice="auto"
  └── temperature=0.3
      ↓
得到 response
      ↓
取出 msg = response.choices[0].message
      ↓
判断 msg.tool_calls
  ├── 没有 tool_calls
  │     ↓
  │   模型认为信息足够
  │     ↓
  │   trace.final_reply = msg.content
  │     ↓
  │   return trace
  │
  └── 有 tool_calls
        ↓
      messages.append(msg.model_dump())
        ├── 保存 assistant 的工具调用请求
        └── 注意：这里保存的是“我要调用工具”，不是工具结果
        ↓
      遍历 for tc in msg.tool_calls
        ↓
      解析工具调用
        ├── fn_name = tc.function.name
        │     └── 决定调用哪个工具
        └── fn_args = json.loads(tc.function.arguments)
              └── 决定工具具体参数
        ↓
      执行工具
        └── tool_result = await dispatch_tool(fn_name, fn_args)
              ├── dispatch_tool 不是 OpenAI 方法
              ├── 它是项目里的工具分发函数
              └── 根据工具名找到真实 Python 函数执行
        ↓
      记录 trace
        └── trace.tool_calls.append(...)
              ├── 第几轮
              ├── 工具名
              ├── 参数
              └── 结果预览 result_preview
        ↓
      写回工具结果
        └── messages.append({
              "role": "tool",
              "tool_call_id": tc.id,
              "content": tool_result
            })
        ↓
      下一轮循环
        ↓
      新 messages 再传给模型
        ↓
      模型继续判断：信息够不够？
        ├── 不够 → 继续返回 tool_calls
        └── 够了 → 返回普通 content
```

---

## 3. 核心代码

```python
for iteration in range(MAX_TOOL_ITERATIONS):
    trace.iterations = iteration + 1

    response = await client.chat.completions.create(
        model=settings.deepseek_model,
        messages=messages,
        tools=ALL_TOOLS,
        tool_choice="auto",
        temperature=0.3,
    )
    msg = response.choices[0].message

    # 没有工具调用 → 终止循环，返回最终回答
    if not msg.tool_calls:
        trace.final_reply = msg.content or ""
        return trace

    # 有工具调用 → 先保存 assistant 的 tool_calls 消息
    messages.append(msg.model_dump())

    # 遍历模型这一轮想调用的所有工具
    for tc in msg.tool_calls:
        # 取出工具名
        fn_name = tc.function.name

        # 取出工具参数：JSON 字符串 → Python dict
        fn_args = json.loads(tc.function.arguments)

        # Acting：真正执行工具
        tool_result = await dispatch_tool(fn_name, fn_args)

        # 记录工具调用轨迹，方便调试和评测
        trace.tool_calls.append(
            ToolCallRecord(
                iteration=iteration + 1,
                name=fn_name,
                arguments=fn_args,
                result_preview=tool_result[:200],
            )
        )

        # Observation：把工具结果写回 messages
        messages.append(
            {
                "role": "tool",
                "tool_call_id": tc.id,
                "content": tool_result,
            }
        )
```

---

## 4. 模型怎么知道有哪些工具？

模型不是凭空知道工具的。

关键代码是：

```python
response = await client.chat.completions.create(
    model=settings.deepseek_model,
    messages=messages,
    tools=ALL_TOOLS,
    tool_choice="auto",
    temperature=0.3,
)
```

核心是：

```python
tools=ALL_TOOLS
```

`ALL_TOOLS` 里面提前写好了工具 schema，通常包含：

```text
1. name：工具名
2. description：工具适合解决什么问题
3. parameters：工具需要哪些参数
```

所以程序每一轮都会告诉模型：

```text
你现在可以使用这些工具。
每个工具叫什么。
每个工具适合干什么。
每个工具需要哪些参数。
```

工具能力是通过 `tools` 参数暴露给模型的。

---

## 5. 模型怎么知道该调用哪个工具？

模型根据两部分判断：

```text
1. 当前 messages 上下文
2. tools 里面每个工具的 description 和 parameters
```

例如用户问：

```text
劳动合同法第八十二条是什么？
```

模型看到工具说明里有：

```text
get_law_article：按法律名 + 条款号精确查询条文原文
```

于是模型会判断：

```text
这个问题有具体法律名：劳动合同法
这个问题有具体条号：第八十二条
所以应该调用 get_law_article
```

然后生成工具调用请求：

```json
{
  "name": "get_law_article",
  "arguments": {
    "law_title": "劳动合同法",
    "article_no": "第八十二条"
  }
}
```

注意：

```text
模型不是执行工具。
模型只是生成“工具调用请求”。
真正执行工具的是 Python 程序。
```

真正执行的是：

```python
tool_result = await dispatch_tool(fn_name, fn_args)
```

---

## 6. dispatch_tool 是什么？

`dispatch_tool` 不是 OpenAI 的方法，也不是模型内部的方法。

它是项目里自己写的工具分发函数。

作用是：

```text
工具名字符串 → 项目里的真实 Python 函数
```

例如：

```text
"legal_search"          → legal_search()
"get_law_article"       → get_law_article()
"get_related_articles"  → get_related_articles()
```

所以：

```python
tool_result = await dispatch_tool(fn_name, fn_args)
```

意思是：

```text
根据模型给出的工具名 fn_name，
找到项目里对应的真实函数，
再把 fn_args 作为参数传进去执行，
最后拿到 tool_result。
```

这一步才是 ReAct 里的 Acting。

---

## 7. messages 为什么是核心？

这版 ReAct 的核心不是某个神秘算法，而是维护 `messages` 这个上下文列表。

一开始：

```json
[
  {
    "role": "system",
    "content": "你是法律 Agent，可以调用工具。"
  },
  {
    "role": "user",
    "content": "劳动合同法第八十二条是什么？"
  }
]
```

模型返回工具调用请求后：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_123",
      "type": "function",
      "function": {
        "name": "get_law_article",
        "arguments": "{\"law_title\":\"劳动合同法\",\"article_no\":\"第八十二条\"}"
      }
    }
  ]
}
```

执行：

```python
messages.append(msg.model_dump())
```

此时 messages 里多了一条 assistant 工具调用请求。

工具执行后，程序继续追加：

```json
{
  "role": "tool",
  "tool_call_id": "call_123",
  "content": "《劳动合同法》第八十二条原文是……"
}
```

第二轮传给模型的 messages 就变成：

```text
system
+ user
+ assistant tool_calls
+ tool result
```

模型看到新的上下文后，再判断信息是否足够。

---

## 8. model_dump() 是什么？

```python
messages.append(msg.model_dump())
```

`msg` 是 SDK 返回的 assistant 消息对象。

`model_dump()` 的作用是：

```text
把对象转成 Python dict。
```

为什么要转？

因为 `messages` 里存的是字典格式。比如：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [...]
}
```

所以这句的含义是：

```text
把模型这一轮返回的 assistant 消息对象转成 dict，
再追加到 messages 上下文里。
```

它保存的是：

```text
assistant：我要调用工具，工具名和参数是这些。
```

---

## 9. 为什么要先 append assistant tool_calls，再 append tool result？

顺序必须是：

```text
assistant tool_calls
  ↓
tool result
```

不能反过来。

因为工具结果必须对应某一次工具调用。

正常结构是：

```json
[
  {
    "role": "assistant",
    "content": null,
    "tool_calls": [
      {
        "id": "call_123",
        "function": {
          "name": "get_law_article",
          "arguments": "{\"law_title\":\"劳动合同法\",\"article_no\":\"第八十二条\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "tool_call_id": "call_123",
    "content": "工具返回结果"
  }
]
```

这里：

```text
assistant.tool_calls[0].id = call_123
tool.tool_call_id = call_123
```

它们靠 `call_123` 对应起来。

所以：

```text
assistant tool_calls 表示：模型说“我要调用工具”。
tool message 表示：程序说“工具执行完了，这是结果”。
```

---

## 10. 模型怎么知道什么时候完成？

在这版代码里，停止条件是：

```python
if not msg.tool_calls:
    trace.final_reply = msg.content or ""
    return trace
```

也就是说：

```text
如果模型这一轮没有返回 tool_calls，
而是返回了普通 content，
程序就认为它已经完成了。
```

模型内部的判断逻辑是：

```text
当前 messages 里的信息是否足够回答用户问题？
```

如果不够，它会继续返回 `tool_calls`。

如果够了，它就不返回 `tool_calls`，而是返回最终回答。

程序判断是否完成的方式很简单：

```text
只看 msg.tool_calls 是否为空。
为空就结束。
不为空就执行工具。
```

---

## 11. 关键风险：当前代码相信模型的判断

这里要特别注意：

```text
不是“模型觉得够了就一定真的够了”，而是“当前代码相信模型的判断”。
```

模型可能判断错：

```text
该查法条，它没查，直接答了。
工具结果不完整，它误以为够了。
用户问题需要计算，它没调 Python。
应该继续查关系，它提前结束。
```

所以更成熟的 Agent 会加入额外机制：

```text
1. Router：先判断问题类型
2. Guard：检查回答是否缺依据或是否越界
3. Evaluator：评估答案是否满足需求
4. 强制规则：某类问题必须调用某个工具
5. 最大轮数：防止一直调工具
```

但当前 M6 循环版的核心是：

```text
LLM 决定是否调用工具。
程序执行工具。
LLM 决定是否结束。
```

---

## 12. 详细流程：以“劳动合同法第八十二条是什么？”为例

### 第 1 轮：初始上下文

```json
[
  {
    "role": "system",
    "content": "你是法律 Agent，可以调用工具。"
  },
  {
    "role": "user",
    "content": "劳动合同法第八十二条是什么？"
  }
]
```

模型看到问题后，判断需要查具体法条。

---

### 第 2 步：模型返回 tool_calls

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_123",
      "type": "function",
      "function": {
        "name": "get_law_article",
        "arguments": "{\"law_title\":\"劳动合同法\",\"article_no\":\"第八十二条\"}"
      }
    }
  ]
}
```

意思是：

```text
模型说：我要调用 get_law_article。
参数是：劳动合同法、第八十二条。
```

---

### 第 3 步：程序执行工具

程序解析出：

```text
fn_name = get_law_article

fn_args = {
  "law_title": "劳动合同法",
  "article_no": "第八十二条"
}
```

然后执行：

```python
tool_result = await dispatch_tool(fn_name, fn_args)
```

---

### 第 4 步：工具结果写回上下文

```json
{
  "role": "tool",
  "tool_call_id": "call_123",
  "content": "《劳动合同法》第八十二条原文是……"
}
```

这时 messages 变成：

```text
system
+ user
+ assistant tool_calls
+ tool result
```

---

### 第 5 步：第二轮传给模型

模型看到：

```text
用户问了什么
自己刚才请求查什么
工具返回了什么
```

于是它判断：

```text
信息够了，可以回答。
```

返回：

```json
{
  "role": "assistant",
  "content": "《劳动合同法》第八十二条主要规定了未签订书面劳动合同的二倍工资责任……",
  "tool_calls": null
}
```

代码看到没有 `tool_calls`，结束循环。

---

## 13. 最短理解版

```text
工具不是模型自己知道的，是程序通过 tools=ALL_TOOLS 提供给模型的。

模型根据 messages 上下文和工具 schema 判断要不要调用工具、调用哪个工具、参数是什么。

模型只生成 tool_calls，不真正执行工具。

Python 程序通过 dispatch_tool 执行真实工具，把 tool_result 作为 role="tool" 写回 messages。

下一轮模型基于新的 messages 继续判断信息够不够。

如果还不够，继续返回 tool_calls。

如果够了，就返回普通 content。

代码看到没有 tool_calls，就认为任务完成并返回最终回答。
```

---

## 14. 面试对话版总结

### 面试官：你这个 ReAct 是怎么实现的？

可以回答：

```text
我早期实现的是一个手写版 Tool Use Loop，本质就是 ReAct 的 Reasoning、Acting、Observation 循环。

每一轮我会把当前 messages 上下文、工具列表 ALL_TOOLS、tool_choice="auto" 一起传给大模型。工具列表里通过 schema 描述了每个工具的 name、description 和 parameters，所以模型知道有哪些工具、每个工具适合什么场景、需要哪些参数。
```

### 面试官：模型怎么决定调用哪个工具？

可以回答：

```text
模型会结合当前 messages 上下文和工具 description 来判断。比如用户问“劳动合同法第八十二条是什么”，工具 schema 里 get_law_article 的描述是按法律名和条款号精确查询条文原文，所以模型会生成 get_law_article 的 tool_call，并把 law_title 和 article_no 作为参数传出来。
```

### 面试官：工具是谁真正执行的？

可以回答：

```text
模型并不真正执行工具，它只是返回 tool_calls。我的程序会解析 tool_calls 里的 function.name 和 function.arguments，然后通过 dispatch_tool 根据工具名找到项目里对应的真实 Python 函数，再把参数传进去执行。比如 get_law_article 会被分发到真实的 get_law_article 函数。
```

### 面试官：工具结果怎么回到模型？

可以回答：

```text
工具执行后，我会把 tool_result 包装成 role="tool" 的消息追加回 messages，并用 tool_call_id 对应前面 assistant tool_calls 里的 id。这样下一轮模型就能看到用户问题、自己刚才请求调用的工具，以及工具返回结果，然后继续判断是否还需要调用工具。
```

### 面试官：怎么判断任务完成？

可以回答：

```text
在早期版本里，停止条件主要依赖模型是否继续返回 tool_calls。如果模型认为当前上下文已经足够，它会返回普通 content 而不是 tool_calls。程序看到 msg.tool_calls 为空，就把 msg.content 作为最终答案返回。

这个实现简单直接，但风险是当前代码比较相信模型自己的判断。模型可能误判信息是否足够，所以后续可以通过 Router、Guard、Evaluator 或强制工具调用规则来增强可靠性。
```

### 面试官：这版实现的核心是什么？

可以回答：

```text
核心是维护 messages 这个不断增长的上下文列表。大模型负责基于 messages 和 tools 判断下一步要不要调用工具，Python 程序负责真正执行工具并把结果写回 messages。整个循环不断重复，直到模型不再返回 tool_calls，而是生成最终回答。
```

---

## 15. 最终压缩句

```text
我的 M6 ReAct 实现，本质是一个围绕 messages 的 Tool Use Loop：LLM 根据 messages 和 tools=ALL_TOOLS 判断是否调用工具；如果返回 tool_calls，程序通过 dispatch_tool 执行真实工具并把 tool_result 回填到 messages；下一轮 LLM 基于新的 messages 继续推理；如果不再返回 tool_calls，就把 content 作为最终答案返回。
```
