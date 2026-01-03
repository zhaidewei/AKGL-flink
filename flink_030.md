# keyBy()：按key分组数据流

## 核心概念

**`keyBy()`** 用于将 DataStream 按指定的 key 分组，生成 **KeyedStream**。分组后，相同 key 的数据会被路由到同一个并行任务中处理。

### 类比理解

这就像：
- **SQL 的 GROUP BY**：`SELECT * FROM trades GROUP BY symbol`
- **HashMap**：相同 key 的数据放在同一个桶中
- **Spark 的 groupByKey()**：如果你用过 Spark，Flink 的 keyBy() 类似

### 核心特点

1. **数据分区**：相同 key 的数据在同一分区
2. **状态隔离**：不同 key 的状态是隔离的
3. **窗口和状态的前提**：只有 KeyedStream 才能使用窗口和状态

## 源码位置

keyBy() 方法在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

## 最小可用例子

### 方式1：Lambda 表达式（推荐）

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Trade> trades = env.addSource(new BinanceSource());

// 按交易对分组
KeyedStream<Trade, String> keyedTrades = trades.keyBy(trade -> trade.getSymbol());

// 按价格范围分组（自定义key）
KeyedStream<Trade, String> priceGroups = trades.keyBy(trade -> {
    if (trade.getPrice() > 50000) return "HIGH";
    else if (trade.getPrice() > 30000) return "MEDIUM";
    else return "LOW";
});
```

### 方式2：字段索引（Tuple类型）

```java
// 如果数据是Tuple类型
DataStream<Tuple2<String, Double>> data = ...;

// 按第一个字段分组
KeyedStream<Tuple2<String, Double>, Tuple> keyed = data.keyBy(0);

// 按第二个字段分组
KeyedStream<Tuple2<String, Double>, Tuple> keyed = data.keyBy(1);
```

### 方式3：实现 KeySelector

```java
public class SymbolKeySelector implements KeySelector<Trade, String> {
    @Override
    public String getKey(Trade trade) throws Exception {
        return trade.getSymbol();
    }
}

// 使用
KeyedStream<Trade, String> keyed = trades.keyBy(new SymbolKeySelector());
```

## 币安交易数据示例

### 按交易对分组

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 按交易对分组（BTC、ETH等）
KeyedStream<Trade, String> keyedBySymbol = trades.keyBy(trade -> trade.getSymbol());

// 现在可以对每个交易对独立处理
keyedBySymbol.map(trade -> {
    // 这里处理的是同一个交易对的数据
    return processTrade(trade);
});
```

### 按价格区间分组

```java
// 按价格区间分组
KeyedStream<Trade, String> priceGroups = trades.keyBy(trade -> {
    double price = trade.getPrice();
    if (price > 50000) return "HIGH";
    else if (price > 30000) return "MEDIUM";
    else return "LOW";
});
```

### 组合key

```java
// 按交易对和价格区间组合分组
KeyedStream<Trade, String> combined = trades.keyBy(trade -> {
    return trade.getSymbol() + "_" +
           (trade.getPrice() > 50000 ? "HIGH" : "LOW");
});
```

## 数据分区机制

```
原始流: [BTC交易1, ETH交易1, BTC交易2, ETH交易2, BTC交易3]

keyBy(symbol) 后:
  Key=BTC: [BTC交易1, BTC交易2, BTC交易3]  → 分区1
  Key=ETH: [ETH交易1, ETH交易2]            → 分区2
```

## 关键要点

1. **key 的选择**：应该选择分布均匀的字段（避免数据倾斜）
2. **key 的类型**：必须是可序列化的
3. **状态隔离**：不同 key 的状态完全隔离
4. **并行度**：keyBy 会改变数据的分区，影响并行度

## 与 groupBy 的区别

| 操作 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `keyBy()` | DataStream | KeyedStream | 流式分组（用于窗口、状态） |
| SQL `GROUP BY` | 表 | 分组表 | 批处理分组 |

## 什么时候你需要想到这个？

- 当你需要**按某个字段分组处理数据**时（如按交易对分组）
- 当你需要使用**窗口操作**时（必须先keyBy）
- 当你需要使用**状态**时（必须先keyBy）
- 当你需要**为每个key独立处理**时（状态隔离）
- 当你需要**实现聚合操作**时（按key聚合）

