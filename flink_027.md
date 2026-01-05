# FilterFunction接口：实现过滤逻辑

## 核心概念

**FilterFunction** 是 Flink 中用于实现 `filter()` 过滤的接口。通过实现这个接口来定义过滤条件。

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

### 关键特点

1. **函数式接口**：可以用 Lambda 表达式
2. **返回布尔值**：`true` 保留，`false` 丢弃
3. **类型不变**：输入和输出类型相同

## 源码位置

FilterFunction 接口定义在：
[flink-core/src/main/java/org/apache/flink/api/common/functions/FilterFunction.java](https://github.com/apache/flink/blob/master/flink-core/src/main/java/org/apache/flink/api/common/functions/FilterFunction.java)

## 实现方式

### 方式1：Lambda 表达式（最简单）

```java
DataStream<String> words = env.fromElements("hello", "world", "hi");
DataStream<String> longWords = words.filter(s -> s.length() > 4);
```

### 方式2：匿名内部类

```java
DataStream<String> words = env.fromElements("hello", "world", "hi");
DataStream<String> longWords = words.filter(new FilterFunction<String>() {
    @Override
    public boolean filter(String value) throws Exception {
        return value.length() > 4;
    }
});
```

### 方式3：实现类（复杂逻辑）

```java
public class HighPriceFilter implements FilterFunction<Trade> {
    private final double threshold;

    public HighPriceFilter(double threshold) {
        this.threshold = threshold;
    }

    @Override
    public boolean filter(Trade trade) throws Exception {
        return trade.getPrice() > threshold;
    }
}

// 使用
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);
DataStream<Trade> highPriceTrades = trades.filter(new HighPriceFilter(50000));
```

## 币安交易数据示例

### 按价格过滤

```java
public class PriceFilter implements FilterFunction<Trade> {
    private final double minPrice;
    private final double maxPrice;

    public PriceFilter(double minPrice, double maxPrice) {
        this.minPrice = minPrice;
        this.maxPrice = maxPrice;
    }

    @Override
    public boolean filter(Trade trade) throws Exception {
        double price = trade.getPrice();
        return price >= minPrice && price <= maxPrice;
    }
}

// 使用：只保留价格在50000-60000之间的交易
DataStream<Trade> filtered = trades.filter(new PriceFilter(50000, 60000));
```

### 按交易对过滤

```java
public class SymbolFilter implements FilterFunction<Trade> {
    private final Set<String> allowedSymbols;

    public SymbolFilter(String... symbols) {
        this.allowedSymbols = Set.of(symbols);
    }

    @Override
    public boolean filter(Trade trade) throws Exception {
        return allowedSymbols.contains(trade.getSymbol());
    }
}

// 使用：只保留BTC和ETH交易
DataStream<Trade> filtered = trades.filter(new SymbolFilter("BTCUSDT", "ETHUSDT"));
```

### 按交易金额过滤

```java
public class LargeTradeFilter implements FilterFunction<Trade> {
    private final double minAmount;

    public LargeTradeFilter(double minAmount) {
        this.minAmount = minAmount;
    }

    @Override
    public boolean filter(Trade trade) throws Exception {
        double amount = trade.getPrice() * trade.getQuantity();
        return amount >= minAmount;
    }
}

// 使用：只保留金额>=10000的交易
DataStream<Trade> largeTrades = trades.filter(new LargeTradeFilter(10000));
```

## 使用 Lambda（推荐）

对于简单条件，使用 Lambda 更简洁：

```java
// 价格过滤
trades.filter(trade -> trade.getPrice() > 50000);

// 交易对过滤
trades.filter(trade -> trade.getSymbol().equals("BTCUSDT"));

// 组合条件
trades.filter(trade ->
    trade.getSymbol().equals("BTCUSDT") &&
    trade.getPrice() > 50000 &&
    trade.getQuantity() > 0.1
);
```

## 使用实现类（复杂逻辑）

对于复杂逻辑或需要配置的过滤，使用实现类：

```java
public class ComplexFilter implements FilterFunction<Trade> {
    private final FilterConfig config;

    public ComplexFilter(FilterConfig config) {
        this.config = config;
    }

    @Override
    public boolean filter(Trade trade) throws Exception {
        // 复杂过滤逻辑
        if (!config.getAllowedSymbols().contains(trade.getSymbol())) {
            return false;
        }

        if (trade.getPrice() < config.getMinPrice()) {
            return false;
        }

        double amount = trade.getPrice() * trade.getQuantity();
        if (amount < config.getMinAmount()) {
            return false;
        }

        // 时间窗口过滤
        long currentTime = System.currentTimeMillis();
        if (trade.getTradeTime() < currentTime - config.getTimeWindow()) {
            return false;
        }

        return true;
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**实现 filter() 过滤逻辑**时（定义过滤条件）
- 当你需要**处理复杂过滤条件**时（使用实现类）
- 当你需要**复用过滤逻辑**时（定义可重用的FilterFunction）
- 当你需要**配置化过滤**时（通过构造函数传入参数）
- 当你需要**调试过滤逻辑**时（在filter方法中设置断点）

