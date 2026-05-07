# legal-agent 项目检测与评测体系

## 1. 简单 ASCII 树状图

```text
legal-agent 检测 / 评测体系
├── 1. Router 检测
│   ├── 检测 greeting / legal / off_topic 是否分对
│   ├── pytest 自动测试
│   ├── latency 脚本半自动评估
│   └── 指标：分类准确率、延迟、Rule / LLM 一致率
│
├── 2. ReAct / Tool Use 检测
│   ├── 检测模型是否选对工具
│   ├── 检测工具参数是否正确
│   ├── 检测 tool_result 是否写回 messages
│   ├── 自动记录 tool_calls_log
│   └── 是否最优仍需人工 case / LLM judge
│
├── 3. RAG 检索检测
│   ├── 检测正确法条 / chunk 是否被召回
│   ├── 检测排序是否合理
│   ├── 理论指标：Recall@K / Precision@K / MRR / NDCG
│   └── 需要人工标注 query → relevant articles
│
├── 4. Memory 检测
│   ├── 检测 Buffer / Summary / Hard Memory 是否读写成功
│   ├── Redis / PostgreSQL 读写可自动测
│   ├── 记忆注入效果需要多轮观察
│   └── 可用脚本 + 人工判断
│
├── 5. Persona / Guard 检测
│   ├── 检测 persona 风格是否生效
│   ├── 检测是否角色漂移 / 越界表达
│   ├── Guard 触发自动检测
│   └── Persona 效果主要靠脚本输出后人工观察
│
└── 6. 性能检测
    ├── Router 延迟
    ├── LLM 调用耗时
    ├── 工具调用耗时
    ├── QPS / P95 / P99
    └── token 成本 / 资源占用
```

---

## 2. 总体判断

项目里现在不是所有评测都自动化，也不是全部交给大模型。

更准确是：

```text
当前项目评测方式 =
自动化单测
+
脚本半自动评测
+
人工观察
+
少量可用 LLM judge 辅助
```

其中：

```text
Router：自动化程度较高
Guard：自动化程度较高
Persona：半自动观察
Memory：读写可自动测，效果要人工观察
Tool Use：自动记录日志，是否最优要人工 / LLM judge
RAG：理论指标明确，但完整指标体系依赖人工标注集
```

不能简单说：

```text
我的项目都用 Recall / Precision / NDCG 评测
```

更准确的说法是：

```text
RAG 检索层适合用 Recall@K、Precision@K、MRR、NDCG；
但整个 Agent 项目还需要 Router、Tool Use、Memory、Guard、Persona 和性能评测。
```

---

## 3. Router 检测

### 3.1 检测目标

Router 主要检测：

```text
1. query 是否分对类别
2. 分类延迟是否可接受
```

分类包括：

```text
greeting
legal
off_topic
```

例如：

```text
你好 → greeting
我被公司辞退了 → legal
民法典第577条 → legal
今天天气怎么样 → off_topic
推荐电影 → off_topic
```

### 3.2 自动化部分

Router 有 pytest 单元测试。

它可以自动检查：

```text
规则样例是否分类正确
CascadeRouter 是否优先走规则
明确 query 是否不调用 LLM
模糊 query 是否允许升级 LLM
```

这类属于：

```text
自动化测试
```

也就是一跑测试就能知道是否通过。

### 3.3 半自动部分

`m9_router_latency.py` 属于脚本评测。

它会对比：

```text
RuleBasedRouter
LLMRouter
```

主要观察：

```text
分类结果是否一致
LLMRouter 延迟
RuleRouter 延迟
平均耗时
```

这类不是严格自动判分，更像：

```text
跑脚本
↓
看输出
↓
分析哪里不一致
```

### 3.4 Router 指标

```text
Accuracy：分类准确率
Latency：分类延迟
Rule / LLM 一致率
是否减少不必要 LLM 调用
```

---

## 4. ReAct / Tool Use 检测

### 4.1 检测目标

ReAct 工具调用层主要检测：

```text
1. 是否应该调用工具
2. 是否调用了正确工具
3. 工具参数是否正确
4. 工具结果是否写回 messages
5. 是否过早结束
6. 是否陷入过多工具调用
```

例如：

```text
用户问：劳动合同法第八十二条是什么？
理想工具：get_law_article

用户问：被辞退没签合同怎么办？
理想工具：legal_search

用户问：民法典第577条和哪些条文有关？
理想工具：get_related_articles
```

### 4.2 当前项目怎么做

当前项目会自动记录：

```text
调用了哪个工具
传了什么参数
工具返回结果预览
调用发生在第几轮
平均调用几轮
```

也就是：

```text
tool_calls_log
trace.tool_calls
result_preview
```

这些日志能帮助分析：

