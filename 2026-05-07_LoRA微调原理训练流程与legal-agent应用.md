# LoRA 微调原理、训练流程与 legal-agent 应用

## 1. ASCII 总览图

```text
LoRA / SFT 微调理解
├── 1. LoRA adapter 是什么
│   ├── 冻结 base model
│   ├── 插入少量可训练参数
│   └── 只训练 adapter，不全量更新大模型
│
├── 2. 为什么不用全量微调
│   ├── 全量微调显存高
│   ├── 训练慢
│   ├── 成本高
│   ├── 容易训坏原模型
│   └── 每个业务版本都要保存完整模型
│
├── 3. LoRA 主要解决什么问题
│   ├── 固定回答格式
│   ├── 固定 persona 风格
│   ├── 固定业务话术
│   ├── 固定拒答边界
│   ├── 固定工具调用习惯
│   └── 减少重复 prompt 规则带来的 token 成本
│
├── 4. LoRA 和 Prompt / RAG / Guard 的关系
│   ├── Prompt：临时告诉模型怎么做
│   ├── LoRA：让模型长期更习惯这么做
│   ├── RAG：提供外部知识
│   └── Guard：检查输出风险
│
├── 5. LoRA / SFT 为什么像监督学习
│   ├── 输入 X：system prompt + user message + 上下文
│   ├── 标签 Y：标准 assistant 回复 / tool_call 轨迹
│   ├── 模型预测 Ŷ：生成 token 序列
│   ├── 计算 loss
│   └── 反向传播，只更新 LoRA adapter
│
├── 6. legal-agent 里适合训练什么
│   ├── 回答结构
│   ├── 工具调用习惯
│   ├── Persona 风格
│   ├── 拒答边界
│   └── JSON 输出格式
│
└── 7. legal-agent 里不适合训练什么
    ├── 最新法律条文
    ├── 全部判例
    ├── 实时政策
    ├── 用户历史上下文
    └── 大量外部知识
```

---

## 2. LoRA adapter 是什么

LoRA adapter 的目的很简单：

```text
不想动整个大模型，
但又想让模型学一点业务习惯。
```

所以 LoRA 相当于给基础模型加了一个“小外挂”。

它的基本思路是：

```text
原始大模型参数冻结，不动；
额外插入一小块可训练参数；
训练时只更新这一小块参数。
```

结构可以理解为：

```text
Base Model 原始权重
  └── 冻结，不更新

LoRA Adapter
  └── 新增小参数，训练它
```

训练后：

```text
原模型还是原模型；
LoRA adapter 学到了业务风格、格式、工具调用习惯和安全边界。
```

推理时：

```text
Base Model + LoRA Adapter = 带业务习惯的模型
```

---

## 3. 为什么不直接全量微调

因为大模型参数太多。

比如一个 7B 模型：

```text
7B = 70 亿参数
```

如果全量微调，就要更新大量参数，会带来这些问题：

```text
显存高
训练慢
成本高
容易训坏
保存模型很大
每个业务版本都要存一整份模型
```

LoRA 的好处是：

```text
1. 省显存
   只训练少量参数，不用更新整个大模型。

2. 省训练成本
   比全量微调便宜很多。

3. 保存文件小
   只保存 adapter，不用保存完整模型。

4. 可插拔
   同一个 base model 可以挂不同 adapter。

5. 不容易破坏原模型主体
   原模型参数冻结，风险比全量微调低。

6. 方便多业务版本
   一个法律 adapter，一个客服 adapter，一个企业风格 adapter。
```

例如：

```text
Base Model：Qwen2.5-7B

Adapter A：legal-agent-lora
Adapter B：customer-service-lora
Adapter C：enterprise-style-lora
```

同一个基础模型，挂不同 adapter，就可以表现出不同任务习惯。

---

## 4. LoRA 主要解决什么 token 成本问题

LoRA 不是直接解决“上下文太长”这个问题。

它主要解决的是：

```text
高频重复规则每次都塞进 prompt，导致 token 成本高。
```

也就是说，它省的是这类内容：

```text
固定回答格式
固定 persona 风格
固定业务话术
固定拒答边界
固定工具调用习惯
固定 JSON 输出格式
少量 few-shot 示例
```

原来每次 prompt 里都要写：

```text
请先给结论；
再给依据；
再列证据；
最后风险提醒；
不要说保证胜诉；
不要自称律师；
具体法条问题调用 get_law_article；
模糊问题调用 legal_search；
下面给你 5 个示例……
```

这些每次都会消耗 token。

LoRA adapter 可以把一部分稳定模式学进去，于是 prompt 可以缩短成：

```text
按 legal-agent 风格回答。
```

但注意边界：

```text
LoRA 不能替代 RAG。
LoRA 不能替代 Memory。
LoRA 不能把所有历史上下文都压掉。
LoRA 不适合存大量新知识、法条、案例、实时信息。
```

如果上下文长是因为：

