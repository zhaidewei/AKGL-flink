# 摄入时间（Ingestion Time）：数据进入Flink的时间

## 核心概念

**摄入时间（Ingestion Time）** 是数据**进入 Flink 系统的时间**，介于事件时间和处理时间之间。它是数据源算子接收到数据时的时间戳。

### 类比理解

这就像：
- **入库时间**：数据进入数据库的时间
- **接收时间戳**：系统接收到数据的时间
- **中间时间**：介于事件发生时间和处理时间之间

### 核心特点

1. **自动分配**：Flink 自动分配，不需要提取时间戳
2. **介于两者之间**：比事件时间晚，比处理时间早
3. **较少使用**：大多数场景使用事件时间或处理时间

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 设置时间特性为摄入时间
env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);

DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 摄入时间窗口
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingIngestionTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .print();
```

## 三种时间的对比

| 时间类型 | 时间来源 | 准确性 | 性能 | 使用频率 |
|---------|---------|--------|------|---------|
| **事件时间** | 数据本身 | 最高 | 较慢 | ⭐⭐⭐⭐⭐ |
| **摄入时间** | Flink接收时 | 中等 | 中等 | ⭐ |
| **处理时间** | 系统时钟 | 最低 | 最快 | ⭐⭐⭐⭐ |

## 时间线示例

```
事件发生时间（事件时间）: 12:00:00
    ↓ 网络传输
数据进入Flink（摄入时间）: 12:00:05
    ↓ 排队等待
数据被处理（处理时间）: 12:00:10
```

## 使用场景

### 场景1：数据没有时间戳

```java
// 如果数据本身没有时间戳，但需要时间窗口
// 可以使用摄入时间（比处理时间更准确）
env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
```

### 场景2：折中方案

```java
// 需要时间窗口，但数据没有时间戳，也不想用处理时间
// 摄入时间是折中方案
```

## 注意事项

1. **较少使用**：大多数场景使用事件时间或处理时间
2. **已废弃**：Flink 1.12+ 中，摄入时间特性已被标记为废弃
3. **推荐使用事件时间**：如果数据有时间戳，使用事件时间

## 现代 Flink 的推荐

```java
// 推荐：使用事件时间（如果数据有时间戳）
trades.assignTimestampsAndWatermarks(...)
      .window(TumblingEventTimeWindows.of(...));

// 或者：使用处理时间（如果不需要时间戳）
trades.window(TumblingProcessingTimeWindows.of(...));
```

## 什么时候你需要想到这个？

- 当你需要理解 Flink 的**三种时间概念**时（完整性）
- 当你遇到**旧代码使用摄入时间**时（了解概念）
- 当你需要**选择合适的时间特性**时（通常不选这个）
- 当你学习 Flink 的**时间系统**时（了解所有选项）
- 当你需要**处理没有时间戳的数据**时（但这种情况很少）

