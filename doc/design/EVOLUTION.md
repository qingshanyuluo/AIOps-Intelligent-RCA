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

## 📅 2025-12-23: Phase 7 - 拓扑归因的升维：从“线性追踪”到“加权因果游走” (H-RCA)

### 1. Context (背景与痛点)

在解决了 Trace 断链问题后，我们发现现有的 Trace 分析逻辑在处理 “非显性延迟” 故障时存在巨大的盲区。

传统的 APM 分析往往只关注 Span Duration (单次耗时)，导致以下“隐形杀手”长期逃逸：

* **重试风暴 / N+1 问题:** 单次调用极快（1ms），但由上游发起数千次调用。Trace 视图中下游全是绿色的，导致根因误判为上游代码慢。
* **网络/序列化损耗:** A 调用 B 超时，B 内部处理极快。传统逻辑认为 B 没问题，却忽略了 A->B 之间的“路”堵了。
* **快速失败 (Fail-Fast):** 异步调用报错瞬间返回，耗时极短，权重被自动降低，导致严重错误被忽略。

### 2. Key Decisions (关键决策)

#### A. 重新定义“严重程度”：引入多维影响权重 (Multi-Dimensional Impact Score)

* **决策:** 放弃单一的“时间”维度，构建一个新的权重公式 **\$W\$** 来衡量边（Edge）的“阻力”。
* **演进:**
  
  * *V1 (乘法模型):* 发现对于极小耗时 (0.1ms) 的高频调用，乘积依然不够大，容易漏判。
  * V2 (加法混合模型 - 最终态): 采用 “基准消耗 + 惩罚项” 结构。
    $$
    W = \sum Duration + (ErrorCount \times P_{error}) + (Gap_{network})
    $$
* **价值:** 算法不再只看“谁慢”，而是看“谁给系统造成的熵增最大”。这让 **“高频低耗时”** 和 **“高错低耗时”** 的节点能瞬间浮出水面。

#### B. 算法核心：基于时空权重的涟漪下探 (Heuristic Ripple Descent)

* **设计:** 拒绝全图遍历（计算成本过高）。采用 **On-Demand (按需)** 的局部启发式搜索。
* **流程:**
  1. **Snapshot:** 拉取告警节点下游 N 层的局部拓扑。
  2. **Calculate:** 计算每条边的 \$W\$。
  3. **Walk:** 从入口开始，总是流向 \$W\$ 最大的分支。
  4. **Stop:** 当发现节点的 `Self-Time` (自身计算耗时) 占比超过阈值，或 `Network Gap` 占比过高时，锁定凶手。
* **价值:** 将原本依赖 LLM 也就是 O(N^2) 的推理复杂度，降低为 O(K) 的纯内存计算，实现了毫秒级的根因定位。

#### C. 工程防守：规避分布式系统的“物理陷阱”

* **时钟偏差 (Clock Skew):**
  * *问题:* 直接用 \$Start\_{client} - Start\_{server}\$ 计算网络耗时，在机器时钟不同步时会出现负数或极大值。
  * *修正:* 改用 **Duration 差值法** (\$Dur\_{client} - Dur\_{server}\$)。虽然无法精确测量单向延迟，但这足以反映 TCP 重传或序列化阻塞造成的“膨胀”。
* **并发陷阱 (The Concurrency Trap):**
  * *问题:* 高并发调用下，累计耗时可能大于墙上时间（Wall Clock）。
  * *修正:* 引入 `Concurrency Discount` 逻辑。在判断根因时，结合节点的 CPU/Thread 饱和度指标，区分是“真慢”还是“并发带来的等待”。

### 3. Impact (影响)

* **覆盖率突破:** 成功捕获了此前 100% 漏判的 **Redis 轮询 (N+1)** 和 **Kubernetes Service 丢包** 故障。
* **LLM 减负:** 算法充当了“前置侦探”。它不再把几千个 Span 丢给 LLM，而是直接输出结构化结论（如 ​*“Found dominant edge A->B due to massive call count”*​）。LLM 的 Token 消耗进一步降低，只负责生成最终的人话解释。


## 🔮 Future Roadmap (未来规划)

* **[Planned]** 引入multagent架构、扩展能力指至基础层、代码层...
* **[Planned]** 部分接入私有化微调模型、降低成本