```text
检索出来的法条很多
历史对话很多
案例材料很长
用户上传文档很大
```

那应该用：

```text
RAG 重排
chunk 压缩
Summary Memory
上下文裁剪
检索 topK 控制
长文摘要
```

不是靠 LoRA。

最准确的一句话：

```text
LoRA 主要减少“重复提示词规则”的 token 消耗，
不是减少“真实业务上下文”的 token 消耗。
```

---

## 5. LoRA 和 Prompt / RAG / Guard 的关系

```text
Prompt：
每次调用都临时告诉模型怎么做。

LoRA adapter：
让模型长期更习惯这么做。

RAG：
回答前检索外部知识，给模型看依据。

Guard：
回答后检查风险表达和角色漂移。
```

对比表：

| 方式 | 作用 | 缺点 |
|---|---|---|
| Prompt | 快速控制格式和风格 | 每次都占 token，长了会贵 |
| LoRA adapter | 固化高频行为模式 | 需要数据、训练、评测 |
| RAG | 提供外部知识 | 依赖检索质量 |
| Memory | 提供用户历史和上下文 | 需要控制长度和误记 |
| Guard | 检查输出风险 | 本身不教模型怎么答 |

正确策略是：

```text
先 Prompt 调试；
稳定后整理成训练数据；
再用 LoRA 固化高频模式。
```

也可以压成一句：

```text
知识放 RAG；
历史放 Memory；
边界靠 Guard；
行为模式可以用 Prompt 先调，稳定后再用 LoRA 固化。
```

---

## 6. LoRA / SFT 为什么可以类比监督学习

LoRA / SFT 微调的训练流程，本质上可以类比监督学习：

```text
给定输入 X 和标准答案 Y，
模型预测 Ŷ，
计算 Ŷ 和 Y 的 loss，
再反向传播更新参数。
```

只是这里的“标签”不是一个类别，而是一整段 assistant 回复，或者一段工具调用轨迹。

传统监督学习：

```text
传统监督学习
├── 输入 X：图片 / 文本 / 特征
├── 标签 Y：猫 / 狗 / 分类标签
├── 模型预测 Ŷ
├── 计算 loss
└── 更新模型参数
```

LoRA / SFT 微调：

```text
LoRA / SFT 微调
├── 输入 X：system prompt + user message + 上下文
├── 标签 Y：标准 assistant 回复 / tool_call 轨迹
├── 模型预测 Ŷ：生成的 token 序列
├── 计算 loss：预测 token 和标准 token 的差距
└── 更新 LoRA adapter 参数
```

放到 legal-agent 里：

```text
训练样本
├── 输入：
│   ├── system：法律助手规则
│   ├── user：用户法律问题
│   └── 可选：工具描述 / 上下文
│
└── 标准答案：
    ├── 标准法律咨询回复
    ├── 标准 JSON 输出
    ├── 标准 tool_call
    └── 标准拒答 / 风险提醒
```

---

## 7. LoRA 微调完整训练流程

训练过程可以按监督学习流程理解：

```text
1. 准备高质量样本
   user 问题 + 标准 assistant 回复

2. tokenizer 编码
   文本 → token id

3. base model + LoRA adapter 前向传播
   模型根据输入预测 assistant 回复

4. 计算 loss
   预测 token 和标准答案 token 比较

5. 反向传播
   loss 传回模型

6. 更新参数
   全量微调：更新大模型参数
   LoRA 微调：只更新 adapter 参数

7. 重复多轮 epoch
   让模型越来越接近标准回答模式

8. 保存 LoRA adapter
   推理时 base model + adapter 一起加载
```

整体图：

```text
训练数据
  ├── 用户问题
  └── 标准回答
        ↓
tokenizer 编码
        ↓
base model + LoRA adapter
        ↓
模型预测 assistant 回复
        ↓
和标准回答计算 loss
        ↓
反向传播
        ↓
只更新 LoRA adapter
        ↓
重复多轮
        ↓
保存 adapter
        ↓
推理时加载 base model + adapter
```

核心区别：

```text
全量微调：
更新整个模型参数。

LoRA 微调：
冻结 base model，只更新 adapter 参数。
```

---

## 8. legal-agent 中适合用 LoRA 训练什么

LoRA 不适合用来“背知识”，更适合训练模型的行为模式。

对 legal-agent 来说，它适合训练：

```text
回答结构
固定格式
Persona 风格
拒答边界
工具调用习惯
JSON 输出格式
```

例如：

```text
具体法条问题 → get_law_article
模糊法律问题 → legal_search
法条关系问题 → get_related_articles
风险请求 → 拒答并引导合法路径
```

加了 legal-agent 的 LoRA adapter 后，模型应该更倾向于：

```text
【核心结论】
你可以考虑劳动仲裁。

【证据清单】
工资流水、考勤记录、工作聊天记录、辞退通知。

【处理路径】
先固定证据，再协商，协商无效后申请仲裁。

【风险提醒】
不能仅凭描述判断一定胜诉。
```

