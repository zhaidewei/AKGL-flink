# Watermark：事件时间的进度指标

## 核心概念

**Watermark** 是 Flink 中用于**跟踪事件时间进度**的机制。它告诉系统"某个时间点之前的数据应该已经到达了"。

### 类比理解

这就像：
- **进度条**：显示处理进度
- **检查点**：标记"到这里的数据应该都到了"
- **时间戳**：表示时间进度

### 核心特点

1. **时间进度**：Watermark(t) 表示时间 t 之前的数据应该已经到达
2. **触发窗口**：当 Watermark >= 窗口结束时间时，窗口触发计算
3. **处理乱序**：允许一定时间的乱序数据

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 设置Watermark：最大乱序时间10秒
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// Watermark会触发窗口计算
withWatermark.keyBy(trade -> trade.getSymbol())
             .window(TumblingEventTimeWindows.of(Time.minutes(1)))
             .sum("quantity")
             .print();
```

## Watermark 的工作原理

### 时间线示例

```
事件时间线:
  12:00:00 - 交易1
  12:00:10 - 交易2
  12:00:20 - 交易3

Watermark推进:
  收到交易1 (事件时间12:00:00) → Watermark = 11:59:50 (12:00:00 - 10秒)
  收到交易2 (事件时间12:00:10) → Watermark = 12:00:00 (12:00:10 - 10秒)
  收到交易3 (事件时间12:00:20) → Watermark = 12:00:10 (12:00:20 - 10秒)

窗口触发:
  当Watermark >= 窗口结束时间时，窗口触发
  例如：[12:00:00, 12:01:00) 窗口在Watermark >= 12:01:00时触发
```

## 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 设置Watermark策略
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    // 最大乱序时间：10秒
    // 意味着：如果当前Watermark是12:00:10，那么12:00:00之前的数据应该已经到达
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 使用事件时间窗口
withWatermark.keyBy(trade -> trade.getSymbol())
             .window(TumblingEventTimeWindows.of(Time.minutes(1)))
             .aggregate(new TradeAggregator())
             .print();
```

## 最大乱序时间

### 设置原则

```java
// 如果数据最大延迟5秒，设置10秒的乱序时间（留有余量）
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10));

// 如果数据最大延迟1分钟，设置2分钟的乱序时间
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofMinutes(2));
```

### 平衡延迟和准确性

```java
// 乱序时间太小：可能丢失延迟数据
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(1));  // 太小

// 乱序时间太大：窗口延迟触发，实时性差
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofMinutes(10));  // 太大

// 合适的值：根据实际数据延迟设置
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10));  // 合适
```

## 什么时候你需要想到这个？

- 当你使用**事件时间**时（必须设置Watermark）
- 当你需要**处理乱序数据**时（Watermark处理乱序）
- 当你需要**触发窗口计算**时（Watermark触发窗口）
- 当你需要**理解事件时间处理**时（Watermark是核心）
- 当你需要**优化窗口延迟**时（调整乱序时间）

