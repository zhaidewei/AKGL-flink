# SourceReader的时间戳处理：事件时间支持

> ✅ **重要提示**：在新 Source API 中，时间戳处理通过 `WatermarkStrategy` 和 `ReaderOutput.collect()` 配合实现。它替代了 Legacy API 中的 `SourceContext.collectWithTimestamp()` 方法。

## 核心概念

在新 Source API 中，**时间戳处理**主要通过以下方式实现：
1. **`ReaderOutput.collect()`** - 发送数据（时间戳通过 WatermarkStrategy 分配）
2. **`WatermarkStrategy`** - 在创建 DataStream 时指定，用于提取时间戳和生成 Watermark

### 类比理解

这就像：
- **带时间戳的日志**：不仅记录内容，还记录时间
- **时间序列数据**：每个数据点都有对应的时间
- **事件溯源**：记录事件发生的时间

### 核心特点

1. **分离关注点**：时间戳提取和 Watermark 生成在 WatermarkStrategy 中处理
2. **更灵活**：可以在创建 DataStream 时指定不同的时间戳策略
3. **支持事件时间**：完全支持基于事件时间的窗口、Watermark 等

## 实现方式

### 方式1：在 WatermarkStrategy 中提取时间戳

```java
// 1. 创建 Source
BinanceWebSocketSource source = new BinanceWebSocketSource("btcusdt");

// 2. 创建 WatermarkStrategy（提取时间戳）
WatermarkStrategy<Trade> watermarkStrategy = WatermarkStrategy
    .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());

// 3. 创建 DataStream（自动应用时间戳）
DataStream<Trade> trades = env.fromSource(
    source,
    watermarkStrategy,
    "Binance Trade Source"
);
```

### 方式2：在 SourceReader 中发送带时间戳的数据（高级用法）

对于需要更精细控制的场景，可以在 SourceReader 中直接处理时间戳：

```java
public class BinanceWebSocketReader implements SourceReader<Trade, BinanceWebSocketSplit> {
    @Override
    public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
        Trade trade = recordQueue.poll();
        if (trade != null) {
            // 发送数据（时间戳会在 WatermarkStrategy 中提取）
            output.collect(trade);
            return InputStatus.MORE_AVAILABLE;
        }
        return InputStatus.NOTHING_AVAILABLE;
    }
}
```

## 最小可用例子

### 完整示例

```java
// 1. Trade 数据类（包含时间戳字段）
public class Trade implements Serializable {
    private String symbol;
    private double price;
    private double quantity;
    private long tradeTime;  // 事件时间（毫秒）

    public long getTradeTime() {
        return tradeTime;
    }
    // ... getter/setter
}

// 2. 创建 Source
BinanceWebSocketSource source = new BinanceWebSocketSource("btcusdt");

// 3. 创建 WatermarkStrategy（提取时间戳并生成 Watermark）
WatermarkStrategy<Trade> watermarkStrategy = WatermarkStrategy
    .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))  // 最大乱序时间 5 秒
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());  // 提取事件时间

// 4. 创建 DataStream
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<Trade> trades = env.fromSource(
    source,
    watermarkStrategy,
    "Binance Trade Source"
);

// 5. 使用事件时间窗口
trades
    .keyBy(Trade::getSymbol)
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new TradeAggregator())
    .print();

env.execute();
```

## 时间戳格式

时间戳必须是**毫秒数**，从 1970-01-01 00:00:00 UTC 开始：

```java
// 方式1：从数据中提取（币安返回的时间戳）
long timestamp = trade.getTradeTime();

// 方式2：使用 System.currentTimeMillis()
long timestamp = System.currentTimeMillis();

// 方式3：解析时间字符串
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
long timestamp = sdf.parse("2024-01-01 12:00:00").getTime();
```

## 使用场景

### 场景1：事件时间处理（币安交易数据）

```java
// 币安交易数据包含交易时间，使用事件时间处理
WatermarkStrategy<Trade> watermarkStrategy = WatermarkStrategy
    .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());

DataStream<Trade> trades = env.fromSource(source, watermarkStrategy, "Binance Source");
```

### 场景2：处理乱序数据

```java
// 数据可能乱序到达，但包含事件时间，Flink可以正确处理
WatermarkStrategy<MyEvent> watermarkStrategy = WatermarkStrategy
    .<MyEvent>forBoundedOutOfOrderness(Duration.ofSeconds(10))
    .withTimestampAssigner((event, timestamp) -> event.getEventTime());
```

## 与 Legacy API 的对比

| 特性 | Legacy collectWithTimestamp() | 新 WatermarkStrategy |
|------|------------------------------|---------------------|
| 时间戳指定 | 在 collectWithTimestamp() 中 | 在 WatermarkStrategy 中 |
| Watermark 生成 | 手动调用 emitWatermark() | 自动生成 |
| 灵活性 | 较低 | 更高 |
| 推荐度 | ⚠️ 不推荐 | ✅ 推荐 |

### 示例对比

```java
// Legacy API（不推荐）
@Override
public void run(SourceContext<Trade> ctx) throws Exception {
    synchronized (ctx.getCheckpointLock()) {
        ctx.collectWithTimestamp(trade, trade.getTradeTime());  // 在 Source 中指定时间戳
    }
}

// 新 API（推荐）
// 在 SourceReader 中
@Override
public InputStatus pollNext(ReaderOutput<Trade> output) throws Exception {
    output.collect(trade);  // 只发送数据
    return InputStatus.MORE_AVAILABLE;
}

// 在创建 DataStream 时
WatermarkStrategy<Trade> watermarkStrategy = WatermarkStrategy
    .<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime());  // 在策略中提取时间戳

DataStream<Trade> trades = env.fromSource(source, watermarkStrategy, "Source");
```

## 注意事项

1. **时间戳单位**：必须是毫秒（不是秒）
2. **时间戳来源**：应该来自数据本身（事件时间），不是当前系统时间
3. **WatermarkStrategy**：必须在创建 DataStream 时指定

## 什么时候你需要想到这个？

- 当你需要**事件时间处理**时（窗口、Watermark等）
- 当你处理**币安交易数据**时（数据包含交易时间）
- 当你需要处理**乱序数据**时（基于事件时间排序）
- 当你需要**时间相关的计算**时（如计算1分钟内的交易量）
- 当你实现**实时数据源**时（数据本身包含时间戳）