它不是让模型突然知道所有法律知识，而是让模型更稳定地按项目要求的格式、边界和工具习惯输出。

---

## 9. legal-agent 中不适合用 LoRA 解决什么

不适合靠 LoRA 解决：

```text
最新法律条文
全部判例
公司文档
实时政策
用户长期历史上下文
大量外部知识
```

这些应该继续放在：

```text
RAG
数据库
知识图谱
Memory
Tool
```

尤其是法律场景：

```text
法条、案例、司法解释、政策更新
```

更适合用 RAG / 数据库 / 知识图谱，而不是微调进模型。

原因是：

```text
1. 知识会更新
2. 法律依据需要可追溯
3. 微调不保证精确记住某条知识
4. 微调后的知识不容易定位来源
```

所以原则是：

```text
知识放 RAG；
行为放 LoRA。
```

---

## 10. LoRA 数据应该怎么准备

适合准备的数据：

```text
1. 法律咨询标准回答数据
   user query → structured answer

2. 工具调用轨迹数据
   user query → tool_call → tool_result → final answer

3. 拒答 / 安全边界数据
   risky query → safe response

4. Persona 风格数据
   same query → strict / friendly / enterprise / litigation

5. JSON 结构化输出数据
   user query → fixed JSON schema
```

不适合直接作为 LoRA 训练数据：

```text
原始法条全文
原始判例全文
未清洗聊天记录
网页复制内容
错误回答
隐私信息未脱敏的数据
```

高质量样本应该满足：

```text
用户问题真实
场景明确
回答结构稳定
法律依据准确
没有过度承诺
有行动建议
有风险提醒
格式统一
隐私已脱敏
不把不确定说成确定
```

---

## 11. 训练后怎么评估

不要只看 loss。

loss 降了，不代表项目一定变好。

训练后要看：

```text
1. 回答格式是否更稳定
2. 是否减少长 prompt
3. 是否更少说高风险话
4. 工具调用是否更准
5. 是否损伤原模型通用能力
6. RAG 问答是否仍然引用正确依据
7. 同一测试集上，微调前后对比
```

评估方式：

```text
base model + prompt
vs
base model + LoRA + 短 prompt
```

如果 LoRA 没有明显省 token 或提升稳定性，就不应该上生产。

---

## 12. 面试表达版

```text
LoRA adapter 的核心价值是参数高效微调。它不是全量修改整个大模型，而是冻结基础模型的大部分参数，只在指定线性层旁边加入少量可训练的低秩矩阵。训练时只更新 adapter，不更新原模型主体。这样可以用较低显存和成本，让通用模型适配业务任务。

在我的 legal-agent 项目里，LoRA 不适合用来替代 RAG 或 Memory。法条、案例、法规更新、用户历史上下文这些仍然应该通过 RAG、数据库、知识图谱和 Memory 提供。LoRA 更适合固化高频、稳定的行为模式，比如固定回答结构、Persona 风格、拒答边界、JSON 输出格式和工具调用习惯。

它的一个收益是降低长期 prompt 成本。如果每次都在 system prompt 里写大量固定规则、few-shot 示例和工具选择说明，会造成 token 消耗和延迟增加。LoRA 可以把一部分稳定规则训练进 adapter，让模型长期更习惯这种输出方式，从而缩短 prompt，提高输出稳定性。

训练流程上，LoRA / SFT 可以类比监督学习。输入是 system prompt、user message 和上下文，标签是人工整理好的标准回复或标准 tool_call 轨迹。模型先根据输入预测 token，然后和标准答案 token 计算 loss，再反向传播。不同的是，LoRA 冻结 base model，只更新额外插入的 adapter 参数。因此它比全量微调更省显存、更省训练成本，也更方便多业务版本切换。

所以我的策略是：先用 Prompt 调试回答结构、风格和工具规则；如果稳定后发现这些规则高频重复、prompt 太长、成本太高，再整理高质量样本，用 LoRA 固化这些行为模式。知识仍然放 RAG，行为才放 LoRA。
```

---

## 13. 最短版

```text
LoRA adapter 是一个轻量微调外挂。

它冻结 base model，只训练少量 adapter 参数，用较低成本让模型适配业务风格、格式、边界和工具调用习惯。

它不是用来替代 RAG 或 Memory 的。
RAG 负责外部知识；
Memory 负责用户历史；
Guard 负责输出检查；
LoRA 负责固化高频行为模式。

LoRA 可以减少重复 prompt 规则造成的 token 成本，但不能减少真实业务上下文，比如法条、案例、长文档和历史对话。

训练流程可以类比监督学习：
输入 X 是 system prompt + user message + 上下文；
标签 Y 是标准 assistant 回复或 tool_call 轨迹；
模型预测 Ŷ；
计算 loss；
反向传播；
只更新 LoRA adapter。

核心原则：
知识放 RAG，行为放 LoRA。
```
