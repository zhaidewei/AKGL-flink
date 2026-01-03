# MapFunction接口：实现map转换逻辑

## 核心概念

**MapFunction** 是 Flink 中用于实现 `map()` 转换的接口。你可以通过实现这个接口来定义转换逻辑。

### 接口定义

```java
@FunctionalInterface
public interface MapFunction<T, O> extends Function, Serializable {
    O map(T value) throws Exception;
}
```

### 关键特点

1. **函数式接口**：可以用 Lambda 表达式
2. **泛型**：`<T, O>` - T是输入类型，O是输出类型
3. **可序列化**：因为要在集群中传输

## 源码位置

MapFunction 接口定义在：
[flink-core/src/main/java/org/apache/flink/api/common/functions/MapFunction.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/functions/MapFunction.java)

## 实现方式

### 方式1：Lambda 表达式（最简单）

```java
DataStream<String> words = env.fromElements("hello", "world");
DataStream<String> upper = words.map(s -> s.toUpperCase());
```

### 方式2：匿名内部类

```java
DataStream<String> words = env.fromElements("hello", "world");
DataStream<String> upper = words.map(new MapFunction<String, String>() {
    @Override
    public String map(String value) throws Exception {
        return value.toUpperCase();
    }
});
```

### 方式3：实现类（复杂逻辑）

```java
public class PriceExtractor implements MapFunction<Trade, Double> {
    @Override
    public Double map(Trade trade) throws Exception {
        return trade.getPrice();
    }
}

// 使用
DataStream<Trade> trades = env.addSource(new BinanceSource());
DataStream<Double> prices = trades.map(new PriceExtractor());
```

## 币安交易数据示例

### 提取价格

```java
public class PriceMapFunction implements MapFunction<Trade, Double> {
    @Override
    public Double map(Trade trade) throws Exception {
        return trade.getPrice();
    }
}

DataStream<Trade> trades = env.addSource(new BinanceSource());
DataStream<Double> prices = trades.map(new PriceMapFunction());
```

### 计算交易金额

```java
public class AmountMapFunction implements MapFunction<Trade, Double> {
    @Override
    public Double map(Trade trade) throws Exception {
        return trade.getPrice() * trade.getQuantity();
    }
}

DataStream<Double> amounts = trades.map(new AmountMapFunction());
```

### 转换为简化对象

```java
public class TradeSummary {
    private String symbol;
    private double price;
    private long timestamp;
    // getter/setter...
}

public class TradeSummaryMapFunction implements MapFunction<Trade, TradeSummary> {
    @Override
    public TradeSummary map(Trade trade) throws Exception {
        TradeSummary summary = new TradeSummary();
        summary.setSymbol(trade.getSymbol());
        summary.setPrice(trade.getPrice());
        summary.setTimestamp(trade.getTradeTime());
        return summary;
    }
}

DataStream<TradeSummary> summaries = trades.map(new TradeSummaryMapFunction());
```

## 使用 Lambda（推荐）

对于简单逻辑，使用 Lambda 更简洁：

```java
// 提取价格
DataStream<Double> prices = trades.map(trade -> trade.getPrice());

// 计算金额
DataStream<Double> amounts = trades.map(trade -> trade.getPrice() * trade.getQuantity());

// 转换为字符串
DataStream<String> symbols = trades.map(trade -> trade.getSymbol());
```

## 使用实现类（复杂逻辑）

对于复杂逻辑，使用实现类更清晰：

```java
public class ComplexMapFunction implements MapFunction<Trade, ProcessedTrade> {
    @Override
    public ProcessedTrade map(Trade trade) throws Exception {
        ProcessedTrade processed = new ProcessedTrade();

        // 复杂处理逻辑
        processed.setSymbol(trade.getSymbol());
        processed.setPrice(trade.getPrice());
        processed.setAmount(trade.getPrice() * trade.getQuantity());
        processed.setTimestamp(trade.getTradeTime());

        // 业务逻辑
        if (trade.getPrice() > 50000) {
            processed.setCategory("HIGH");
        } else {
            processed.setCategory("NORMAL");
        }

        return processed;
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**实现 map() 转换逻辑**时（定义如何转换元素）
- 当你需要**处理复杂转换**时（使用实现类）
- 当你需要**理解 Flink 的函数接口**时（MapFunction、FilterFunction等）
- 当你需要**复用转换逻辑**时（定义可重用的MapFunction）
- 当你需要**调试转换逻辑**时（在map方法中设置断点）

