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

## 🔮 Future Roadmap (未来规划)

* **[Planned]** 引入multagent架构、扩展能力指至基础层、代码层...
* **[Planned]** 部分接入私有化微调模型、降低成本
