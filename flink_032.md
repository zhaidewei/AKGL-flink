# KeyedStream：分组后的流类型

## 核心概念

**KeyedStream** 是 `keyBy()` 操作后返回的流类型。它表示数据已经按 key 分组，相同 key 的数据会被路由到同一个并行任务中处理。

### 类比理解

这就像：
- **分组后的表**：SQL 中 `GROUP BY` 后的结果
- **HashMap**：相同 key 的数据在同一个桶中
- **分区流**：数据按 key 分区，每个分区处理特定 key 的数据

### 核心特点

1. **状态隔离**：不同 key 的状态完全隔离
2. **窗口支持**：只有 KeyedStream 才能使用窗口
3. **状态支持**：只有 KeyedStream 才能使用状态（ValueState、MapState等）

## 源码位置

KeyedStream 类定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/KeyedStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/KeyedStream.java)

### 核心结构（伪代码）

```java
public class KeyedStream<T, KEY> extends DataStream<T> {
    // key选择器
    private final KeySelector<T, KEY> keySelector;

    // key的类型信息
    private final TypeInformation<KEY> keyType;

    // KeyedStream支持窗口操作
    public <W extends Window> WindowedStream<T, KEY, W> window(WindowAssigner<T, W> assigner) {
        // ...
    }

    // KeyedStream支持状态操作（通过ProcessFunction）
    public <R> SingleOutputStreamOperator<R> process(KeyedProcessFunction<T, R> function) {
        // ...
    }
}
```

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Trade> trades = env.addSource(new BinanceSource());

// keyBy() 返回 KeyedStream
KeyedStream<Trade, String> keyedTrades = trades.keyBy(trade -> trade.getSymbol());

// KeyedStream 可以使用窗口
keyedTrades.window(TumblingEventTimeWindows.of(Time.minutes(1)))
           .sum("quantity")
           .print();

// KeyedStream 可以使用状态（通过ProcessFunction）
keyedTrades.process(new KeyedProcessFunction<String, Trade, String>() {
    private ValueState<Double> priceState;

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 访问状态
        Double lastPrice = priceState.value();
        priceState.update(trade.getPrice());
    }
});

env.execute();
```

## 与 DataStream 的区别

| 特性 | DataStream | KeyedStream |
|------|-----------|-------------|
| 状态 | ❌ 不支持 | ✅ 支持 |
| 窗口 | ❌ 不支持 | ✅ 支持 |
| keyBy | ✅ 可以 | ❌ 不可以（已经是分组的） |
| map/filter | ✅ 支持 | ✅ 支持 |

## 数据分区机制

```
原始流: [BTC交易1, ETH交易1, BTC交易2, ETH交易2, BTC交易3]

keyBy(symbol) 后生成 KeyedStream:
  Key=BTC: [BTC交易1, BTC交易2, BTC交易3]  → 分区1（并行任务1）
  Key=ETH: [ETH交易1, ETH交易2]            → 分区2（并行任务2）
```

## 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 按交易对分组
KeyedStream<Trade, String> keyedBySymbol = trades.keyBy(trade -> trade.getSymbol());

// 现在可以对每个交易对独立处理
keyedBySymbol
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new TradeAggregator())
    .print();
```

## 关键要点

1. **必须先 keyBy**：只有 KeyedStream 才能使用窗口和状态
2. **状态隔离**：不同 key 的状态完全隔离，互不影响
3. **并行度**：keyBy 会改变数据分区，影响并行度
4. **key 的选择**：应该选择分布均匀的字段（避免数据倾斜）

## 什么时候你需要想到这个？

- 当你需要使用**窗口操作**时（必须先keyBy）
- 当你需要使用**状态**时（必须先keyBy）
- 当你需要**按key分组处理数据**时（如按交易对分组）
- 当你需要理解 Flink 的**数据分区机制**时
- 当你需要**实现聚合操作**时（按key聚合）

