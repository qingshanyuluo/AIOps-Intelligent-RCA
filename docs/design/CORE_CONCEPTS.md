# AIOps-Intelligent-RCA v2.0: 神经符号化故障分诊引擎

> **Author:** qingshanyuluo
> **Status:** Production-Grade State of the Art
> **Core Philosophy:** "Deterministic Workflow, Probabilistic Inference" (确定性的流，概率性的脑)
> **Architecture Pattern:** Neuro-Symbolic AI (神经符号人工智能)

---

## 1. 序言：从“黑盒代理”到“白盒工厂”

在经历了 ReAct (Reasoning and Acting) 模式在运维领域的幻觉与不可控后，我们回归了第一性原理：**运维排障的本质不是探索（Exploration），而是验证（Verification）。**

当前市面上的 AIOps 方案依然沉迷于让 LLM 像人一样去使用工具（Tool-use），这忽略了一个致命的事实：**大模型严重缺乏运维领域的“过程思维链”训练数据。**

**本项目的核心破局点在于“去 Agent 化”：**
1.  **收敛入口：** 我们不再监听全网所有指标，而是锁死 **RPC Error** 这一“黄金咽喉”。经验证明，RPC 异常覆盖了 90% 的异常与 100% 的生产故障。
2.  **固化流程：** 放弃不确定的 ReAct 循环，采用 **分支型固定 SOP (Branching Fixed SOP)**。
3.  **LLM 降级：** LLM 不再是指挥官（Commander），而是流水线上的 **高智商质检员 (Node)**。它不负责决定“下一步做什么”，只负责在给定的步骤中“看懂非结构化数据”。

我们构建的不是一个会聊天的机器人，而是一座**由算法驱动传送带、由 LLM 负责精细加工的自动化分诊工厂**。

---

## 2. 核心架构：漏斗式分诊流水线 (The Triage Funnel)

我们将 RCA 过程重构为**高度收敛**的四阶段漏斗。所有故障必须经过这套严格的“安检”。

### Stage 0: 全局嫌疑人识别与证据固定 (The Dossier)
**解决痛点：** 仅关注报错应用会遗漏“沉默的性能恶化者”，导致分析起点缺失。我们需要在第一时间构建一个完整的、可供后续所有阶段评估的“案发现场快照”。

*   **输入：**
    1.  聚合的实时 RPC 错误流（含采样链路）。
    2.  全局所有服务的自身耗时（Self-Time）等性能指标。
*   **算法：** **带噪底的自适应 Z-Score (Adaptive Z-Score with Noise Floor)**。
    *   此算法会扫描所有服务的性能指标，精准识别出那些“行为反常”（耗时显著偏离自身历史基线）的服务，即使它们尚未报错。
*   **动作：**
    1.  **生成嫌疑人名单：** 名单由两部分组成：
        *   **报错池 (Error Pool):** 所有上报 RPC 错误的应用。
        *   **异动池 (Anomaly Pool):** 所有被 Z-Score 算法判定为性能异常的应用。
    2.  **并发证据固定 (Concurrent Evidence Collection):** 对上述名单中的 **所有应用**，并发抓取其排障所需的全量数据，包括：所有相关指标、聚类错误日志、告警、变更历史、以及线程 Dump（若可用）。
*   **输出：** 一个标准化的、包含所有潜在关联方信息的 **“故障档案 (Dossier)”**，作为整个后续分析流程的唯一数据源。

### Stage 1: 全局共模熔断 (The Shield)

**解决痛点：** 基础设施故障导致的“告警风暴”与“AI 幻觉”。

* **输入：** 实时 RPC 错误流。
* **算法：** **基于“错误率梯度”的故障定界算法 (Error Rate Gradient-based Fault Localization)**。
* **判定：**
    1. **归一化 (Normalize):** 计算所有报错服务的**错误率** (错误数 / 请求总数)，抹平流量差异。
    2. **排序 (Sort):** 将所有服务的错误率从高到低排序。
    3. **判决 (Judge):** 检查相邻服务错误率的**梯度（倍数差）**。
        * **平台型（Plateau）：** 若错误率分布平缓（相邻差异 < 3倍），则为“场域攻击”。
        * **断崖型（Cliff）：** 若错误率出现断崖（相邻差异 > 3倍），则为“点源爆发”。
* **动作：**
    * 若判定为 **平台型故障**，**直接熔断**。判定为网络专线抖动或云厂商故障，终止单应用分析，输出全局告警。
    * 若判定为 **断崖型故障**，将排在第一位的应用视为根因嫌疑人，放行至下一阶段处理。

### Stage 2: 加权拓扑挖掘与入口纠偏 (The Compass)
**解决痛点：** 报错节点往往是“受害者”，真正的“始作俑者”常因“隐性高耗时”（如 N+1）而被忽略，导致分析方向从一开始就跑偏。

*   **算法核心：** **基于 Trace 聚合的加权拓扑挖掘 (Trace-Aggregated Weighted Mining)**。
    *   **内存图构建 (In-Memory Graph):** 无需外部图数据库，直接在内存中将批量错误 Trace **折叠 (Folding)** 成一个局部加权拓扑图。
    *   **统一权重公式 (The Resistance Formula):** 对图中的每条**边 (Edge)** 计算其“破坏力权重”，寻找链路中的“最大阻力点”。
    $$W_{edge} = \underbrace{(AvgLat \times SpanCount)}_{\text{交互耗时 (含N+1)}} + \underbrace{\sum(ClientDur - ServerDur)}_{\text{网络/GC等传输损耗}} + \underbrace{(ErrorCount \times K)}_{\text{稳定性惩罚}}$$

    <img width="886" height="104" alt="image" src="https://github.com/user-attachments/assets/641af63d-7744-4032-a38e-01b0c80222ae" />


