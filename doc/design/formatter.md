# 格式化器核心逻辑文档


---

## 一、架构设计

### 1.1 策略模式框架

```
┌─────────────────────────────────────────────────────────┐
│                  FormatterRegistry                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │  formatters: List[BaseFormatter] (按 priority 排序) │  │
│  └───────────────────────────────────────────────────┘  │
│                          │                               │
│              format(data, json_path, context)           │
│                          │                               │
│                          ▼                               │
│         遍历 formatters，找到第一个 can_handle=True      │
│                          │                               │
│                          ▼                               │
│              调用 formatter.format() 返回结果            │
└─────────────────────────────────────────────────────────┘
```

### 1.2 BaseFormatter 接口

| 属性/方法 | 说明 |
|-----------|------|
| `priority: int` | 优先级，数值越小越优先，默认 100 |
| `can_handle(data, json_path) -> bool` | 判断是否能处理该数据 |
| `format(data, json_path, context) -> str` | 执行格式化，返回 Markdown |

### 1.3 上下文 Context

```python
context = {
    "navigator": CaseDataNavigator,  # 可访问其他文件
    "current_file": str,             # 当前正在读取的文件名
}
```

---

## 二、通用工具函数

### 2.1 数字格式化

```python
def format_number(num: float) -> str:
    """将大数字转换为可读形式"""
    if num >= 1_000_000_000:
        return f"{num/1_000_000_000:.2f}B"   # 10亿 → 1.00B
    elif num >= 1_000_000:
        return f"{num/1_000_000:.2f}M"       # 100万 → 1.00M
    elif num >= 1_000:
        return f"{num/1_000:.2f}K"           # 1000 → 1.00K
    elif isinstance(num, float) and num != int(num):
        return f"{num:.2f}"                   # 保留2位小数
    else:
        return str(int(num))                  # 整数原样输出
```

### 2.2 Sparkline 生成

```python
SPARK_CHARS = " ▁▂▃▄▅▆▇█"  # 9级高度

def generate_sparkline(values: List[float]) -> str:
    """将数值序列转换为迷你折线图"""
    min_val, max_val = min(values), max(values)
    range_val = max_val - min_val
    
    if range_val == 0:
        return SPARK_CHARS[4] * len(values)  # 全部中等高度
    
    sparkline = []
    for v in values:
        # 归一化到 0-8 范围
        level = int((v - min_val) / range_val * 8)
        level = min(8, max(0, level))
        sparkline.append(SPARK_CHARS[level])
    return "".join(sparkline)

# 示例：[1, 5, 3, 8, 2] → "▁▅▃█▂"
```

### 2.3 异常检测（Sigma 阈值法）

```python
def detect_anomalies(values: List[float], sigma: float = 2.5) -> List[tuple]:
    """检测超过均值 + N 倍标准差的数据点"""
    if len(values) < 3:
        return []
    
    avg = sum(values) / len(values)
    variance = sum((v - avg) ** 2 for v in values) / len(values)
    std = variance ** 0.5
    
    if std == 0:
        return []
    
    threshold = avg + sigma * std
    anomalies = [(i, val) for i, val in enumerate(values) if val > threshold and val > 0]
    anomalies.sort(key=lambda x: x[1], reverse=True)  # 按值降序
    return anomalies
```

**阈值选择：**
- `sigma=2.5`：单序列异常检测（更严格）
- `sigma=2.0`：多序列峰值检测（更敏感）

---

## 三、告警格式化器 (AlertsFormatter)

### 3.1 路径匹配规则

```python
def can_handle(data, json_path):
    # 条件1：是列表
    # 条件2：路径包含 "alerts" 和 "items"
    # 条件3：首元素有 trigger_time, level, content 字段
    return isinstance(data, list) and \
           "alerts" in json_path and "items" in json_path and \
           has_alert_fields(data[0])
```

### 3.2 等级映射

