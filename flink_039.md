# Watermark的作用：告诉系统时间进度

## 核心概念

**Watermark** 的主要作用是**告诉 Flink 系统事件时间的进度**，让系统知道"某个时间点之前的数据应该已经到达了"。

### 核心作用

1. **时间进度跟踪**：Watermark(t) 表示时间 t 之前的数据应该已经到达
2. **窗口触发**：当 Watermark >= 窗口结束时间时，触发窗口计算
3. **乱序处理**：允许一定时间内的乱序数据

## 工作原理

### 时间进度示例

```
事件时间线:
  12:00:00 - 交易1
  12:00:10 - 交易2
  12:00:20 - 交易3

Watermark推进（最大乱序10秒）:
  收到交易1 (12:00:00) → Watermark = 11:59:50
    "11:59:50之前的数据应该已经到达"

  收到交易2 (12:00:10) → Watermark = 12:00:00
    "12:00:00之前的数据应该已经到达"

  收到交易3 (12:00:20) → Watermark = 12:00:10
    "12:00:10之前的数据应该已经到达"
```

### 窗口触发机制

```java
// 窗口：[12:00:00, 12:01:00)
// 当Watermark >= 12:01:00时，窗口触发计算

时间线:
  12:00:00 - 窗口开始
  12:00:30 - 收到数据，Watermark = 12:00:20（未触发）
  12:00:50 - 收到数据，Watermark = 12:00:40（未触发）
  12:01:10 - 收到数据，Watermark = 12:01:00（触发！）
    → 窗口计算，输出结果
```

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 设置Watermark：告诉系统时间进度
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// Watermark会触发窗口
withWatermark.keyBy(trade -> trade.getSymbol())
             .window(TumblingEventTimeWindows.of(Time.minutes(1)))
             .sum("quantity")
             .print();
```

## 币安交易数据示例

### 场景：统计每分钟交易量

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 设置Watermark：最大乱序10秒
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 每分钟窗口：当Watermark >= 窗口结束时间时触发
withWatermark.keyBy(trade -> trade.getSymbol())
             .window(TumblingEventTimeWindows.of(Time.minutes(1)))
             .aggregate(new TradeAggregator())
             .print();
```

### 处理流程

```
1. 收到交易数据（事件时间12:00:05）
   → 更新Watermark = 11:59:55（12:00:05 - 10秒）
   → 窗口[12:00:00, 12:01:00)还未触发（Watermark < 12:01:00）

2. 收到更多数据（事件时间12:00:50）
   → 更新Watermark = 12:00:40（12:00:50 - 10秒）
   → 窗口还未触发

3. 收到数据（事件时间12:01:10）
   → 更新Watermark = 12:01:00（12:01:10 - 10秒）
   → 窗口触发！计算[12:00:00, 12:01:00)的数据
```

## 关键理解

1. **Watermark是时间进度**：告诉系统"到这里的数据应该都到了"
2. **触发窗口**：Watermark >= 窗口结束时间时触发
3. **处理乱序**：允许一定时间内的延迟数据

## 什么时候你需要想到这个？

- 当你需要理解**Watermark的作用**时（时间进度跟踪）
- 当你需要**触发窗口计算**时（Watermark触发窗口）
- 当你需要**处理乱序数据**时（Watermark处理乱序）
- 当你需要**优化窗口延迟**时（调整Watermark策略）
- 当你需要**调试窗口问题**时（检查Watermark是否正确）

