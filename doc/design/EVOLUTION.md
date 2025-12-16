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
* ​**价值:** **实现了** **Open/Closed Principle (开闭原则)**。未来新增数据类型（如拓扑图文本化），无需修改核心逻辑，只需注册新的 Formatter。
  
  ​

### 3. Impact (影响)

* **精度提升:** **Agent 对“瞬间脉冲 (Spike)”的识别率显著提升，幻觉误报率大幅下降。**
* **成本优化:** **通过“智能折叠”非关键数据，单次诊断的 Token 消耗降低了约 40%。**
  

## 🔮 Future Roadmap (未来规划)

* **[Planned]** 引入multagent架构、扩展能力指至基础层、代码层...
* **[Planned]** 部分接入私有化微调模型、降低成本
