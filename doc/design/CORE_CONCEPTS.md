# AIOps-Intelligent-RCA: 核心概念与架构设计

> **Author:** qingshanyuluo **Status:** Production-Ready **Core Philosophy:** "Heavy Collection, Smart Reasoning"

## 1. 序言：工程化的胜利

本项目不迷信 LLM 的黑盒能力。在真实的生产环境中，Agent 的智商上限取决于 **上下文（Context）** 的质量。
我们的核心设计理念是：**在 Agent 介入之前，通过确定性算法完成“战场清洗”和“全量取证”。**

## 2. 诊断流水线 (The Pipeline)

我们将 RCA 过程重构为三个严格的阶段：**拦截 -> 采集 -> 分析**。

### Stage 1: 全局共模拦截 (The Shield)
**解决痛点：** 基础设施故障导致的“告警风暴”与“AI 幻觉”。

在启动任何单应用分析之前，系统首先运行 **归一化与时空相关性检测算法 (Phase 6)**：
*   **逻辑：** 实时计算 Top N 报错应用的错误率曲线。
*   **判定：** 如果应用间错误曲线的 **Pearson 相关系数 $R > 0.85$** 且拓扑上互不相关（离散）。
*   **动作：** **直接熔断**。判定为网络专线抖动或基础设施故障，终止后续所有 Agent 任务，输出全局告警。

### Stage 2: 智能数据采集与补全 (The Dragnet)
**解决痛点：** Trace 断裂、下游隐性劣化（不报错但变慢）、数据不完整。

这是本架构最关键的**“补全”**步骤。系统不仅仅拉取报错应用的 Trace，而是启动 **Z-Score 自适应异常检测算法**，对全局服务进行“拉网式排查”。

#### 核心算法：基于 Z-Score 的自适应检测
系统扫描全局应用，寻找所有满足以下条件的服务，将其指标和日志一并打包进 Context：

1.  **自适应基线：** 基于过去 1 小时数据的均值 ($\mu$) 和标准差 ($\sigma$) 建立模型。
2.  **异常判定：**
    $$Z = \frac{\text{CurrentMax} - \mu}{\sigma + \text{NoiseFloor}}$$
    *   **Noise Floor (噪音保护):** 防止低波动服务的数学误报。
    *   **QPS 门槛:** 过滤小流量服务的统计噪声。
3.  **价值：**
    *   **解决断链：** 即使 Trace 在 A 处断裂，下游 C 因为变慢被 Z-Score 算法捕获。A 和 C 同时出现在 Context 中，Agent 即可通过拓扑知识将其关联。
    *   **发现隐患：** 捕捉到那些“沉默”但耗时突增的罪魁祸首。

### Stage 3: 内存图构建与入口决策 (The Anchor)
**解决痛点：** 选错分析入口（受害者 vs 始作俑者）。

基于 Stage 2 采集到的全量数据（Trace + Z-Score 异常列表），在内存中构建 **加权拓扑图 (Phase 7)**：
*   **权重计算：** $$W = (交互耗时 \times N+1) + (传输损耗) + (错误惩罚)$$
*   **决策：** 综合 Trace 中的显性错误和 Z-Score 发现的隐性高耗时节点，计算出破坏力最大的节点作为 **Agent 的唯一分析入口**。

### Stage 4: Agent 深度分析 (The Surgeon)
**解决痛点：** 最终定性与报告生成。

此时，Agent 进场。它面对的不再是破碎的片段，而是一个**经过清洗、补全、数学验证过的完整案发现场**。
*   **RaaT (Retrieval-as-a-Tool):** Agent 通过 `analyze_metrics_report` 等高阶工具读取已缓存的指标快照。
*   **反事实验证:** 进行最终的逻辑闭环（例如：“确认下游 C 的 Z-Score 飙升是导致 A 线程池满的唯一原因”）。
*   **语义纠偏:** 遵循“拓扑真理”原则，无视服务命名误导，输出最终根因报告。

---


## 3. 核心创新：RaaT (Retrieval-as-a-Tool) 交互框架

**这是 Agent 进行 Stage 4 深度分析的核心引擎。**
我们发现直接 Dump JSON 或让 LLM 写 SQL 既脆弱又浪费 Token。因此，我们构建了一套 **模拟 Linux 文件系统** 的虚拟交互环境。

