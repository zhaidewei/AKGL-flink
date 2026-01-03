# flatMap()：一对多元素转换

## 核心概念

**`flatMap()`** 将一个元素转换为**零个或多个**元素。与 `map()` 的一对一不同，`flatMap()` 可以产生多个输出或没有输出。

### 类比理解

这就像：
- **字符串分割**：`"hello world"` → `["hello", "world"]` - 一个输入，多个输出
- **数组展开**：`[[1,2], [3,4]]` → `[1,2,3,4]` - 展开嵌套数组
- **Spark 的 flatMap()**：如果你用过 Spark，Flink 的 flatMap() 完全一样

### 核心特点

1. **一对多或零**：一个输入可以产生零个、一个或多个输出
2. **使用 Collector**：通过 Collector 输出多个元素
3. **可以过滤**：不调用 `collect()` 就相当于过滤

## 源码位置

flatMap() 方法在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

FlatMapFunction 接口在：
[flink-core/src/main/java/org/apache/flink/api/common/functions/FlatMapFunction.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/functions/FlatMapFunction.java)

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

## 最小可用例子

### 方式1：Lambda 表达式

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> lines = env.fromElements("hello world", "flink streaming");

// 将每行按空格分割成单词
DataStream<String> words = lines.flatMap((String line, Collector<String> out) -> {
    for (String word : line.split(" ")) {
        out.collect(word);
    }
});

words.print();  // 输出: hello, world, flink, streaming
env.execute();
```

### 方式2：实现 FlatMapFunction 接口

```java
public class WordSplitter implements FlatMapFunction<String, String> {
    @Override
    public void flatMap(String line, Collector<String> out) throws Exception {
        for (String word : line.split(" ")) {
            out.collect(word);
        }
    }
}

// 使用
DataStream<String> words = lines.flatMap(new WordSplitter());
```

## 常见使用场景

### 1. 字符串分割

```java
// 将句子分割成单词
DataStream<String> sentences = env.fromElements("hello world", "flink is great");
DataStream<String> words = sentences.flatMap((s, out) -> {
    for (String word : s.split(" ")) {
        out.collect(word);
    }
});
```

### 2. 展开嵌套结构

```java
// 将交易列表展开
public class TradeListFlatMap implements FlatMapFunction<List<Trade>, Trade> {
    @Override
    public void flatMap(List<Trade> trades, Collector<Trade> out) throws Exception {
        for (Trade trade : trades) {
            out.collect(trade);
        }
    }
}
```

### 3. 条件输出（可以过滤）

```java
// 只输出满足条件的元素
DataStream<Trade> trades = env.addSource(new BinanceSource());
DataStream<Trade> largeTrades = trades.flatMap((trade, out) -> {
    if (trade.getPrice() * trade.getQuantity() > 10000) {
        out.collect(trade);  // 只输出大额交易
    }
    // 不调用collect()就相当于过滤掉
});
```

### 4. 一对多转换

```java
// 一个交易产生多个事件
DataStream<Trade> trades = env.addSource(new BinanceSource());
DataStream<TradeEvent> events = trades.flatMap((trade, out) -> {
    // 产生价格事件
    out.collect(new PriceEvent(trade.getSymbol(), trade.getPrice()));

    // 产生数量事件
    out.collect(new QuantityEvent(trade.getSymbol(), trade.getQuantity()));

    // 产生金额事件
    out.collect(new AmountEvent(trade.getSymbol(),
        trade.getPrice() * trade.getQuantity()));
});
```

## 与 map() 的区别

| 操作 | 输入输出关系 | 使用方式 | 用途 |
|------|-------------|---------|------|
| `map()` | 一对一 | 返回一个值 | 转换元素 |
| `flatMap()` | 一对多或零 | 使用 Collector | 展开、分割、条件输出 |

```java
// map(): 一个输入，一个输出
words.map(s -> s.toUpperCase());  // "hello" → "HELLO"

// flatMap(): 一个输入，多个输出
lines.flatMap((s, out) -> {
    for (String word : s.split(" ")) {
        out.collect(word);
    }
});  // "hello world" → "hello", "world"
```

## 币安交易数据示例

### 分割交易对

```java
// 假设一个消息包含多个交易对的数据
DataStream<MultiTradeMessage> messages = env.addSource(new BinanceSource());

// 展开为单个交易
DataStream<Trade> trades = messages.flatMap((message, out) -> {
    for (Trade trade : message.getTrades()) {
        out.collect(trade);
    }
});
```

### 生成多个事件

```java
// 一个交易产生多个分析事件
DataStream<Trade> trades = env.addSource(new BinanceSource());
DataStream<AnalysisEvent> events = trades.flatMap((trade, out) -> {
    // 价格变化事件
    out.collect(new PriceChangeEvent(trade));

    // 如果是大额交易，产生额外事件
    if (trade.getPrice() * trade.getQuantity() > 10000) {
        out.collect(new LargeTradeEvent(trade));
    }
});
```

## 什么时候你需要想到这个？

- 当你需要**将一个元素展开为多个元素**时（分割、展开）
- 当你需要**实现一对多转换**时（一个输入，多个输出）
- 当你需要**同时实现转换和过滤**时（flatMap可以过滤）
- 当你需要**处理嵌套结构**时（展开列表、数组）
- 当你需要**生成多个事件**时（一个数据产生多个事件）

