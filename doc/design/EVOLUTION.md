# Architecture Evolution Log (架构演进日志)

本文档记录 **AIOps-Intelligent-RCA** 项目从 0 到 1 的核心架构决策、设计权衡（Trade-offs）以及迭代思考过程。

---

## 📅 2025-12-11: Phase 1 - 核心架构确立与“反事实”机制引入

### 1. Context (背景与痛点)

在早期的 POC (概念验证) 阶段，直接使用 LLM 分析告警日志存在两个致命问题：

* **严重的幻觉 (Hallucination):** 模型倾向于顺着日志的字面意思“猜”结论，而不是查证。
* **实时查询导致benchmark困难**无法沉淀案例，全量备份监控数据困难

### 2. Key Decisions (关键决策)

#### A. 引入主动反事实验证 (Active Counterfactual Verification)

* **设计:** 放弃单向的 Chain-of-Thought (CoT)，采用 `Hypothesis` -> `Counterfactual Reasoning` -> `Verification` 的闭环结构。
* **逻辑:** "如果 X 是根因，那么指标 Y 必然异常"。通过这一逻辑强制 Agent 调用工具查证。
* **价值:** 将根因分析从“文本生成任务”转变为“科学实验任务”，大幅降低误报率。

#### B. 确立 Retrieval-as-a-Tool (RaaT) 与数据预处理

* **设计:** 查询聚合后的关键指标日志等上下文数据作为原始数据存储，加工后再使用RaaT替代手动开发tool减少开发工作量
* **优化:** 实施 **Data Pre-collection (数据预存)** 策略。
  * 并不让 Agent 实时去拼写 PromQL（容易出错）。
  * 而是预先聚合关键指标（错误率、P99、饱和度）并存储为结构化上下文，供 Agent 快速检索。
* **价值:** 实现了 Agent 决策层与底层数据设施的解耦，提升了推理速度，并让过程可复现，方便后续prompt和流程优化

---

## 📅 2025-12-11: Phase 2 - 解决“断链”与微服务雪崩难题

### 1. Problem (新发现的问题)

在模拟 **“线程池满” (Thread Pool Exhaustion)** 等级联故障场景时，发现传统的基于 Trace 的分析失效了。

* **现象:** 上游 A 调用 B 超时，Trace 在 A 处中断。
* **盲区:** 真正的根因（下游 C 变慢）因为没有生成完整的 Trace 跨度，导致 Agent 无法顺着 Trace 找到 C。

### 2. Architectural Upgrade (架构升级)

#### A. 提出“全局异常交集”逻辑 (Global Anomaly Intersection)

* **设计:** 从“单兵追踪”升级为“全景侦查”。
  * **Input 1:** 实时维护一份“全局嫌疑人名单” (Global Suspicious List)，包含全公司范围内 P99 或错误率突增的 Top N 应用。
  * **Input 2:** 故障入口服务的下游依赖关系（基于历史数据或轻量级拓扑）。
* **逻辑:**`Root Cause Candidates = Intersection(Global_Anomalies, Downstream_Dependencies)`。
* **价值:** 即使 Trace 断裂，Agent 也能通过“时间相关性”和“拓扑相关性”的交集，精准锁定没有出现在 Trace 里的下游故障节点。

#### B. 存储策略调整 (Sparse Topology)

* **设计:** 为了支持上述逻辑且不引入重型 CMDB，决定只存储 ​**“异常边” (Abnormal Edges)**​。即只记录那些发生过高耗时或错误的调用关系。
* **价值:** 极大降低了数据存储成本，去除了正常链路的噪音。

---

## 📅 2025-12-15: Phase 3 - Retrieval-as-a-Tool 的范式转移：从 SQL 到文件系统隐喻

### 1. Context (背景与痛点)

**在落地** **Retrieval-as-a-Tool (RaaT)** **机制时，需要让 Agent 从预处理好的“灾难现场快照” (JSON 格式) 中提取信息。**
初期尝试引入 **DuckDB** **作为中间层，让 LLM 编写 SQL 进行查询，但遭遇了显著的“ROI”问题：**

* **SQL 生成脆性:** **LLM 在处理简单的** **SELECT \*** **时表现尚可，但在涉及 JSON 字段提取和嵌套查询时，错误率显著升高。**
* **杀鸡用牛刀:** **面对的数据对象已经是经过清洗的、行数有限的 Context JSON，引入数据库引擎显得过于厚重，且并未带来预期的检索精度提升。**