### 3.1 交互协议：文件系统隐喻 (File System Metaphor)
利用 LLM 在预训练阶段对 Shell/Linux 环境的深刻直觉，将复杂的 APM 数据抽象为“目录”和“文件”。我们精简出 4 个原子工具：

1.  **`ls` (探索):** 查看当前上下文结构（如 `ls /metrics/redis`）。
2.  **`guide` (导航):** **Lazy Loading 的核心。**
    *   Agent 进入目录先读 `guide`，它仅返回“这里有什么数据”及“建议排查思路”，**不返回**具体数据。
    *   *价值:* 防止 Agent 被海量 Context 淹没，迫使它像人类专家一样“先看目录，再查细节”。
3.  **`cat` (读取):** 读取具体数据切片（详见下文“智能渲染”）。
4.  **`grep` (搜索):** 跨维度的关键词搜索（如全局搜 `TraceID` 或 `Exception`）。

### 3.2 观测层升级：富观测渲染 (Rich-Observation Rendering)
Agent 最大的弱点是 **数值盲区** 和 **Token 效率低**。我们确立了 **"Tool Should Be Smart"** 原则，`cat` 不仅仅是读取，更是**渲染**。

#### A. 解决数值盲区：ASCII Sparklines
LLM 对浮点数组 `[0.1, 0.2, ... 99.0]` 无感。
*   **渲染方案：** `SmartCatTool` 调用 `TimeSeriesFormatter`，将数据渲染为字符趋势图：
    > `Latency: 12ms ▂▃▄▆██ (P99 Spike Detected)`
*   **效果：** 利用 LLM 强大的**视觉/形状语义理解能力**，实现“一眼看穿”故障趋势（脉冲、阶梯、渐变）。

#### B. 解决 Token 爆炸：Formatter Registry
原始 JSON 信息密度极低。我们引入 **策略模式 (Strategy Pattern)** 对不同文件类型进行语义折叠：
*   **`LogFormatter`:** 降噪无用 Tag，折叠重复堆栈。
*   **`EventFormatter`:** 将复杂的 K8s Event JSON 转换为高可读性的 Markdown 摘要。
*   **扩展性:** 符合开闭原则，新增数据类型只需注册新的 Formatter。

### 3.3 卫生治理：输出拦截器 (The Interceptor)
为了防止 `cat` 意外读取超大日志撑爆 Context Window：
*   **熔断机制：** 若工具输出 > 2000 字符，中间件自动拦截。
*   **行为修正：** 仅返回 Head 预览，并提示 Agent *"Output too large. Use `grep` or `summarize` tool instead."*
*   **价值：** 强制 Agent 保持清醒，避免陷入 "Lost in the Middle" 陷阱。

---

## 4. 核心算法详解 (Key Algorithms)

### 4.1 工业级微服务耗时异常检测 (At Collection Layer)
为了在数据采集阶段解决“断链”和“漏判”，我们引入了非对称采样的 Z-Score 算法：
*   **Target (当前):** 取过去 5 分钟内，每分钟平均耗时的**最大值**（捕捉瞬间尖刺）。
*   **Baseline (历史):** 取过去 1 小时的均值与标准差。
*   **优势:**
    *   **对稳定服务敏感:** 标准差小，微小劣化立刻被捕获。
    *   **对抖动服务宽容:** 自动调高阈值，忽略正常毛刺。
    *   **计算性能:** 这种非对称采样策略大幅降低了 Prometheus 的查询压力，使得“全网扫描”成为可能。

### 4.2 内存加权拓扑挖掘 (At Entry Selection Layer)
*   **统一竞价:** 不再区分“查报错”还是“查耗时”，将两者统一转化为“破坏力权重”。
*   **零 IO:** 算法完全基于已采集的 Trace 和异常列表在内存运行，毫秒级产出最佳分析入口。

---

## 4. 总结

**AIOps-Intelligent-RCA** 的本质是一个 **由算法驱动数据采集，由 LLM 驱动逻辑推理** 的混合系统。

*   **Z-Score 算法** 负责“看见”隐形的故障。
*   **Pearson 相关性** 负责“过滤”物理的噪音。
*   **LLM Agent** 负责“讲述”故障的故事。

我们通过在 Data Pipeline 层的重投入，换取了 Agent 推理层的极简与高准确率。