```python
LEVEL_EMOJI = {
    "Disaster": "💀",   # 灾难
    "Critical": "🔴",   # 严重
    "High": "🟠",       # 高
    "Warning": "🟡",    # 警告
    "Info": "🔵",       # 信息
    "Low": "🟢",        # 低
}

# 严重程度排序（用于概览展示）
SEVERITY_ORDER = ["Disaster", "Critical", "High", "Warning", "Info", "Low"]
```

### 3.3 核心格式化逻辑

```
输入: alerts.items 数组
    │
    ▼
① 按等级分组统计 → 概览行
    │
    ▼
② 按时间正序排序（最新在后，LLM 更关注后面内容）
    │
    ▼
③ 按类别分组
    │
    ▼
④ 每类最多显示 10 条，超出提示 "还有 N 条同类告警"
    │
    ▼
输出: Markdown 格式
```

### 3.4 单条告警格式

```markdown
- 💀 **2025-12-08 17:22:00** [Disaster] MySQL连接池耗尽
  - 触发值: 100
  - 位置: cluster-prod / db-master
```

---

## 四、依赖关系格式化器 (DependenciesFormatter)

### 4.1 上下游定义

| 类型 | 路径匹配 | 调用关系 |
|------|----------|----------|
| **Downstream** | `downstream.services` | 当前服务(Client) → 调用 → 下游(Server) |
| **Upstream** | `upstream.clients` | 上游(Client) → 调用 → 当前服务(Server) |

### 4.2 流量等级判断

```python
def get_traffic_level(rpm: float) -> str:
    if rpm >= 100_000:   # 100K+
        return "high"
    elif rpm >= 10_000:  # 10K+
        return "medium"
    else:
        return "normal"
```

### 4.3 服务状态检查

```python
# 通过 navigator 检查服务是否有对应的异常文件
has_issue_file = navigator.get_file(service_name) is not None
```

### 4.4 状态矩阵

| 流量等级 | 有异常文件 | 状态显示 |
|----------|------------|----------|
| high | ✓ | 🚨 高流量 服务自身异常 |
| high | ✗ | ✅ 高流量 服务自身正常 |
| medium | ✓ | 🟡 中流量 服务自身异常 |
| medium | ✗ | ✅ 中流量 |
| normal | ✓ | 🟡 正常 服务自身异常 |
| normal | ✗ | ✅ 正常 |

### 4.5 关键设计原则

> **数据路径 > 名字语义**
> 
> 即使服务名称包含 'Engine'、'Strategy'、'Gateway' 等看起来像上游的词汇，
> 只要它出现在 downstream 列表中，它就是下游（被调用方）！

---

## 五、变更事件格式化器 (ChangeEventsFormatter)

### 5.1 事件类型映射

```python
TYPE_EMOJI = {
    "中间件操作": "⚙️",
    "发布变更": "🚀",
    "配置变更": "📝",
    "容量变更": "📦",
    "数据库操作": "🗄️",
    "网络变更": "🌐",
}

LEVEL_EMOJI = {
    "P0": "🔴",
    "P1": "🟠",
    "P2": "🟡",
    "P3": "🔵",
    "P4": "🟢",
}
```

### 5.2 格式化流程

```
① 按事件类型统计 → 概览
② 按时间正序排序（最新在后）
③ 渲染时间线
```

### 5.3 单事件格式

```markdown
- **2025-12-08 17:22:00** 🔴[P0] 🚀 服务发布
  - 版本从 v1.2.0 升级到 v1.3.0
  - 操作者: zhangsan
  - 状态: ✅ 发布结束
```

### 5.4 详情截断

```python
# 截断过长的详情，保留前 77 字符
if len(detail) > 80:
    detail = detail[:77] + "..."
```

---

## 六、指标格式化器 (MetricsFormatter)

### 6.1 单序列 vs 多序列识别

```python
# 单序列：有 values 字段
{
    "metric_name": "cpu_usage",
    "values": [10, 20, 30, ...]
}

# 多序列：有 pods/methods/tables 等数组字段
{
    "metric_name": "pod_cpu",
    "pods": [
        {"pod": "pod-1", "values": [10, 20, ...]},
        {"pod": "pod-2", "values": [15, 25, ...]}
    ]
}
```

