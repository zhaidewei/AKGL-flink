# 周期性Watermark生成：定时生成策略

## 核心概念

**周期性 Watermark 生成** 是每隔一定时间（如每200ms）生成一次 Watermark 的策略。这是 Flink 中常用的 Watermark 生成方式。

### 核心特点

1. **定时生成**：每隔固定时间生成一次
2. **基于最新数据**：基于最新收到数据的时间戳生成
3. **性能好**：不需要每条数据都生成

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 周期性生成Watermark（默认每200ms）
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
        // 默认就是周期性生成，每200ms生成一次
);

withWatermark.keyBy(trade -> trade.getSymbol())
             .window(TumblingEventTimeWindows.of(Time.minutes(1)))
             .sum("quantity")
             .print();
```

## 工作原理

### 生成流程

```
时间线:
  12:00:00.000 - 收到交易1（事件时间12:00:00）
  12:00:00.100 - 收到交易2（事件时间12:00:05）
  12:00:00.200 - 定时器触发 → 生成Watermark = 11:59:55（基于最新数据12:00:05 - 10秒）
  12:00:00.300 - 收到交易3（事件时间12:00:10）
  12:00:00.400 - 定时器触发 → 生成Watermark = 12:00:00（基于最新数据12:00:10 - 10秒）
```

### 默认配置

```java
// Flink默认每200ms生成一次Watermark
// 可以通过配置修改
env.getConfig().setAutoWatermarkInterval(200);  // 200ms
```

## 自定义生成间隔

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 设置Watermark生成间隔：100ms（更频繁）
env.getConfig().setAutoWatermarkInterval(100);

// 或500ms（更少）
env.getConfig().setAutoWatermarkInterval(500);

DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

## 与标点式Watermark的对比

| 特性 | 周期性生成 | 标点式生成 |
|------|-----------|-----------|
| 生成时机 | 定时（每200ms） | 基于特定数据 |
| 性能 | 好（批量生成） | 较好 |
| 实时性 | 稍差（最多延迟200ms） | 好（立即生成） |
| 使用频率 | ⭐⭐⭐⭐⭐ | ⭐⭐ |

## 币安交易数据示例

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 周期性生成Watermark（每200ms）
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
        // 默认周期性生成，不需要额外配置
);

// 窗口会在Watermark触发时计算
withWatermark.keyBy(trade -> trade.getSymbol())
             .window(TumblingEventTimeWindows.of(Time.minutes(1)))
             .sum("quantity")
             .print();
```

## 优化建议

### 调整生成间隔

```java
// 高吞吐场景：增大间隔（减少生成频率）
env.getConfig().setAutoWatermarkInterval(500);  // 500ms

// 低延迟场景：减小间隔（更频繁生成）
env.getConfig().setAutoWatermarkInterval(100);  // 100ms
```

## 什么时候你需要想到这个？

- 当你使用**事件时间**时（默认就是周期性生成）
- 当你需要**优化Watermark性能**时（调整生成间隔）
- 当你需要**平衡延迟和性能**时（间隔大小的影响）
- 当你需要理解**Watermark生成机制**时（周期性 vs 标点式）
- 当你需要**调试Watermark问题**时（检查生成频率）

