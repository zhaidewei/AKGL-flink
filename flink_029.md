# FlatMapFunction接口：实现一对多转换

## 核心概念

**FlatMapFunction** 是 Flink 中用于实现 `flatMap()` 转换的接口。通过实现这个接口来定义如何将一个元素转换为零个或多个元素。

### 接口定义

```java
@FunctionalInterface
public interface FlatMapFunction<T, O> extends Function, Serializable {
    /**
     * 将一个元素转换为零个或多个元素
     * @param value 输入值
     * @param out Collector，用于输出结果
     */
    void flatMap(T value, Collector<O> out) throws Exception;
}
```

### 关键特点

1. **使用 Collector**：通过 `out.collect()` 输出元素
2. **可以输出多个**：调用多次 `collect()` 输出多个元素
3. **可以过滤**：不调用 `collect()` 就相当于过滤

## 源码位置

FlatMapFunction 接口定义在：
[flink-core/src/main/java/org/apache/flink/api/common/functions/FlatMapFunction.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/functions/FlatMapFunction.java)

## 实现方式

### 方式1：Lambda 表达式

```java
DataStream<String> lines = env.fromElements("hello world", "flink streaming");
DataStream<String> words = lines.flatMap((String line, Collector<String> out) -> {
    for (String word : line.split(" ")) {
        out.collect(word);
    }
});
```

### 方式2：实现类

```java
public class WordSplitter implements FlatMapFunction<String, String> {
    @Override
    public void flatMap(String line, Collector<String> out) throws Exception {
        String[] words = line.split(" ");
        for (String word : words) {
            out.collect(word);
        }
    }
}

// 使用
DataStream<String> words = lines.flatMap(new WordSplitter());
```

## 币安交易数据示例

### 展开批量交易

```java
public class BatchTradeFlatMap implements FlatMapFunction<BatchTradeMessage, Trade> {
    @Override
    public void flatMap(BatchTradeMessage batch, Collector<Trade> out) throws Exception {
        // 将批量消息展开为单个交易
        for (Trade trade : batch.getTrades()) {
            out.collect(trade);
        }
    }
}

// 使用
DataStream<BatchTradeMessage> batches = env.addSource(new BinanceBatchSource());
DataStream<Trade> trades = batches.flatMap(new BatchTradeFlatMap());
```

### 生成多个分析事件

```java
public class TradeAnalysisFlatMap implements FlatMapFunction<Trade, AnalysisEvent> {
    @Override
    public void flatMap(Trade trade, Collector<AnalysisEvent> out) throws Exception {
        // 价格事件
        out.collect(new PriceEvent(trade.getSymbol(), trade.getPrice()));

        // 数量事件
        out.collect(new QuantityEvent(trade.getSymbol(), trade.getQuantity()));

        // 如果是大额交易，产生额外事件
        double amount = trade.getPrice() * trade.getQuantity();
        if (amount > 10000) {
            out.collect(new LargeTradeEvent(trade.getSymbol(), amount));
        }
    }
}
```

### 条件展开

```java
public class ConditionalFlatMap implements FlatMapFunction<Trade, Trade> {
    private final double minAmount;

    public ConditionalFlatMap(double minAmount) {
        this.minAmount = minAmount;
    }

    @Override
    public void flatMap(Trade trade, Collector<Trade> out) throws Exception {
        double amount = trade.getPrice() * trade.getQuantity();

        // 只输出满足条件的交易
        if (amount >= minAmount) {
            out.collect(trade);
        }
        // 不调用collect()就相当于过滤掉
    }
}

// 使用：只输出金额>=10000的交易
DataStream<Trade> largeTrades = trades.flatMap(new ConditionalFlatMap(10000));
```

## Collector 的使用

### 基本用法

```java
@Override
public void flatMap(T value, Collector<O> out) throws Exception {
    // 输出一个元素
    out.collect(element1);

    // 输出多个元素
    out.collect(element2);
    out.collect(element3);

    // 不调用collect()就相当于过滤
}
```

### 条件输出

```java
@Override
public void flatMap(Trade trade, Collector<Trade> out) throws Exception {
    // 条件1：输出所有BTC交易
    if (trade.getSymbol().equals("BTCUSDT")) {
        out.collect(trade);
    }

    // 条件2：输出大额交易
    if (trade.getPrice() * trade.getQuantity() > 10000) {
        out.collect(trade);
    }
}
```

## 与 MapFunction 的区别

| 接口 | 输出方式 | 输出数量 | 用途 |
|------|---------|---------|------|
| `MapFunction` | 返回值 | 固定1个 | 一对一转换 |
| `FlatMapFunction` | Collector | 0个或多个 | 一对多转换、展开 |

```java
// MapFunction: 返回一个值
public String map(String value) {
    return value.toUpperCase();
}

// FlatMapFunction: 使用Collector输出
public void flatMap(String value, Collector<String> out) {
    for (String word : value.split(" ")) {
        out.collect(word);
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**实现 flatMap() 转换逻辑**时（定义如何展开元素）
- 当你需要**处理一对多转换**时（一个输入，多个输出）
- 当你需要**同时实现转换和过滤**时（flatMap可以过滤）
- 当你需要**展开嵌套结构**时（列表、数组等）
- 当你需要**生成多个事件**时（一个数据产生多个事件）