### 6.2 多序列字段识别

```python
SERIES_ARRAY_FIELDS = [
    "pods",            # Pod 级别指标
    "dbkeys",          # 数据库 Key 级别
    "methods",         # 方法级别
    "redis_instances", # Redis 实例级别
    "tables",          # 表级别
    "clusters",        # 集群级别
    "hosts"            # 主机级别
]

SERIES_NAME_FIELDS = {
    "pods": "pod",
    "dbkeys": "dbKey",
    "methods": "method",
    "redis_instances": "redisName",
    "tables": "tableName",
    "clusters": "cluster",
    "hosts": "host"
}
```

### 6.3 低方差检测

```python
LOW_VARIANCE_THRESHOLD = 0.1

def is_low_variance(values, max_val, std):
    """判断序列是否稳定（波动小）"""
    if max_val == 0:
        return True
    return (std / max(max_val, 0.001)) < LOW_VARIANCE_THRESHOLD

# 低方差序列在输出中标记为 "✅ Stable" 或隐藏
```

### 6.4 Dashboard 渲染

```
┌────────────────────────────────────────────────────────────────────────────────┐
│ 序列名称                      Max          趋势 (30min compressed)      状态   │
│────────────────────────────────────────────────────────────────────────────────│
│ 1. pod-app-001               1.25K        ▁▂▃▅█▇▅▃▂▁                    🚨 Spike!  │
│ 2. pod-app-002               856          ▃▃▃▃▃▃▃▃▃▃                    ✅ Stable  │
│ 3. pod-app-003               0            ▁▁▁▁▁▁▁▁▁▁                    💤 Idle    │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 6.5 相关性分析

```python
def analyze_correlation(series_data):
    """检测多序列是否在同一时间窗口出现峰值（级联故障模式）"""
    spiking_series = [(name, spikes) for s in series_data if s["spikes"]]
    
    # 将峰值点按时间桶分组（每 5 个点一桶）
    all_spike_indices = {}
    for name, spikes in spiking_series:
        for idx in spikes:
            bucket = idx // 5
            all_spike_indices.setdefault(bucket, []).append(name)
    
    # 找出同时出现 2 个以上峰值的时间桶
    correlated = [(bucket, names) for bucket, names in all_spike_indices.items() 
                  if len(names) >= 2]
    
    # 输出示例：
    # Index ~10-15: pod-1, pod-2, pod-3 同时出现峰值
```

### 6.6 数据压缩显示

```python
def compress_values(values, max_display=20):
    """压缩长数组，只显示头尾各 10 个"""
    if len(values) <= max_display:
        return str(values)
    
    head = values[:10]
    tail = values[-10:]
    return f"[{head}, ... , {tail}]"
```

---

## 七、优先级设计

| 优先级 | 格式化器 | 说明 |
|--------|----------|------|
| 20 | DownstreamDependenciesFormatter | 依赖关系优先 |
| 20 | UpstreamDependenciesFormatter | |
| 25 | AlertsSummaryFormatter | 概览类 |
| 25 | ChangeEventsSummaryFormatter | |
| 30 | AlertsFormatter | 详情列表类 |
| 30 | ChangeEventsFormatter | |
| 40 | MultiSeriesMetricsFormatter | 多序列优先于单序列 |
| 50 | MetricsTimeSeriesFormatter | 单序列 |
| 100 | (默认) | 兜底 |

---

## 八、设计模式总结

### 8.1 策略模式
- `BaseFormatter` 定义接口
- 各具体格式化器实现 `can_handle` + `format`
- `FormatterRegistry` 管理和调度

### 8.2 Chain of Responsibility（责任链）
- 按优先级排序的格式化器列表
- 依次尝试匹配，第一个匹配成功的处理

### 8.3 LLM 友好设计
- **时间排序**：正序（最新在后），LLM 更关注后面的内容
- **分层概览**：先统计概览，再详情
- **Emoji 标记**：视觉突出重点
- **信息截断**：避免过长内容干扰分析
- **操作提示**：如 `> 💡 使用 cat <服务名> 查看详细数据`

