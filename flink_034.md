# 处理时间（Processing Time）：系统时间

## 核心概念

**处理时间（Processing Time）** 是 Flink **处理数据时的系统时间**。它是数据进入 Flink 算子并开始处理时的机器时钟时间。

### 类比理解

这就像：
- **日志时间戳**：记录日志时的系统时间
- **数据库插入时间**：数据插入数据库时的系统时间
- **处理时间戳**：处理数据时的当前时间

### 核心特点

1. **系统时间**：使用 Flink 运行机器的系统时钟
2. **简单快速**：不需要提取时间戳，性能最好
3. **不确定性**：受处理速度影响，可能乱序

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 默认使用处理时间
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 处理时间模式下，时间戳是Flink处理数据时的系统时间
trades.map(trade -> {
    long processingTime = System.currentTimeMillis();  // 当前系统时间
    return trade;
});
```

## 时间戳来源

```java
// 处理时间模式下，Flink自动使用系统时间
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 在ProcessFunction中获取处理时间
trades.process(new ProcessFunction<Trade, Trade>() {
    @Override
    public void processElement(Trade trade, Context ctx, Collector<Trade> out) {
        // 获取处理时间（系统时间）
        long processingTime = ctx.timerService().currentProcessingTime();

        // 处理时间就是当前系统时间
        System.out.println("Processing time: " + processingTime);
    }
});
```

## 使用场景

### 场景1：不需要事件时间

```java
// 如果不需要基于事件时间处理，使用处理时间最简单
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 处理时间窗口（每5秒统计一次）
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
      .sum("quantity")
      .print();
```

### 场景2：实时性要求高

```java
// 处理时间延迟最低，适合实时性要求高的场景
trades.window(TumblingProcessingTimeWindows.of(Time.seconds(1)))
      .aggregate(new TradeAggregator());
```

## 与事件时间的区别

| 特性 | 处理时间 | 事件时间 |
|------|---------|---------|
| 时间来源 | 系统时钟 | 数据本身 |
| 性能 | 最快 | 较慢（需要提取时间戳） |
| 准确性 | 受处理速度影响 | 准确（基于数据时间） |
| 乱序处理 | 不支持 | 支持（通过Watermark） |

## 注意事项

1. **受处理速度影响**：如果处理慢，时间戳会延迟
2. **可能乱序**：数据到达顺序和处理顺序可能不一致
3. **不适合精确计算**：不适合需要精确时间窗口的场景

## 什么时候你需要想到这个？

- 当你需要**快速处理数据**时（处理时间性能最好）
- 当你**不需要精确的时间窗口**时（不需要事件时间）
- 当你需要**实时性优先**时（延迟要求高）
- 当你**数据本身没有时间戳**时（无法使用事件时间）
- 当你学习 Flink 的**时间概念**时（从最简单的开始）