*   **动态入口纠偏 (Dynamic Entry Rectification):** 根据权重计算结果，**强制校准**分析入口。
    *   **纠偏至“边” (Edge-Dominant):** 若某条边的 `SpanCount` 或 `传输损耗` 极高，则判定为**交互/网络问题**（如 N+1 查询、网络丢包），LLM 将被引导关注调用关系而非下游代码。
    *   **纠偏至“节点” (Node-Dominant):** 若某节点的 `Self-Time`（自身耗时）远超其出边权重，则判定为**自身计算问题**（如 CPU 密集、死循环），LLM 将被引导关注该节点内部逻辑。
    *   **保持“报错” (Error-Dominant):** 若权重主要由 `ErrorCount` 贡献，则保持原有逻辑，分析**报错节点**。
*   **价值：** 此阶段用纯数学方法替代了“最深报错节点”的启发式规则，确保后续的 SOP 和 LLM 推理，都聚焦在经过验证的、真正的瓶颈之上。

### Stage 3: 分支型 SOP 路由 (The Router)
**解决痛点：** 通用 Prompt 无法处理特异性故障。

这是本架构彻底摒弃 ReAct 的关键。我们不让 LLM 决定怎么查，而是根据 RPC 错误类型，**硬编码**进入三条完全不同的 SOP 轨道：

1.  **轨道 A (Type: Slow / Timeout):**
    *   **特征:** `SocketTimeout`, `ReadTimeout`, P99 飙升。
    *   **SOP 侧重:** 资源饱和度分析 (CPU Steal, GC Stop-the-world, DB Lock)。
2.  **轨道 B (Type: Dead / Unreachable):**
    *   **特征:** `ConnectionRefused`, `NoRouteToHost`, `UnknownHost`.
    *   **SOP 侧重:** 编排状态与网络分析 (K8s Pod Status, Service Mesh Config, Liveness Probe)。
3.  **轨道 C (Type: Wrong / Logic):**
    *   **特征:** `5xx`, `NullPointer`, `BusinessException`.
    *   **SOP 侧重:** 代码堆栈语义分析 (Stacktrace Parsing, Git Blame, Config Change)。

### Stage 4: 神经符号推理与定责 (The Judge)
**解决痛点：** 最终定性与“责任甩锅”。

LLM 仅在此阶段介入。它接收的是经过 Stage 3 整理好的、高度结构化的证据包。它的任务只有两个：

1.  **反事实陪审团 (Counterfactual Jury):** 执行下文提到的逻辑验证。
2.  **责任定界 (Boundary Definition):**
    *   如果排除了所有基础设施与中间件问题，LLM **必须**输出：“经数学验证，网络/资源/数据库均正常。根因为应用层逻辑异常（如代码 Bug），请转派研发团队。”

---

## 3. 核心创新：反事实验证 (Counterfactual Verification)

在 SOP 流程的最后一步，我们引入 LLM 作为“逻辑过滤器”，专门用于**否决**不合理的结论。

### 3.1 核心逻辑：质疑显性症状

我们强制 LLM 执行以下“反直觉”的推演：

#### A. 概率独立性原则 (针对超时场景)
*   **现象:** MySQL, Redis, ES 同时变慢。
*   **直觉:** 三个组件都坏了。
*   **反事实:** “三个独立组件同时物理故障的概率趋近于零。”
*   **结论:** 必然是**应用自身**（Caller）出现了 CPU 争抢或 GC 停顿，导致客户端计时器变慢。
*   **口诀:** **单独涨 = 组件问题；一起涨 = 自身问题。**

#### B. 资源受害者假设 (针对资源报警)
*   **现象:** CPU 100%。
*   **反事实:** “如果没有代码 Bug，CPU 为什么会满？”
*   **验证:** 计算 `Impact = ΔLatency × QPS`。
*   **推论:** 如果下游变慢导致线程池堆积，那么 **CPU 飙升是结果（受害者），而非原因**。

#### C. T-1 时序铁律
*   **逻辑:** 原因必须早于结果。
*   **验证:** 若 $T_{cause} > T_{error}$，直接标记为“幻觉”并丢弃该结论。

---

## 4. 算法引擎详解 (The Math behind the Magic)

我们坚持：**能用数学解决的，绝不问 LLM。**

### 4.1 Z-Score 异常检测 (用于 Stage 2)
用于在茫茫微服务中，精准定位那个“沉默但致命”的下游节点。
$$Z = \frac{\text{CurrentMax} - \mu}{\sigma + \text{NoiseFloor}}$$
*   **NoiseFloor:** 保护机制，防止低流量服务的数学误报。
*   **作用:** 即使没有报错日志，只要耗时违背了正态分布，就会被“抓”出来送给 LLM 审判。

### 4.2 拓扑加权挖掘 (用于 Stage 2)
不再盲目扫描所有依赖，而是计算“破坏力权重”：
$$W = (\text{Latency} \times \text{QPS}) + (\text{ErrorCount} \times 10)$$
*   **策略:** 优先采集权重 $W$ 最高的 Top 3 依赖节点的数据。

---

## 5. 总结：工业级的克制

**AIOps-Intelligent-RCA v2.0** 代表了一种架构上的成熟与克制。

*   我们放弃了让 Agent 像人类一样去 Terminal 里敲 `ls` 和 `grep` 的幻想。
*   我们选择了 **RPC Error** 这个单一且覆盖率极高的抓手。
*   我们建立了 **“算法清洗 -> SOP 路由 -> LLM 判决”** 的严密流水线。

这是一个**“SRE 经验代码化”**与**“LLM 语义理解能力”**的完美结合。它也许不再像科幻电影里的 AI 那样“自主”，但它能稳定地、准确地、7x24 小时地守住生产环境的生命线。