### 2. Key Decisions (关键决策)

#### A. 转向“文件系统隐喻”交互 (File System Metaphor Interface)

* **设计:** **放弃 SQL 接口，重构为模拟 Linux/ZooKeeper 的文件系统交互模式。**
* **工具集:** **精简为 4 个原子工具：**
  
  * **ls**: 探索当前上下文结构（看目录）。
  * **cat**: 读取具体的数据切片（读文件）。
  * **grep**: 跨维度的关键词搜索（如全局搜索 TraceID）。
  * **guide**: 获取当前层级数据的元信息与分析建议。
* **价值:** **利用 LLM 在预训练阶段对 Shell/Linux 环境的深刻直觉（Native Intuition），降低了 Tool Use 的学习成本，比生成的 SQL 更鲁棒。**

#### B. 引入动态导航机制 (The "Guide" Pattern)

* **灵感:** **参考 Anthropic 最新的 Agent Skills 设计理念，强调 Context 与 Data 的分离。**
* **设计:** **实现** **"Guide/README"** **机制。**
  
  * **Agent 进入某个数据目录时，先读取** **guide**。
  * **guide** **仅告知“这里有什么数据”以及“建议怎么用”，而不直接 dump 数据。**
* ​**价值:** **实现了上下文的** **按需加载 (Lazy Loading)**。Agent 不再会被海量 Context 淹没，而是像人类专家一样，先看目录索引，再按需调取细节。这极大节省了 Token 开销，并减少了无关信息对推理的干扰。

## 📅 2025-12-16: Phase 3.5 - 观测层升级：从“原始数据搬运”到“语义化渲染”

### 1. Context (背景与痛点)

**在实现了基础的** **cat** **工具后，我们发现 Agent 在处理时序数据（Metrics）时面临两难：**

* **数值盲区 (Numerical Blindness):** **LLM 难以仅凭一串浮点数（如** **[0.1, 0.2, ... 99.0]**）敏锐地感知趋势变化，且容易算错统计值。
* **Token 效率低:** **简单的 Top-N 时序数据平铺会消耗大量 Token，且分散了 Agent 的注意力。**
* **日志、告警、变更等原始json信息密度低，含有大量无用tag，消耗大量 Token，且分散了 Agent 的注意力**

### 2. Key Decisions (关键决策)

#### A. 确立 "Rich-Observation" 交互范式

* **设计:** **拒绝简单的** **file.read()**。确立 **Tool Should Be Smart** **的原则，工具不仅负责取数，更负责\*\*“渲染 (Rendering)”\*\*。**
* **实现:** **在** **cat** **工具中引入** **ASCII Sparklines (字符趋势图)** **技术。**
  
  * **将** **[10, 12, ..., 98]** **渲染为** **▂▃▄▆██**。
  * **价值:** **利用 LLM 强大的视觉/形状语义理解能力，替代脆弱的数值推理能力，实现“一眼看穿”故障趋势。**

#### B. 采用“格式化器注册”模式 (Formatter Registry Pattern)

* **问题:** **随着 Log、Trace、Metrics 多种数据类型的引入，单一的** **cat** **函数逻辑变得臃肿。**
* ​**架构:** **借鉴** **Strategy Pattern (策略模式)**，构建了可扩展的插件架构：
  
  ​* **SmartCatTool** **作为入口，维护一个** **Formatter** **列表。**
  
  * **TimeSeriesFormatter**: 负责计算 P99、绘制 Sparklines、异常点高亮。
  * **LogFormatter**: 负责日志降噪、错误堆栈折叠。
  * **Event/AlertFormatter**: 负责精简json，形成节省token，可读性良好的markdown格式的事件告警描述
* ​**价值:** **实现了** **Open/Closed Principle (开闭原则)**。未来新增数据类型（如拓扑图文本化），无需修改核心逻辑，只需注册新的 Formatter。​

### 3. Impact (影响)

* **精度提升:** **Agent 对“瞬间脉冲 (Spike)”的识别率显著提升，幻觉误报率大幅下降。**
* **成本优化:** **通过“智能折叠”非关键数据，单次诊断的 Token 消耗降低了约 40%。**


