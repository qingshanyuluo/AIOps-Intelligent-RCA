# 时间序列分析文档

本文档说明该项目对时间序列数据的处理方式、使用的算法以及提取的特征。

## 1. 概述

本项目的异常检测系统基于 **滑动窗口对比** 的核心思想：将最近一段时间的数据（当前窗口）与历史数据（基线窗口）进行对比，识别异常模式。

### 时间窗口划分

- **基线窗口**：前 55 分钟数据（`first_55min`）
- **当前窗口**：最后 5 分钟数据（`last_5min`）
- **数据粒度**：每分钟 1 个数据点，共 60 个点

```
|<-------- 基线窗口 (55分钟) -------->|<-- 当前窗口 (5分钟) -->|
[0]  [1]  [2]  ...  [54]              [55] [56] [57] [58] [59]
```

---

## 2. 时间序列数据结构

### TimeSeriesData

```python
@dataclass
class TimeSeriesData:
    name: str           # 指标/序列名称
    values: List[float] # 数值列表（60个点，每点1分钟）
    unit: str = ""      # 单位
```

### 计算属性

| 属性 | 描述 | 计算方式 |
|------|------|----------|
| `last_5min` | 最后5分钟数据 | `values[-5:]` |
| `first_55min` | 前55分钟数据 | `values[:-5]` |
| `last_5min_avg` | 当前窗口均值 | `mean(last_5min)` |
| `first_55min_avg` | 基线窗口均值 | `mean(first_55min)` |

---

## 3. 检测算法详解

### 3.1 错误数飙高检测 (ErrorSpikeDetector)

**算法名称**：历史峰值对比法

#### 算法选择原因

错误数通常具有以下特征：
- **稀疏性**：大量为 0，偶尔有值
- **非正态分布**：不符合高斯分布假设
- **均值/标准差失效**：容易被大量 0 值拉低

因此采用 **历史最大值** 作为基线，而非均值。

#### 检测逻辑

```
1. 计算基线特征：
   - baseline_max = max(first_55min)  # 历史峰值
   - baseline_avg = mean(first_55min) # 历史均值

2. 绝对值检测：
   - IF sum(last_5min) >= 100 → CRITICAL
   
3. 逐点检测（对 last_5min 每个点）：
   - 过滤低噪音：val < 10 → 跳过
   - 基线为0突增：baseline_max == 0 且 val > 0 → WARNING
   - 超过历史峰值：val > baseline_max × 1.5 → CRITICAL
```

#### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `spike_ratio` | 1.5 | 超过历史最大值的倍数阈值 |
| `min_error_count` | 10 | 最小错误数阈值（过滤噪音） |
| `absolute_critical` | 100 | 绝对值 Critical 阈值 |

#### 提取特征

- `current_value`: 最后5分钟均值
- `baseline_value`: 前55分钟均值
- `baseline_max`: 历史峰值
- `change_ratio`: 变化比例
- `anomalous_points`: 异常点列表

---

### 3.2 延迟增高检测 (LatencyIncreaseDetector)

**算法名称**：MAD 稳健统计法

#### 算法选择原因

耗时指标特征：
- **相对稳定**：正常情况下波动较小
- **偶发毛刺**：GC、网络抖动会产生异常点
- **3-Sigma 失效**：毛刺会拉高标准差，导致阈值过宽

**MAD (Median Absolute Deviation)** 是「稳健的标准差」：
- 对异常值不敏感
- 能敏锐捕捉整体水位的抬升

#### MAD 计算公式

```
MAD = median(|x_i - median(x)|)

其中：
- median(x) 是数据的中位数
- |x_i - median(x)| 是每个点到中位数的绝对偏差
```

#### 稳健标准差估计

```
σ_robust = 1.4826 × MAD
```

> 1.4826 是将 MAD 转换为等效标准差的系数（假设正态分布）

#### 检测逻辑

```
1. 计算稳健统计量：
   - median = median(first_55min)
   - mad = MAD(first_55min)
   - σ_robust = 1.4826 × mad

2. 计算动态阈值：
   - threshold = median + 3 × σ_robust

3. 逐点检测（对 last_5min 每个点）：
   - absolute_increase = val - median
   - IF val > threshold AND absolute_increase >= 1ms:
     - IF absolute_increase >= 10ms → CRITICAL
     - ELSE → WARNING
```

#### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `sigma_threshold` | 3.0 | 超过 N 倍 MAD 为异常 |
| `min_absolute_increase` | 1.0ms | 最小绝对增量（避免无意义报警） |
| `critical_increase_ms` | 10.0ms | Critical 绝对增量阈值 |

#### 提取特征

- `current_value`: 最后5分钟均值
- `baseline_value`: 中位数 (median)
- `median`: 基线中位数
- `mad`: 中位数绝对偏差
- `threshold`: 动态阈值
- `change_ratio`: 变化比例
- `anomalous_points`: 异常点列表

---

### 3.3 流量变化检测 (TrafficChangeDetector)

**算法名称**：趋势分析 + 突变检测

#### 检测维度

1. **趋势分析**：整体流量是否在上涨/下降
2. **突变检测**：是否存在突然的尖峰或骤降
3. **变化幅度判断**：变化是否超出合理范围

#### 检测逻辑

```
1. 计算变化比例：
   change_ratio = (last_5min_avg - first_55min_avg) / first_55min_avg

2. 趋势检测（线性回归斜率）：
   slope = Σ(i - x̄)(v_i - ȳ) / Σ(i - x̄)²
   trend = slope / ȳ × n  # 归一化

3. 突变点检测（Z-Score 法）：
   mean = mean(values)
   std = stdev(values)
   FOR each point:
     zscore = (val - mean) / std
     IF zscore > 2.5 → 标记为尖峰
     IF zscore < -2.5 → 标记为骤降

4. 异常判定：
   - 流量下降 ≥ 50% → WARNING
   - 流量上涨 ≥ 30% → WARNING
   - 流量上涨 10%~30% → INFO（正常波动）
```

#### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `trend_increase_info` | 0.1 (10%) | 趋势增长 INFO 阈值 |
| `trend_increase_warning` | 0.3 (30%) | 趋势增长 WARNING 阈值 |
| `spike_ratio` | 2.0 | 单点突增倍数阈值 |
| `drop_ratio` | 0.5 (50%) | 流量下降阈值 |

#### 提取特征

- `current_value`: 最后5分钟均值
- `baseline_value`: 前55分钟均值
- `change_ratio`: 变化比例
- `trend_slope`: 趋势斜率（归一化）
- `spike_indices`: 尖峰点索引列表
- `drop_indices`: 骤降点索引列表

---

### 3.4 GC 耗时检测 (GCDetector)

**算法名称**：Z-Score 波动检测

#### 算法选择原因

GC 耗时特征：
- 有明显的「正常区间」
- 异常时会出现显著偏离
- 需要同时考虑相对波动和绝对值

#### Z-Score 计算

```
Z-Score = (value - mean) / std
```

Z-Score 表示某个值偏离均值多少个标准差。

#### 检测逻辑

```
1. 计算统计量：
   - mean = mean(first_55min)
   - std = stdev(first_55min)

2. 逐点检测（对 last_5min 每个点）：
   - 绝对值检测：
     IF val >= 1000ms → CRITICAL（可能是 Full GC）
   
   - 相对波动检测：
     IF val >= 200ms:  # 过滤低耗时噪音
       zscore = (val - mean) / std
       IF zscore > 3.0 → WARNING
```

#### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `zscore_threshold` | 3.0 | Z-Score 阈值 |
| `min_absolute_ms` | 200ms | 最小绝对值（过滤噪音） |
| `critical_absolute_ms` | 1000ms | Critical 绝对值阈值 |

#### 提取特征

- `current_value`: 最后5分钟均值
- `baseline_value`: 基线均值 (mean)
- `mean`: 前55分钟均值
- `std`: 前55分钟标准差
- `change_ratio`: 变化比例
- `anomalous_points`: 异常点列表

---

### 3.5 CPU 使用检测 (CPUDetector)

**算法名称**：比例变化检测

#### 检测逻辑

