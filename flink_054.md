# 窗口触发：Watermark如何触发窗口计算

## 核心概念

**窗口触发**是指窗口在什么时候执行计算。对于事件时间窗口，窗口的触发由 **Watermark** 决定。

### 核心机制

1. **Watermark推进**：Watermark表示事件时间的进度
2. **窗口结束时间**：每个窗口有结束时间
3. **触发条件**：当 Watermark >= 窗口结束时间时，窗口触发

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 窗口：[12:00:00, 12:01:00)
// 当Watermark >= 12:01:00时，窗口触发
withWatermark.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .sum("quantity")
              .print();
```

## 触发机制示例

### 时间线

```
事件时间线:
  12:00:00 - 交易1
  12:00:30 - 交易2
  12:00:50 - 交易3
  12:01:10 - 交易4

Watermark推进（最大乱序10秒）:
  收到交易1 (12:00:00) → Watermark = 11:59:50
  收到交易2 (12:00:30) → Watermark = 12:00:20
  收到交易3 (12:00:50) → Watermark = 12:00:40
  收到交易4 (12:01:10) → Watermark = 12:01:00  ← 触发！

窗口触发:
  窗口[12:00:00, 12:01:00)在Watermark = 12:01:00时触发
  计算窗口内的数据：交易1、交易2、交易3
```

## 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 设置Watermark：最大乱序10秒
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 1分钟窗口
// 窗口[12:00:00, 12:01:00)在Watermark >= 12:01:00时触发
withWatermark.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .sum("quantity")
              .print();
```

## 触发延迟

### 延迟计算

```
窗口结束时间: 12:01:00
最大乱序时间: 10秒

最早触发时间: 12:01:10（收到事件时间12:01:10的数据时）
  Watermark = 12:01:10 - 10秒 = 12:01:00
  → 窗口触发

延迟: 10秒（最大乱序时间）
```

### 减少延迟

```java
// 减少最大乱序时间，可以减少触发延迟
// 但可能丢失延迟数据
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5));  // 延迟5秒
```

## 处理时间窗口 vs 事件时间窗口

| 窗口类型 | 触发方式 | 延迟 |
|---------|---------|------|
| 处理时间窗口 | 系统时间到达 | 无延迟 |
| 事件时间窗口 | Watermark到达 | 有延迟（乱序时间） |

## 什么时候你需要想到这个？

- 当你需要理解**窗口何时触发**时（Watermark机制）
- 当你需要**优化窗口延迟**时（调整乱序时间）
- 当你需要**调试窗口问题**时（检查Watermark）
- 当你需要**理解事件时间处理**时（触发机制）
- 当你需要**平衡延迟和准确性**时（触发时机）