## 📅 2025-12-18: Phase 4 - 自动化执行固定流程分析，产出高质量单应用报告

*  **问题:** **利用llmagent自主查询执行固定化的分析流程既耗时又浪费token，大模型注意力稀释**
*  ​**架构:** 流程化，固定化，统计学方法整理指标

详情：[指标分析算法](https://github.com/qingshanyuluo/AIOps-Intelligent-RCA/blob/main/doc/design/timeseries-analysis.md)

* ​**价值:** **指标分析几分钟=>毫秒级**

## 📅 2025-12-18: Phase 5 - 交互协议重构：强制“思考后行动”

### 1. Problem (痛点)

模型在 Function Calling 模式下有严重的“扳机指 (Trigger-Happy)”现象：为了追求响应速度，往往跳过推理直接抛出 JSON。这导致 Agent 陷入“盲目试错”循环，且缺乏可追溯的思维链路，调试困难。

### 2. Key Decisions (关键决策)

#### A. 实施“工具内嵌思考” (In-Tool Reasoning Injection)

* **设计:** 修改所有工具的 Schema，强制加入一个必填参数 ​**rationale ​(调用理由)**​。
* **机制:** 利用 JSON 线性生成的特性，迫使模型在生成具体业务参数（如 metric\_name）之前，必须先写出 rationale。
* **价值:** 在协议层面（Schema Level）物理强制了 CoT（思维链），比纯 Prompt 约束更稳健，实现了“想清楚再动手”。

#### B. 放弃“两步法” (Rejection of Two-Step Generation)

* **权衡:** 虽然“先生成思考文本，再调用工具”的准确率最高，但会带来双倍延迟。
* **结论:** **Schema 注入方案** 在单次 API 调用中同时完成了“思考”与“执行”，在保持低延迟的同时，解决了 90% 的盲目调用问题。

### 3. Impact (影响)

* **自愈能力:** 模型在填写 rationale 过程中经常出现“自我纠正”行为，无效工具调用大幅减少。
* **可观测性:** 现在的 Agent 日志天然包含了“为什么做”的逻辑轨迹，极大方便了后续优化。


## 📅 2025-12-19: Phase 6 - 混合智能架构：从“经验直觉”到“统计科学” (Hybrid Intelligence)

### 1. Context (背景与痛点)

**随着监控规模的扩大，我们遭遇了 AIOps 领域的经典难题——“基础设施共模故障 (Common Mode Failure)”。**

* **场景:** 当底层网络专线发生物理抖动或丢包时，上层数十个无依赖关系的应用会同时抛出超时告警。
* **Agent 的困境:** LLM 缺乏全局物理视角，倾向于将每个应用的报错孤立看待（认为是代码 Bug），瞬间并发数十个无意义的分析任务。这导致 Token 消耗爆炸，且最终结论往往是错误的。

### 2. Key Decisions (关键决策)

#### A. 引入“确定性模式识别”拦截层 (Deterministic Pattern Interceptor)

* **决策:** **AI 不是万能的，规则 + AI 才是。** 在 Agent 介入前，引入基于专家经验的 **System 1 (快思考)** 拦截层。
* **初版算法 (Based on Expert Intuition):**
  我们将资深 SRE 的经验——“大面积报错 + 阶梯状分布 = 网络抖动”——转化为启发式规则：
  
  * **Scope:** 报错应用数 > Threshold (e.g., 20)。
  * **Shape:** 错误数排序后呈\*\*“平原/阶梯状”\*\*分布，而非个别应用的“脉冲状”。
  * **Variance:** Top N 应用的报错数量差距​**不超过 3 倍**​。
* **逻辑:** 只有底层物理设施（路面）出问题时，跑在上面的车（应用）才会无论型号大小，都出现相似幅度的颠簸。

#### B. 算法迭代：归一化与时空相关性修正 (The "Normalized Co-Occurrence" Upgrade)

* **问题 (The QPS Trap):** 初版算法基于“绝对报错数”，忽略了应用间的 QPS 差异。高并发应用的报错数自然更高，容易掩盖低并发应用的异常，导致漏判。
* **架构升级:** 为了提升算法在异构环境下的鲁棒性，我们将“老师傅经验”升级为严格的​**统计学检测 (Statistical Detection)**​：
  
  * **Metric Normalization (归一化):** 将比较维度从 ErrorCount 升级为 ErrorRate。
    
    * 逻辑: 无论 QPS 差多少倍，在物理丢包面前，受难程度（错误率）应保持一致（Homogeneity）。
  * **Temporal Alignment (时序对齐):** 引入 ​**Pearson Correlation (皮尔逊相关系数)**​。
    
    * 逻辑: 计算 Top N 应用错误曲线的相关性。若
      
      ```rendered
      R>0.85
      ```
      
      ，证明它们在同频共振，进一步排除巧合噪音。
  * **Topological Isolation (拓扑离散性):**
    
    * 逻辑: 验证这些报错应用在调用链图谱上是​**离散的点 (Isolated Nodes)**​，而非连通子图，彻底区分“网络震荡”与“服务雪崩”。

#### C. 短路机制 (Short-Circuit Mechanism)

* **设计:** 一旦拦截层命中上述特征：
  
  * **立即终止** 后续所有 LLM 的 Root Cause Analysis 任务。
  * **直接输出** 确定性结论：“检测到共模故障（如网络专线抖动），涉及应用 N 个，请检查基础设施。”

### 3. Impact (影响)

* **降噪比:** 在网络抖动演练中，无效的 LLM 调用减少了 ​**100%**​，避免了“一本正经胡说八道”的幻觉。
* **成本骤降:** 此类大规模故障的诊断成本从无意义的llm推理降低到了**0 (纯规则计算)**。
* **响应速度:** 诊断时间从分钟级（Agent 逐个分析）缩短至 ​**毫秒级**​。
* **架构意义:** 成功实现了 ​**Hybrid Intelligence (混合智能)**​，证明了在 T0 级系统中，优质的规则代码可以成为 AI 的“脊柱”。

## 📅 2025-12-23: Phase 7 - 根因分析的“入口纠偏”：基于 Trace 聚合的加权拓扑挖掘 (Trace-Aggregated Weighted Mining)

### 1. Context (背景与痛点)

在拥有全量报错 RPC 链路数据后，我们发现单纯依赖“最深报错节点”或“单纯耗时检测”作为分析入口存在严重的**归因偏差**。
**核心矛盾：** 报错节点往往只是“受害者” (Victim)，真正的“始作俑者” (Culprit) 经常隐藏在链路中，**表现为“隐性高耗时”但未报错**，导致 Agent 盯着错误的节点查不出原因。

常见的**“入口误判”**场景：
*   **误判为下游错:** 实际上是 **上游** 发起了 N+1 或重试风暴，耗尽了链路时间，导致下游随机超时。
*   **误判为应用错:** 实际上是 **中间链路** (网络丢包/序列化阻塞) 偷走了时间，客户端和服务端日志对不上。
*   **断链误判:** 实际上是 **中间节点** 挂起（Hanging），导致上游超时，Trace 中断，旧逻辑只怪上游。

我们需要一套**统一的算法**，在 LLM 介入前，通过计算“破坏力权重”，**自动识别真正的分析入口**。

### 2. Key Decisions (关键决策)

#### A. 算法核心：内存图构建与Trace折叠 (In-Memory Graph Construction & Folding)

*   **决策:** 不依赖昂贵的外部图数据库，也不发起额外的 Prometheus 查询。直接在内存中处理 Trace 数据。
*   **逻辑 (Folding):** 将一批含有错误的 Trace 按 `(Parent -> Child)` 关系进行聚合折叠。将时序日志转化为**局部加权拓扑图**。
*   **特征提取:** 对于每一条聚合边，提取三个核心特征：
    1.  `SpanCount`: 交互次数（识别 N+1/重试）。
    2.  `DurationGap`: Client端耗时 - Server端耗时（识别网络/GC）。
    3.  `ErrorCount`: 报错次数（识别稳定性）。

#### B. 统一权重公式：寻找“最大阻力点” (The Resistance Formula)

*   **决策:** 废弃“报错优先”或“耗时优先”的分裂逻辑，采用**统一竞价模型**。
*   **公式:** 定义边的“破坏力权重” ($W_{edge}$)：
    $$W = \underbrace{(AvgLat \times SpanCount)}_{\text{交互耗时 (含N+1)}} + \underbrace{\sum(ClientDur - ServerDur)}_{\text{传输损耗}} + \underbrace{(ErrorCount \times K)}_{\text{错误惩罚}}$$
*   **参数 $K$ (灵敏度):**
    *   这是一个可调节的超参数，通常设为 **3000ms** (即 1次报错 $\approx$ 3秒超时)。
    *   *价值:* 这使得算法能同时兼容“找报错”和“找隐性慢”两种需求，实现逻辑的**超集替代**。

#### C. 动态入口纠偏策略 (Dynamic Entry Rectification)

计算出图中所有边和节点的权重后，进行全局排序，根据 **Top 1 权重归属** 强制纠偏分析入口：

1.  **纠偏至“边” (Edge-Dominant) $\rightarrow$ 交互/网络问题:**
    *   *现象:* 某条边 $W$ 极高，胜出原因是 `SpanCount` 巨大或 `Gap` 巨大。
    *   *判定:* 下游服务本身无辜。**分析入口锁定为“调用关系”**。
    *   *Agent动作:* 提示 LLM 关注“循环调用”或“网络质量”，而非下游代码。

2.  **纠偏至“节点” (Node-Dominant) $\rightarrow$ 自身计算问题:**
    *   *现象:* 某节点的 **Self-Time** ($Total - \sum W_{children}$) 极高，且无高权重的出边。
    *   *判定:* **分析入口锁定为“节点自身”**。
    *   *Agent动作:* 提示 LLM 关注该节点的 CPU、锁竞争或死循环逻辑。

3.  **纠偏至“错” (Error-Dominant) $\rightarrow$ 稳定性问题:**
    *   *现象:* 某节点 $W$ 极高，完全由 `ErrorCount * K` 贡献。
    *   *判定:* **分析入口锁定为“报错节点”**。
    *   *Agent动作:* 保持旧逻辑行为，分析该节点的错误日志。

### 3. Impact (影响)

*   **逻辑大一统 (Unification):** 用一套算法完美替代了原有的“最深报错”和“耗时分析”两套逻辑。既能抓“硬伤”（报错），也能抓“内伤”（N+1、网络）。
*   **零 IO 成本:** 算法完全基于已采集的 Trace 数据在内存运行，**不需要** 额外的 Prometheus 查询成本，极其高效。
*   **Agent 智商跃升:** 彻底消除了因“受害者报错”导致的误导，确保 LLM 拿到的 Context 是经过数学验证的**真实瓶颈**。

## 📅 2025-12-24: Phase 8 - 上下文卫生治理：大输出的“带外处理”策略 (Context Hygiene & Out-of-Band Strategy)

### 1. Context (背景与痛点)

随着工具链的丰富，Agent 在调用某些深层诊断工具（如抓取完整堆栈、读取大日志块）时，经常遭遇 ​**Context Pollution (上下文污染)**​。

* **现象:** 单个工具返回超过 10k 字符的原始文本，直接撑爆了 Context Window。
* **后果:** 模型出现明显的 **"Lost in the Middle"** 现象，注意力被噪音稀释，导致推理逻辑断层，甚至忘记了最初的任务目标。
* **反思:** 参考 Claude Code 和 Cursor 等先进 IDE 的处理逻辑，**Raw Data 不应直接进入 Prompt，除非被明确请求。**

### 2. Key Decisions (关键决策)

#### A. 实施严格的输出熔断机制 (Hard Output Cap & Offloading)

* **设计:** 在 Tool Executor 层引入中间件拦截器。
* **阈值:** 设定 **2000 字符** 为“安全水位线”。
* **逻辑:**
  * 若 Output Length <= 2000: 正常返回给 Agent。
  * 若 Output Length > 2000:
    1. **截断 (Truncation):** 仅保留头部 (Head) 关键信息作为预览。
    2. **落盘 (Dump):** 将完整内容写入临时文件存储 (e.g., `/tmp/agent_obs/log_xyz.txt`)。
    3. **引用 (Reference):** 仅向 Agent 返回一句话：
       > *"Output too large (>2000 chars). Head shown above. Full content saved to `{file_path}`. Use dedicated tools to inspect."*

#### B. 增设“间接访问”工具组 (Indirect Access Tools)

* **问题:** 数据被拦截后，Agent 如何获取被隐藏的细节？
* **方案:** 既然不让“全读”，就必须提供“精读”和“概览”工具，迫使 Agent 改变阅读习惯。
* **新增工具:**
  * ​**`search_file(file_path, keyword)`**​: 替代 `cat`，利用 `grep` 逻辑只看含关键词的行（类似 Cursor 的 Terminal 读取机制）。
  * ​**`summarize_file(file_path)`**​: 调用轻量级 LLM (或主模型分身) 对文件内容进行 Map-Reduce 摘要，只返回核心结论（如“发现 3 处 NullPointException”）。

### 3. Impact (影响)

* **智商回升:** 彻底根治了因 Token 过载导致的“变傻”问题，Agent 的长期记忆保持清醒。
* **Token 经济:** 单轮交互的 Token 消耗变得可预测且平稳，不再因为一次意外的 `cat large_log` 而产生高额账单。
* **行为修正:** 观察到 Agent 开始主动学习使用 `summarize` -> `search` 的人类专家行为模式，而非由于惰性直接读取全文。

## 📅 2025-12-24: Phase 9.0 - 认知架构升级：从“静态提示词”到“动态注意力锚定”

### 1. Context (背景与痛点)

**在解决了观测层的数据渲染问题后，我们面临着更隐蔽的“认知崩塌”挑战：**

* **注意力漂移 (Attention Drift):** 当 System Prompt（Layer 0）与最终决策之间隔着数千 Token 的日志、报错堆栈时，LLM 出现了严重的\*\*“近因效应 (Recency Bias)”\*\*。
* **直觉覆盖逻辑:** 尽管 System Prompt 定义了高阶逻辑（如：“全链路超时 \$\\rightarrow\$ 查自身 GC”），但当模型看到大量具体的 "Downstream Timeout" 日志时，其预训练的\*\*“直觉常识”​**（超时即下游挂了）压倒了我们植入的**​“反直觉规则”\*\*。
* **总结工具的污染:** 早期的 `summarize_file` 工具过早地输出了结论性评价，导致主 Agent 在最终推理时，实际上是在复读总结工具的观点，而非执行独立逻辑。

### 2. Key Decisions (关键决策)

#### A. 引入“近因注入”机制 (Recency Injection / The Gatekeeper Pattern)

* **设计:** **放弃“一次 Prompt 定终身”的幻想。** 承认 LLM 在长上下文中的遗忘是必然的，必须在关键决策时刻进行\*\*“注意力重置”\*\*。
* **实现:** **构建“守门员 (Gatekeeper)” 拦截层。**
  * 在 Agent 执行完所有步骤准备输出最终报告（Step 6）前，系统强行插入一条 User Message。
  * **指令内容:** 强制要求模型执行“逻辑核对表”——即再次复述 System Prompt 中的核心归因规则（如“多下游同时变慢判定”），强迫模型在生成结论的前一秒，将注意力拉回核心逻辑。
* **价值:** **利用 LLM 的“近因效应”来对抗“近因效应”。通过在 Context 末端再次锚定规则，确保了核心 SOP 的执行权重高于中间过程的噪音。**

#### B. 实施“观察与推断解耦” (Decoupling Observation from Reasoning)

* **问题:** 之前的 Summary 工具既做\*\*“压缩（Compression）”​**也做**​“判断（Judgment）”\*\*，导致 Fact 和 Opinion 混合，污染了主 Agent 的上下文。
* **架构:** **重构 Summary Tool 的 Prompt 原则 —— "Be a Camera, Not a Judge" (做相机，不做法官)。**
  * **Old:** "请分析日志并总结根因。" \$\\rightarrow\$ 输出："Redis 故障导致超时。" (污染源)
  * **New:** "请列出所有异常现象的客观特征（时间点、报错关键词、涉及组件），​**严禁下结论**​。" \$\\rightarrow\$ 输出："10:00 出现大量 Redis Timeout，同时 MySQL Latency 上升。" (纯净事实)
* **价值:** **保护了主 Agent 的“推理主权”。确保主 Agent 拿到的输入是纯净的“现象”，从而逼迫它必须调用自身的逻辑规则去推导“根因”，而不是偷懒信赖工具的结论。**

### 3. Impact (影响)

* **逻辑鲁棒性:** 在复杂故障（如 STW 导致的全链路超时）场景下，Agent 的归因准确率从 ​**60% 提升至 95%**​，不再被显眼的报错日志带偏。
* **可解释性增强:** 由于强制了“最后一步逻辑核对”，Agent 输出的报告中包含了明确的“排除法”推理过程，使得人类专家更容易信任其结论。


## 📅 2025-12-24: Phase 9.5 - 语义纠偏：建立“拓扑真理”优于“命名直觉”的原则

### 1. Context (背景与痛点)

**在 Phase 9.0 解决了注意力遗忘问题后，我们发现 Agent 仍然存在一种诡异的“高级幻觉”——** ​**语义偏见 (Semantic Bias)**​。

* **现象:** 在排查一个 Feature Service 的延迟时，Agent 发现依赖列表中有一个 `AppPHSpatioTemporalSDEngine`（时空策略引擎），其 QPS 极高且响应慢。
* **错误推理:** 尽管该服务明确位于 `downstream`（下游）字段中，Agent 却因为看到 "Engine" 这个单词，结合其训练数据中的架构常识，​**下意识地将其认定为“上游调用方”**​。
* **后果:** Agent 认为“这是上游流量入口，无需排查其内部耗时”，从而直接忽略了这个导致线程池耗尽的真正​**下游瓶颈**​。
* **本质:** 模型的 ​**先验知识 (Prior Knowledge)**​（Engine = 核心/上游）压倒了 ​**上下文事实 (Contextual Fact)**​（Path = meta.dependencies.downstream）。

### 2. Key Decisions (关键决策)

#### A. 确立“拓扑绝对真理”原则 (Topological Ground Truth)

* **设计:** 引入\*\*“宪法级”\*\*的 Prompt 约束，明确“数据结构”的权威性高于“语义理解”。
* **实现:** 在 System Prompt 的 `Core Principles` 章节加入 **"Path Over Name"** 公理：
  > 🛑 拓扑真理公理：
  > 
  > 判断依赖关系时，JSON 路径是唯一的真理。
  > 
  > * 若在 `downstream` 列表 \$\\rightarrow\$ 它是 ​**被调用方 (Callee)**​。
  > * 若在 `upstream` 列表 \$\\rightarrow\$ 它是 ​**调用方 (Caller)**​。
  > * **严禁**基于服务名（如 Engine, Platform, Core）猜测方向。即使它叫 "GodService"，如果在 `downstream` 里，它就是被当前服务调用的下游。

#### B. 实施“语义对抗”校验 (Semantic Adversarial Verification)

* **问题:** LLM 的思维链容易“顺滑”地滑过逻辑漏洞。
* **策略:** 在 Agent 遇到带有强语义误导性的服务名（如 Engine, Strategy, Gateway）出现在下游时，​**强制触发“反直觉自问”**​。
* **实现:** 修改 `analyze_dependency` 的 Step 逻辑：
  * *Trigger:* 检测到 downstream list 中包含 "Engine" 或 "Strategy"。
  * *Action:* 强制输出 Thought："⚠️ 检测到名字像上游的服务出现在下游列表中。根据拓扑真理，这是一次反向调用或回调。我必须检查它的耗时，因为它会阻塞我的线程。"

#### C. 工具层的“语义锚定” (Tool-Level Semantic Anchoring)

* **设计:** 不再信任模型对原始 JSON 的解析能力，直接在工具输出层面“下毒（Inception）”。
* **实现:** 修改 `cat` 工具对依赖文件的输出格式，从纯 JSON 变为 ​**带注释的语义文本**​。
  * **From:** `{"downstream": ["AppEngine"]}`
  * **To:**
* **价值:** 通过在数据源头注入明确的解释性文本，直接切断了模型产生歧义的可能性。

### 3. Impact (影响)

* **纠偏成功率:** 在针对“策略引擎回调”、“网关双向依赖”等复杂拓扑场景的测试中，Agent 的归因准确率从 ​**40% 飙升至 98%**​。
* **认知升级:** Agent 学会了“微服务无贵贱，位置定乾坤”的道理，不再被高大上的服务命名迷惑。
* **长尾价值:** 这套“反偏见”机制后来被复用到 Database 字段解析中，防止模型错误地将 `is_deleted: 1` 的数据当作有效数据处理（仅凭字段名看起来像是有用的数据）。



## 🔮 Future Roadmap (未来规划)

* **[Planned]** 引入multagent架构、扩展能力指至基础层、代码层...
* **[Planned]** 部分接入私有化微调模型、降低成本