```text
模型是否选错工具
参数是否抽错
是否重复调用工具
是否提前结束
```

### 4.3 哪些还不能完全自动

“工具选得是否最优”目前不是完全自动。

因为很多问题不是只有一个绝对工具路径。

比如：

```text
我被公司辞退了，没签合同怎么办？
```

模型可能：

```text
先 legal_search
再 get_law_article
或者直接 get_law_article
```

哪个最优，需要：

```text
人工 case
规则测试集
LLM judge
```

所以 Tool Use 当前更准确说：

```text
调用过程自动记录；
调用是否最优，需要人工或 LLM 辅助判断。
```

### 4.4 Tool Use 指标

```text
工具选择准确率
参数正确率
平均工具调用次数
平均 ReAct 迭代轮数
任务成功率
过早结束率
无效工具调用率
```

---

## 5. RAG 检索检测

### 5.1 RAG 才主要用 Recall / Precision / MRR / NDCG

召回率、精确率、NDCG 主要属于 RAG / 搜索 / reranker 评测。

不是所有 Agent 模块都用这些指标。

### 5.2 RAG 检测目标

RAG 检索层主要检测：

```text
正确法条有没有被召回
相关 chunk 有没有进入 topK
相关结果是否排在前面
reranker 排序是否有效
```

例如用户问：

```text
未签劳动合同能不能主张二倍工资？
```

理想检索结果应该包含：

```text
《劳动合同法》第八十二条
```

然后可以计算：

```text
Recall@5：前 5 个结果有没有召回正确法条
Precision@5：前 5 个结果里有多少是真的相关
MRR：第一个正确结果排在第几
NDCG@5：整体排序质量怎么样
Hit Rate：前 K 个结果里是否命中正确答案
```

### 5.3 关键问题：需要人工标注集

RAG 指标不是系统凭空自动算出来的。

要算 Recall / Precision / MRR / NDCG，必须先有标注数据：

```json
{
  "query": "未签劳动合同能不能主张二倍工资？",
  "relevant_articles": ["劳动合同法第八十二条"]
}
```

没有这个标注集，就只能人工观察：

```text
检索结果看起来对不对
正确法条有没有出现
排在第几
```

所以项目当前更准确说：

```text
RAG 检索能力已经有；
但严格 Recall / NDCG 评测体系还需要补人工标注集。
```

### 5.4 RAG 指标

```text
Recall@K
Precision@K
MRR
NDCG@K
Hit Rate
Context Relevance
Answer Faithfulness
```

---

## 6. Memory 检测

### 6.1 检测目标

Memory 主要检测两类东西：

```text
1. 存储读写是否正常
2. 记忆注入是否真的改善多轮回答
```

第一类可以自动测。

第二类通常需要人工观察。

### 6.2 可自动测试的部分

比如：

```text
Buffer 是否写入 Redis
Summary 是否写入 PostgreSQL
Hard Memory 是否 upsert
record_turn 是否触发压缩
inject_memory_into_messages 是否拼出正确 messages
```

这些都可以写 pytest 或脚本自动检查。

### 6.3 需要人工观察的部分

记忆效果不是只看数据库有没有写进去。

还要看：

```text
第二轮回答有没有使用第一轮信息
用户画像是否被正确注入
有没有误记
有没有漏记
有没有把旧信息污染当前回答
```

例如：

```text
Turn 1：我是上海的中学老师，30岁
等待异步实体抽取
Turn 2：我能问个法律问题吗？涉及孩子抚养权那种
```

这时要观察：

```text
第二轮是否体现了“上海 / 中学老师 / 30岁”等用户画像
```

所以 Memory 是：

```text
读写逻辑可以自动测；
记忆效果需要多轮脚本 + 人工判断。
```

### 6.4 Memory 指标

```text
记忆写入成功率
记忆读取成功率
事实保留率
误记率
漏记率
多轮一致性
记忆注入后回答质量
```

---

## 7. Persona / Guard 检测

### 7.1 Persona 检测

M10 的 Persona 检测主要看：

```text
不同 persona 下回答风格是否不同
default / strict / friendly / enterprise / litigation 是否符合预期
用户画像注入是否生效
```

这个可以用脚本批量跑。

但结果通常需要人工看。

因为：

```text
friendly 是否真的温和
strict 是否真的严谨
enterprise 是否真的商务
```

这些不适合只靠硬规则判断。

### 7.2 Guard 检测

Guard 自动化程度更高。

它可以自动检测：

```text
是否角色漂移
是否出现高风险表达
是否命中 forbidden phrases
severity 是什么级别
triggered_phrases 是什么
```

比如检测：

```text
我是律师
保证胜诉
规避法律
忽略之前指令
```

这类可以自动判。

