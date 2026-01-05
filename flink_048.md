# 滚动窗口（TumblingWindow）：固定大小不重叠

## 核心概念

**滚动窗口（Tumbling Window）** 是**固定大小、不重叠**的窗口。每个数据只属于一个窗口，窗口之间没有重叠。

### 类比理解

这就像：
- **时间桶**：将时间分成固定大小的桶，每个桶不重叠
- **小时统计**：每小时统计一次，不重叠
- **固定间隔采样**：每隔固定时间采样一次

### 核心特点

1. **固定大小**：窗口大小固定（如1分钟）
2. **不重叠**：窗口之间没有重叠
3. **对齐时间**：窗口边界对齐到时间点（如整点、整分）

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 滚动窗口：1分钟，不重叠
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .print();

// 窗口划分：
// [12:00:00, 12:01:00) - 窗口1
// [12:01:00, 12:02:00) - 窗口2
// [12:02:00, 12:03:00) - 窗口3
// （不重叠）
```

## 窗口划分示例

```
时间线:
  12:00:00 ────────── 12:01:00 ────────── 12:02:00 ────────── 12:03:00
    │                    │                    │                    │
    └── 窗口1 ───────────┘                    │                    │
                                              │                    │
                                              └── 窗口2 ───────────┘
                                                                   │
                                                                   └── 窗口3 ──

数据分配:
  12:00:10的交易 → 窗口1
  12:00:50的交易 → 窗口1
  12:01:10的交易 → 窗口2
  12:01:50的交易 → 窗口2
  （每个数据只属于一个窗口）
```

## 币安交易数据示例

### 统计每分钟交易量

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 滚动窗口：1分钟
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .print();

// 结果：
// [12:00:00, 12:01:00) - BTC: 100, ETH: 50
// [12:01:00, 12:02:00) - BTC: 150, ETH: 80
// [12:02:00, 12:03:00) - BTC: 120, ETH: 60
```

### 统计每小时交易额

```java
// 滚动窗口：1小时
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.hours(1)))
      .aggregate(new AmountAggregator())  // 计算交易额
      .print();
```

## 不同时间单位

```java
// 秒级窗口
.window(TumblingEventTimeWindows.of(Time.seconds(10)))

// 分钟级窗口
.window(TumblingEventTimeWindows.of(Time.minutes(1)))

// 小时级窗口
.window(TumblingEventTimeWindows.of(Time.hours(1)))

// 天级窗口
.window(TumblingEventTimeWindows.of(Time.days(1)))
```

## 与滑动窗口的区别

| 特性 | 滚动窗口 | 滑动窗口 |
|------|---------|---------|
| 重叠 | ❌ 不重叠 | ✅ 可重叠 |
| 窗口数量 | 少 | 多 |
| 计算量 | 小 | 大 |
| 适用场景 | 固定间隔统计 | 滑动平均 |

## 什么时候你需要想到这个？

- 当你需要**固定间隔统计**时（每分钟、每小时）
- 当你需要**不重叠的窗口**时（每个数据只属于一个窗口）
- 当你需要**统计交易量、价格等**时（滚动窗口最常用）
- 当你需要**理解窗口类型**时（从最简单的开始）
- 当你需要**优化计算性能**时（滚动窗口计算量小）


