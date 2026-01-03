# assignTimestampsAndWatermarks()：设置时间特性

## 核心概念

**`assignTimestampsAndWatermarks()`** 用于为 DataStream **分配时间戳和生成 Watermark**，这是使用事件时间处理的前提。

### 核心作用

1. **提取时间戳**：从数据中提取事件时间戳
2. **生成 Watermark**：根据时间戳生成 Watermark
3. **启用事件时间**：让 Flink 使用事件时间处理

## 源码位置

assignTimestampsAndWatermarks() 方法在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 设置时间戳和Watermark
DataStream<Trade> withTimestamps = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 现在可以使用事件时间窗口
withTimestamps.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .sum("quantity")
              .print();
```

## 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 1. 设置Watermark策略（最大乱序10秒）
// 2. 提取时间戳（从Trade的tradeTime字段）
DataStream<Trade> withEventTime = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 使用事件时间窗口
withEventTime.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .aggregate(new TradeAggregator())
              .print();
```

## 完整配置

```java
DataStream<Trade> withTimestamps = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy
        // 1. 设置最大乱序时间
        .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        // 2. 提取时间戳
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
        // 3. 可选：设置空闲超时（如果某个分区没有数据）
        .withIdleness(Duration.ofSeconds(60))
);
```

## 关键要点

1. **必须在窗口前调用**：使用事件时间窗口前必须设置
2. **时间戳提取**：通过 TimestampAssigner 提取
3. **Watermark生成**：根据策略自动生成

## 什么时候你需要想到这个？

- 当你需要使用**事件时间**时（必须先设置）
- 当你需要使用**事件时间窗口**时（必须先设置）
- 当你需要**处理乱序数据**时（通过Watermark）
- 当你需要**提取数据中的时间戳**时（TimestampAssigner）
- 当你需要**启用事件时间处理**时（第一步）


