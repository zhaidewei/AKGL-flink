# 滑动窗口（SlidingWindow）：固定大小可重叠

## 核心概念

**滑动窗口（Sliding Window）** 是**固定大小、可重叠**的窗口。窗口之间有重叠，一个数据可能属于多个窗口。

### 类比理解

这就像：
- **滑动平均**：计算最近N个数据的平均值，每次滑动一个位置
- **移动窗口**：窗口在时间轴上滑动
- **重叠采样**：采样窗口之间有重叠

### 核心特点

1. **固定大小**：窗口大小固定（如1分钟）
2. **可重叠**：窗口之间有重叠（如滑动间隔30秒）
3. **一个数据多窗口**：一个数据可能属于多个窗口

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 滑动窗口：窗口大小1分钟，滑动间隔30秒
trades.keyBy(trade -> trade.getSymbol())
      .window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(30)))
      .sum("quantity")
      .print();

// 窗口划分：
// [12:00:00, 12:01:00) - 窗口1
// [12:00:30, 12:01:30) - 窗口2（与窗口1重叠30秒）
// [12:01:00, 12:02:00) - 窗口3（与窗口2重叠30秒）
```

## 窗口划分示例

```
时间线:
  12:00:00 ────────── 12:00:30 ────────── 12:01:00 ────────── 12:01:30
    │                    │                    │                    │
    └── 窗口1 ───────────┼────────────────────┘                    │
                        │                                           │
                        └── 窗口2 ──────────────────────────────────┼──
                                                                    │
                                                                    └── 窗口3 ──

数据分配:
  12:00:10的交易 → 窗口1, 窗口2（属于两个窗口）
  12:00:50的交易 → 窗口1, 窗口2（属于两个窗口）
  12:01:10的交易 → 窗口2, 窗口3（属于两个窗口）
```

## 币安交易数据示例

### 滑动平均价格

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 滑动窗口：窗口1分钟，滑动30秒
// 计算最近1分钟的平均价格，每30秒更新一次
trades.keyBy(trade -> trade.getSymbol())
      .window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(30)))
      .aggregate(new AveragePriceAggregator())
      .print();
```

### 滑动交易量统计

```java
// 统计最近5分钟的交易量，每1分钟更新一次
trades.keyBy(trade -> trade.getSymbol())
      .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
      .sum("quantity")
      .print();
```

## 窗口大小和滑动间隔

```java
// 窗口大小：1分钟
// 滑动间隔：30秒
// 重叠：30秒
.window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(30)))

// 窗口大小：5分钟
// 滑动间隔：1分钟
// 重叠：4分钟
.window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
```

## 与滚动窗口的区别

| 特性 | 滚动窗口 | 滑动窗口 |
|------|---------|---------|
| 重叠 | ❌ 不重叠 | ✅ 可重叠 |
| 窗口数量 | 少 | 多（重叠越多，窗口越多） |
| 计算量 | 小 | 大（一个数据可能计算多次） |
| 实时性 | 固定间隔 | 更频繁更新 |
| 适用场景 | 固定间隔统计 | 滑动平均、趋势分析 |

## 使用场景

### 场景1：滑动平均

```java
// 计算最近1分钟的平均价格，每30秒更新
.window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(30)))
```

### 场景2：趋势分析

```java
// 分析最近5分钟的趋势，每1分钟更新
.window(SlidingEventTimeWindows.of(Time.minutes(5), Time.minutes(1)))
```

## 什么时候你需要想到这个？

- 当你需要**滑动平均**时（最近N分钟的平均值）
- 当你需要**更频繁的更新**时（滑动窗口更新更频繁）
- 当你需要**趋势分析**时（重叠窗口可以看到趋势）
- 当你需要理解**窗口类型**时（滚动 vs 滑动）
- 当你需要**优化计算性能**时（滑动窗口计算量大）