### 7.3 Persona / Guard 指标

```text
drift 是否触发
severity
triggered_phrases
高风险表达命中率
persona 风格一致性
用户画像注入是否生效
```

---

## 8. LLM judge 的位置

LLM judge 可以用，但不是现在所有东西都靠它。

LLM judge 适合辅助判断：

```text
回答是否覆盖用户问题
回答是否依据充分
是否存在幻觉
是否违反 persona
是否应该继续调用工具
工具选择是否合理
```

但它也会错。

所以更稳的是：

```text
自动指标
+
人工标注
+
LLM judge 辅助
```

而不是完全相信大模型。

更准确的项目状态是：

```text
自动化测试覆盖基础逻辑；
脚本评测覆盖延迟和效果观察；
人工标注用于 RAG 严格评测；
LLM judge 可以作为后续增强。
```

---

## 9. 不同模块对应的检测方式

| 模块 | 检测什么 | 当前方式 | 是否全自动 |
|---|---|---|---|
| Router | greeting / legal / off_topic 分类 | pytest + latency 脚本 | 部分自动 |
| ReAct / Tool Use | 是否选对工具、参数是否正确 | tool_calls_log + 人工 case | 半自动 |
| RAG 检索 | 正确法条是否召回、排序是否靠前 | 需要标注集计算指标 | 还需补全 |
| Reranker | 相关结果是否排前 | MRR / NDCG | 依赖标注集 |
| Memory | 记忆读写和注入效果 | 读写可自动测，效果人工看 | 部分自动 |
| Persona | 风格是否符合预期 | compare_personas 脚本 + 人工观察 | 半自动 |
| Guard | 角色漂移 / 风险表达 | 规则检测 / triggered_phrases | 自动 |
| 性能 | 延迟、QPS、资源占用 | 脚本 / 监控 | 可自动 |

---

## 10. 项目当前状态总结

```text
Router：
已有 pytest 自动测试，也有 latency 脚本半自动评估。

RAG：
理论上应该用 Recall@K、Precision@K、MRR、NDCG，但需要人工标注 query 和相关法条。目前更偏功能验证，完整评测体系还需要补标注集。

ReAct / Tool Use：
系统会自动记录 tool_calls、参数和结果预览，但工具选择是否最优还需要人工测试集或 LLM judge。

Memory：
读写逻辑可以自动测；记忆注入效果通常需要多轮脚本观察。

Persona / Guard：
Guard 关键词和漂移检测是自动的；Persona 风格效果主要靠脚本输出后人工观察，也可以后续接 LLM judge。

性能：
Router 延迟、LLM 调用耗时、工具调用耗时可以脚本自动统计。
```

---

## 11. 面试表达版

```text
我的项目里评测不是单一方式，而是分模块处理。

Router 层有自动化单测，比如用 pytest 检查 greeting、legal、off_topic 分类是否正确，同时也有脚本对比 Rule Router 和 LLM Router 的延迟。

RAG 层如果要严格评估，需要构建人工标注集，再计算 Recall@K、Precision@K、MRR、NDCG 这些检索和排序指标。目前这部分更多是功能验证，完整指标体系还需要补标注集。

ReAct 工具调用层会自动记录 tool_calls、工具参数和 result_preview，用来分析模型是否调用了正确工具。但工具选择是否最优，当前更多依赖人工 case 检查，后续可以引入 LLM judge 或标准测试集。

Memory 层可以自动测试 Redis 和 PostgreSQL 的读写，但记忆注入是否真的提升多轮回答效果，需要通过多轮脚本和人工观察验证。

Persona / Guard 层里，Guard 的漂移检测是自动的，比如检测 forbidden phrases、severity 和 triggered_phrases；Persona 风格是否符合预期，目前通过 compare_personas 脚本跑不同模式后人工观察。

所以项目里有自动化测试，也有脚本化评测，但严格的 RAG 指标和最终答案质量评估还需要人工标注集或 LLM-as-a-judge 进一步完善。
```

---

## 12. 最短版

```text
不是所有结果检测都用 Recall / Precision / NDCG。

RAG 检索层才主要用 Recall@K、Precision@K、MRR、NDCG，而且需要人工标注集。

Router 看分类准确率和延迟，有 pytest 自动测试和 latency 脚本。

ReAct 看工具选择是否正确、参数是否正确、调用轮数是否合理，目前靠日志 + 人工 case。

Memory 看记忆有没有正确注入和更新，读写可自动测，效果要多轮观察。

Guard 看是否角色漂移、是否出现高风险表达，这部分可以自动检测。

Persona 风格主要靠脚本输出后人工观察。

所以现在是：自动测试 + 脚本评测 + 人工观察，后续可以加 LLM judge 和标注集。
```
