# 事件时间（Event Time）：数据产生的时间

## 核心概念

**事件时间（Event Time）** 是**数据本身产生的时间**，通常包含在数据记录中。对于币安交易数据，事件时间就是交易实际发生的时间。

### 类比理解

这就像：
- **交易时间**：交易实际发生的时间（不是处理时间）
- **日志时间戳**：日志记录的事件发生时间
- **传感器数据**：传感器采集数据的时间

### 核心特点

1. **数据自带**：时间戳来自数据本身
2. **准确性高**：反映事件真实发生时间
3. **支持乱序**：通过 Watermark 处理乱序数据

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 设置事件时间（从Trade中提取时间戳）
DataStream<Trade> withTimestamps = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 事件时间窗口（基于交易实际发生时间）
withTimestamps.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .sum("quantity")
              .print();
```

## 币安交易数据示例

### Trade 数据类

```java
public class Trade implements Serializable {
    private String symbol;
    private double price;
    private double quantity;
    private long tradeTime;  // 事件时间（币安返回的交易时间）

    public long getTradeTime() {
        return tradeTime;  // 这是事件时间
    }
}
```

### 使用事件时间

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 1. 提取事件时间戳
DataStream<Trade> withEventTime = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 2. 使用事件时间窗口
withEventTime.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .aggregate(new TradeAggregator())
              .print();
```

## 时间戳提取

```java
// 方式1：使用TimestampAssigner
WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());

// 方式2：实现TimestampAssigner接口
public class TradeTimestampAssigner implements TimestampAssigner<Trade> {
    @Override
    public long extractTimestamp(Trade trade, long recordTimestamp) {
        return trade.getTradeTime();  // 返回事件时间
    }
}
```

## 与处理时间的区别

| 特性 | 处理时间 | 事件时间 |
|------|---------|---------|
| 时间来源 | 系统时钟 | 数据本身 |
| 准确性 | 受处理速度影响 | 准确（事件真实时间） |
| 乱序处理 | 不支持 | 支持（Watermark） |
| 适用场景 | 实时性优先 | 准确性优先 |

## 使用场景

### 场景1：需要准确的时间窗口

```java
// 币安交易数据：需要基于交易实际发生时间统计
trades.assignTimestampsAndWatermarks(...)
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity");
```

### 场景2：处理乱序数据

```java
// 数据可能乱序到达，但需要按事件时间处理
trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10))
);
```

## 什么时候你需要想到这个？

- 当你需要**基于数据真实时间处理**时（如币安交易时间）
- 当你需要**准确的时间窗口**时（基于事件时间）
- 当你需要**处理乱序数据**时（通过Watermark）
- 当你处理**时间序列数据**时（传感器、交易等）
- 当你需要**精确的时间计算**时（统计、聚合等）

