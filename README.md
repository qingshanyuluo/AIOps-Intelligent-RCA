# AIOps-Intelligent-RCA
An Agent-based Root Cause Analysis Framework with Counterfactual Verification.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-Research_Prototype-orange.svg)]()
[![DOI](https://zenodo.org/badge/1114458069.svg)](https://doi.org/10.5281/zenodo.17898791)


## 📖 Introduction (项目简介)

**AIOps-Intelligent-RCA** 是一个面向大规模微服务架构的智能根因分析 (RCA) 框架。针对传统运维中告警风暴和专家经验难以沉淀的痛点，本项目提出了一种 **Agent-Driven (智能体驱动)** 的诊断模式。

通过 **Retrieval-as-a-Tool** 机制，Agent 能够自主查询 Metrics、Logs、Traces 和 Events，并引入 **Counterfactual Verification (反事实验证)** 机制，有效解决了 LLM 在运维领域的幻觉问题。

## 🚀 Key Features (核心特性)

* **🕵️‍♂️ Agentic Diagnosis:** 基于 LLM 的推理核心，动态编排排查步骤，而非死板的规则树。
* **📉 Multi-modal Fusion:** 融合 RPC 错误率、Z-score 异常检测、拓扑链路分析、Change Events 等多维数据。
* **🛡️ Counterfactual Verification:** **(创新点)** 系统在得出结论前，会自动生成反事实假设（"如果是网络问题，TCP重传率应升高"）并进行自我验证，大幅提升准确率。
* **🔍 Automatic Topology Drill-down:** 自动识别 Trace 中的最深报错节点与慢节点。

## 🏗️ Architecture (系统架构)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'primaryColor': '#4caf50', 'edgeLabelBackground':'#ffffff', 'tertiaryColor': '#f1f8e9'}}}%%
graph LR
    %% ================== 样式定义 ==================
    classDef storage fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef process fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef agent fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px,stroke-dasharray: 5 5;
    classDef core fill:#ce93d8,stroke:#4a148c,stroke-width:3px;
    
    %% ================== 1. 触发与采集层 ==================
    subgraph Trigger ["1. 触发与采集 (Trigger & Collect)"]
        direction TB
        Alert["🔥 告警触发 (Alert)"]
        RawCollector["数据采集器 (Collector)"]
        RawData[("原始海量数据\nRaw Logs/Metrics")]
    end

    %% ================== 2. 数据处理层 (核心变更) ==================
    subgraph DataPipeline ["2. 数据清洗与分层 (ETL)"]
        direction TB
        Cleaner["清洗与精简 (Cleaning & Pruning)"]
        V2Data[("✨ 版本2数据 (V2 Data)\n(结构化/高信噪比)")]
    end

    %% ================== 3. 智能分析层 ==================
    subgraph Brain ["3. 智能根因分析 (Agent Core)"]
        direction TB
        
        %% Agent 内部逻辑
        subgraph AgentLogic ["Agent 决策流"]
            FixedFlow["固定流程 (SOP/Checklist)"]
            Decision{"SOP能解决?"}
            ReAct["ReAct 推理模式\n(Reasoning & Acting)"]
        end
        
        Tool_Retrieval["工具: Retrieval-as-a-Tool"]
    end

    %% ================== 4. 验证与输出 ==================
    subgraph Outcome ["4. 验证与输出 (Output)"]
        Verifier["🛡️ 反事实验证 (Counterfactual Verification)"]
        FinalReport["📝 最终诊断报告"]
    end

    %% ================== 连线逻辑 ==================
    
    %% 数据流
    Alert --> RawCollector
    RawCollector --> RawData
    RawData --> Cleaner
    Cleaner -->|"降噪/预聚合"| V2Data

    %% 触发 Agent
    Alert -->|"启动诊断"| FixedFlow

    %% Agent 思考流
    FixedFlow --> Decision
    Decision -- Yes --> Verifier
    Decision -- No --> ReAct
    
    %% 工具调用 (只查 V2 数据)
    ReAct <-->|"思考: 数据不足 -> 调用"| Tool_Retrieval
    FixedFlow <-->|"查指标"| Tool_Retrieval
    Tool_Retrieval <-->|"提取高价值信息"| V2Data

    %% 验证流 (核心)
    ReAct -->|"得出初步结论"| Verifier
    Verifier --"假设不成立 -> 重试"--> ReAct
    Verifier --"验证通过"--> FinalReport

    %% 样式应用
    class RawData,V2Data storage;
    class RawCollector,Cleaner,Tool_Retrieval,Verifier process;
    class AgentLogic agent;
    class ReAct,FixedFlow core;
```

### Workflow
1.  **Alert Trigger:** RPC 错误率突增触发诊断。
2.  **Data Aggregation:** 自动聚合 1h 内的 Logs 和 Metrics。
3.  **Root Cause Reasoning:** Agent 通过剪枝及统计学算法等方式识别可疑应用后通过Retrieval-as-a-Tool获取数据，推测根因
4.  **Verification:** 执行反事实推理，验证假设。
5.  **Report:** 生成包含根本原因和建议的报告。

## 💻 Core Logic (核心逻辑摘要)

### 1. Retrieval-as-a-Tool
Agent 并不直接“看”所有数据，而是通过 Tool 调用获取数据，模拟专家排查过程：

### 2. Counterfactual Verification (反事实验证)
这是本框架防止幻觉的核心机制：

Hypothesis: "Redis 响应慢导致上游 RPC 超时" Counterfactual Check: 查询 Redis 实例过去 10 分钟的 P99 延迟。 Result: 如果 P99 < 10ms，则推翻假设，Agent 重新规划排查路径。

## 📂 Case Study (案例分析)

> [TODO: Case Study]

📝 Citation & Contact
如果您对该架构感兴趣，欢迎在 Issues 中讨论。

Disclaimer: This repository contains the architectural design and reference implementation patterns. Proprietary business logic has been obfuscated.
