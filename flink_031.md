# KeySelector：定义key提取逻辑

## 核心概念

**KeySelector** 是 Flink 中用于从数据中提取 key 的接口。在 `keyBy()` 操作中，你可以通过实现 KeySelector 来定义如何从数据中提取分组 key。

### 接口定义

```java
public interface KeySelector<IN, KEY> extends Function, Serializable {
    /**
     * 从输入数据中提取key
     * @param value 输入值
     * @return 提取的key
     */
    KEY getKey(IN value) throws Exception;
}
```

### 核心特点

1. **函数式接口**：可以用 Lambda 表达式
2. **泛型**：`<IN, KEY>` - IN是输入类型，KEY是key类型
3. **可序列化**：因为要在集群中传输

## 源码位置

KeySelector 接口定义在：
[flink-core-api/src/main/java/org/apache/flink/api/java/functions/KeySelector.java](https://github.com/apache/flink/blob/master/flink-core-api/src/main/java/org/apache/flink/api/java/functions/KeySelector.java)

## 实现方式

### 方式1：Lambda 表达式（推荐）

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 直接使用Lambda
KeyedStream<Trade, String> keyed = trades.keyBy(trade -> trade.getSymbol());
```

### 方式2：实现 KeySelector 接口

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

### 按交易对提取key

```java
public class SymbolKeySelector implements KeySelector<Trade, String> {
    @Override
    public String getKey(Trade trade) throws Exception {
        return trade.getSymbol();  // "BTCUSDT", "ETHUSDT"等
    }
}

KeyedStream<Trade, String> keyed = trades.keyBy(new SymbolKeySelector());
```

### 按价格区间提取key

```java
public class PriceRangeKeySelector implements KeySelector<Trade, String> {
    @Override
    public String getKey(Trade trade) throws Exception {
        double price = trade.getPrice();
        if (price > 50000) return "HIGH";
        else if (price > 30000) return "MEDIUM";
        else return "LOW";
    }
}

KeyedStream<Trade, String> priceGroups = trades.keyBy(new PriceRangeKeySelector());
```

### 组合key

```java
public class CompositeKeySelector implements KeySelector<Trade, String> {
    @Override
    public String getKey(Trade trade) throws Exception {
        // 组合交易对和价格区间
        String priceRange = trade.getPrice() > 50000 ? "HIGH" : "LOW";
        return trade.getSymbol() + "_" + priceRange;
    }
}

KeyedStream<Trade, String> composite = trades.keyBy(new CompositeKeySelector());
```

### 使用Lambda（推荐）

```java
// 简单key提取
trades.keyBy(trade -> trade.getSymbol());

// 复杂key提取
trades.keyBy(trade -> {
    String symbol = trade.getSymbol();
    String range = trade.getPrice() > 50000 ? "HIGH" : "LOW";
    return symbol + "_" + range;
});
```

## 关键要点

1. **key 的选择**：应该选择分布均匀的字段（避免数据倾斜）
2. **key 的类型**：必须是可序列化的
3. **性能考虑**：key 提取应该快速（会被频繁调用）
4. **null key**：避免返回 null（可能导致问题）

## 什么时候你需要想到这个？

- 当你需要**自定义 key 提取逻辑**时（复杂key计算）
- 当你需要**复用 key 提取逻辑**时（定义可重用的KeySelector）
- 当你需要**理解 keyBy() 的工作原理**时（key如何提取）
- 当你需要**优化分组性能**时（key提取的性能影响）
- 当你需要**实现复杂分组**时（组合key、条件key等）

