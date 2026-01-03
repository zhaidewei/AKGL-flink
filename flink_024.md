# map()：一对一元素转换

## 核心概念

**`map()`** 是 DataStream 中最基本的转换操作，将流中的每个元素转换为另一个元素，**一对一**的关系。

### 类比理解

这就像：
- **数学函数**：`f(x) = x * 2` - 每个输入对应一个输出
- **Spark 的 map()**：如果你用过 Spark，Flink 的 map() 完全一样
- **数组映射**：将数组中的每个元素转换

### 核心特点

1. **一对一**：每个输入元素产生一个输出元素
2. **类型可以改变**：`DataStream<String>` → `DataStream<Integer>`
3. **不能过滤**：每个元素都会被处理（使用 filter() 过滤）

## 源码位置

map() 方法在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/datastream/DataStream.java)

MapFunction 接口在：
[flink-core/src/main/java/org/apache/flink/api/common/functions/MapFunction.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/functions/MapFunction.java)

### 接口定义

```java
@FunctionalInterface
public interface MapFunction<T, O> extends Function, Serializable {
    /**
     * 将输入元素转换为输出元素
     * @param value 输入值
     * @return 转换后的值
     */
    O map(T value) throws Exception;
}
```

## 最小可用例子

### 方式1：Lambda 表达式（推荐）

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> words = env.fromElements("hello", "world", "flink");

// 将每个字符串转换为大写
DataStream<String> upperWords = words.map(s -> s.toUpperCase());

upperWords.print();
env.execute();
```

### 方式2：实现 MapFunction 接口

```java
public class UpperCaseMap implements MapFunction<String, String> {
    @Override
    public String map(String value) throws Exception {
        return value.toUpperCase();
    }
}

// 使用
DataStream<String> upperWords = words.map(new UpperCaseMap());
```

### 币安交易数据示例

```java
// 从币安WebSocket获取交易数据
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 提取价格
DataStream<Double> prices = trades.map(trade -> trade.getPrice());

// 提取交易量
DataStream<Double> quantities = trades.map(trade -> trade.getQuantity());

// 计算交易金额（价格 * 数量）
DataStream<Double> amounts = trades.map(trade -> trade.getPrice() * trade.getQuantity());
```

## 类型转换示例

```java
// String → Integer
DataStream<String> numbers = env.fromElements("1", "2", "3");
DataStream<Integer> ints = numbers.map(s -> Integer.parseInt(s));

// Trade → String
DataStream<Trade> trades = env.addSource(new BinanceSource());
DataStream<String> symbols = trades.map(trade -> trade.getSymbol());

// Trade → PriceInfo（自定义类型）
DataStream<PriceInfo> priceInfos = trades.map(trade -> {
    PriceInfo info = new PriceInfo();
    info.setSymbol(trade.getSymbol());
    info.setPrice(trade.getPrice());
    return info;
});
```

## 与 filter() 的区别

| 操作 | 输入输出关系 | 用途 |
|------|-------------|------|
| `map()` | 一对一 | 转换元素 |
| `filter()` | 一对零或一 | 过滤元素 |

```java
// map(): 每个元素都转换
words.map(s -> s.toUpperCase());  // "hello" → "HELLO"

// filter(): 只保留满足条件的元素
words.filter(s -> s.length() > 4);  // "hello" 保留，"hi" 丢弃
```

## 什么时候你需要想到这个？

- 当你需要**转换数据流中的元素**时（最常用的操作）
- 当你需要**提取字段**时（从Trade中提取price）
- 当你需要**类型转换**时（String → Integer）
- 当你需要**计算派生值**时（price * quantity）
- 当你学习 Flink 的**基本转换操作**时（从map开始）

