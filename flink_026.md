# filter()：根据条件过滤数据

## 核心概念

**`filter()`** 用于根据条件过滤数据流中的元素，只保留满足条件的元素。

### 类比理解

这就像：
- **SQL 的 WHERE**：`SELECT * FROM trades WHERE price > 50000`
- **数组过滤**：只保留满足条件的元素
- **Spark 的 filter()**：如果你用过 Spark，Flink 的 filter() 完全一样

### 核心特点

1. **一对零或一**：每个输入元素可能被保留（输出一个）或丢弃（输出零个）
2. **类型不变**：`DataStream<T>` → `DataStream<T>`
3. **返回布尔值**：`true` 保留，`false` 丢弃

## 源码位置

filter() 方法在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

FilterFunction 接口在：
[flink-core/src/main/java/org/apache/flink/api/common/functions/FilterFunction.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/functions/FilterFunction.java)

### 接口定义

```java
@FunctionalInterface
public interface FilterFunction<T> extends Function, Serializable {
    /**
     * 判断元素是否应该保留
     * @param value 输入值
     * @return true保留，false丢弃
     */
    boolean filter(T value) throws Exception;
}
```

## 最小可用例子

### 方式1：Lambda 表达式（推荐）

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> words = env.fromElements("hello", "world", "flink", "hi");

// 只保留长度大于4的单词
DataStream<String> longWords = words.filter(s -> s.length() > 4);

longWords.print();  // 输出: hello, world, flink（"hi"被过滤掉）
env.execute();
```

### 方式2：实现 FilterFunction 接口

```java
public class LongWordFilter implements FilterFunction<String> {
    @Override
    public boolean filter(String value) throws Exception {
        return value.length() > 4;
    }
}

// 使用
DataStream<String> longWords = words.filter(new LongWordFilter());
```

### 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 只保留BTC交易
DataStream<Trade> btcTrades = trades.filter(trade -> trade.getSymbol().equals("BTCUSDT"));

// 只保留价格大于50000的交易
DataStream<Trade> highPriceTrades = trades.filter(trade -> trade.getPrice() > 50000);

// 只保留大额交易（金额>10000）
DataStream<Trade> largeTrades = trades.filter(trade ->
    trade.getPrice() * trade.getQuantity() > 10000
);

// 组合条件：BTC且价格>50000
DataStream<Trade> filtered = trades.filter(trade ->
    trade.getSymbol().equals("BTCUSDT") && trade.getPrice() > 50000
);
```

## 常见过滤场景

### 1. 按字段值过滤

```java
// 只保留特定交易对
trades.filter(trade -> trade.getSymbol().equals("BTCUSDT"));

// 只保留买方主动的交易
trades.filter(trade -> trade.isBuyerMaker());
```

### 2. 按数值范围过滤

```java
// 价格范围
trades.filter(trade -> trade.getPrice() >= 50000 && trade.getPrice() <= 60000);

// 交易量阈值
trades.filter(trade -> trade.getQuantity() > 1.0);
```

### 3. 组合条件过滤

```java
trades.filter(trade ->
    trade.getSymbol().equals("BTCUSDT") &&
    trade.getPrice() > 50000 &&
    trade.getQuantity() > 0.1
);
```

## 与 map() 的区别

| 操作 | 输入输出关系 | 返回类型 | 用途 |
|------|-------------|---------|------|
| `map()` | 一对一 | 任意类型 | 转换元素 |
| `filter()` | 一对零或一 | boolean | 过滤元素 |

```java
// map(): 每个元素都转换
words.map(s -> s.toUpperCase());  // "hello" → "HELLO"

// filter(): 只保留满足条件的元素
words.filter(s -> s.length() > 4);  // "hello" 保留，"hi" 丢弃
```

## 性能考虑

1. **尽早过滤**：在数据流早期过滤，减少后续处理的数据量
2. **简单条件**：过滤条件应该简单快速
3. **避免复杂计算**：不要在 filter() 中做复杂计算

```java
// 好的做法：先过滤，再处理
trades.filter(trade -> trade.getPrice() > 50000)
      .map(trade -> complexProcessing(trade));

// 不好的做法：先处理，再过滤
trades.map(trade -> complexProcessing(trade))
      .filter(processed -> processed.getPrice() > 50000);
```

## 什么时候你需要想到这个？

- 当你需要**过滤数据流中的元素**时（只保留满足条件的）
- 当你需要**筛选特定数据**时（如只处理BTC交易）
- 当你需要**减少数据量**时（过滤掉不需要的数据）
- 当你需要**实现条件处理**时（WHERE子句的流式版本）
- 当你需要**优化性能**时（尽早过滤减少处理量）