```
1. 计算变化比例：
   change_ratio = (last_5min_avg - first_55min_avg) / first_55min_avg

2. 异常判定：
   - change_ratio >= 50% → WARNING
   - change_ratio >= 30% → INFO
   - 其他 → NORMAL
```

#### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `increase_ratio_warning` | 0.3 (30%) | WARNING 增长阈值 |
| `increase_ratio_critical` | 0.5 (50%) | CRITICAL 增长阈值 |

#### 提取特征

- `current_value`: 最后5分钟均值
- `baseline_value`: 前55分钟均值
- `change_ratio`: 变化比例
- `max_value`: 最后5分钟最大值

---

### 3.6 线程池检测

**算法名称**：阈值检测

#### 检测逻辑

**队列长度检测**：
```
- max(last_5min) > 10 → WARNING（队列积压）
- avg(last_5min) > 1 → INFO（少量排队）
- 其他 → NORMAL
```

**利用率检测**：
```
- max(last_5min) > 80% → WARNING（利用率过高）
- max(last_5min) > 50% → INFO（利用率偏高）
- 其他 → NORMAL
```

#### 提取特征

- `current_value`: 最后5分钟均值
- `baseline_value`: 前55分钟均值
- `last_5min_max`: 最后5分钟最大值

---

## 4. 特征提取汇总

### 4.1 通用特征

所有检测器都会提取以下基础特征：

| 特征 | 类型 | 描述 |
|------|------|------|
| `current_value` | float | 当前窗口均值 |
| `baseline_value` | float | 基线窗口均值/中位数 |
| `change_ratio` | float | 变化比例 `(current - baseline) / baseline` |
| `is_anomaly` | bool | 是否异常 |
| `severity` | Enum | 严重程度 (NORMAL/INFO/WARNING/CRITICAL) |

### 4.2 检测器特定特征

| 检测器 | 特定特征 |
|--------|----------|
| ErrorSpikeDetector | `baseline_max`, `anomalous_points` |
| LatencyIncreaseDetector | `median`, `mad`, `threshold`, `anomalous_points` |
| TrafficChangeDetector | `trend_slope`, `spike_indices`, `drop_indices` |
| GCDetector | `mean`, `std`, `anomalous_points` |
| CPUDetector | `max_value` |

---

## 5. 算法选择总结

| 指标类型 | 算法 | 原因 |
|----------|------|------|
| 错误数 | 历史峰值对比 | 稀疏数据，均值失效 |
| 延迟 | MAD 稳健统计 | 抗毛刺，敏感捕捉水位抬升 |
| 流量 | 趋势分析 + Z-Score 突变检测 | 需同时检测趋势和突变 |
| GC 耗时 | Z-Score 波动 | 有明显正常区间，需检测偏离 |
| CPU | 比例变化 | 相对变化更有意义 |
| 线程池 | 阈值检测 | 有明确的业务阈值 |

---

## 6. 多序列处理

对于包含多个子序列的指标（如多个 Pod、多个方法），系统支持：

1. **批量检测**：对每个子序列独立运行检测算法
2. **结果聚合**：
   - 统计异常序列数量
   - 找出最严重的异常
   - 生成汇总描述

```python
# 示例：多 Pod 延迟检测
results = detector.detect_batch(pod_series_list)
anomalies = [r for r in results if r.is_anomaly]
worst = max(anomalies, key=lambda r: r.change_ratio)
```

---

## 7. 工具函数

### calc_change_ratio

```python
def calc_change_ratio(current: float, baseline: float) -> float:
    """计算变化比例"""
    if baseline == 0:
        return float('inf') if current > 0 else 0
    return (current - baseline) / baseline
```

### calc_zscore

```python
def calc_zscore(value: float, mean: float, std: float) -> float:
    """计算 Z-Score"""
    if std == 0:
        return 0
    return (value - mean) / std
```

### calc_mad

```python
def calc_mad(values: List[float], median: float = None) -> float:
    """计算 MAD (Median Absolute Deviation)"""
    if not values:
        return 0.0
    if median is None:
        median = statistics.median(values)
    deviations = [abs(x - median) for x in values]
    return statistics.median(deviations)
```
