# KeyedStream的状态隔离：每个key独立状态

## 核心概念

在 **KeyedStream** 中，每个 key 的状态是**完全隔离**的。不同 key 的数据不会共享状态，每个 key 都有自己独立的状态空间。

### 类比理解

这就像：
- **多个独立的HashMap**：每个key有自己的HashMap
- **数据库分区**：每个分区有独立的表
- **命名空间**：每个key有自己的命名空间

### 核心特点

1. **状态隔离**：不同 key 的状态互不影响
2. **独立更新**：每个 key 的状态独立更新
3. **并行处理**：不同 key 可以在不同并行任务中处理

## 最小可用例子

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);
        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 获取当前key的状态（只影响当前key）
        Double lastPrice = lastPriceState.value();

        if (lastPrice != null) {
            double change = trade.getPrice() - lastPrice;
            out.collect(trade.getSymbol() + " price changed: " + change);
        }

        // 更新当前key的状态（只影响当前key）
        lastPriceState.update(trade.getPrice());
    }
}

// 使用
DataStream<Trade> trades = env.addSource(new BinanceSource());
KeyedStream<Trade, String> keyed = trades.keyBy(trade -> trade.getSymbol());

// 每个交易对（BTC、ETH等）有独立的状态
keyed.process(new PriceTracker()).print();
```

## 状态隔离示例

### 场景：跟踪每个交易对的最新价格

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());
KeyedStream<Trade, String> keyed = trades.keyBy(trade -> trade.getSymbol());

keyed.process(new KeyedProcessFunction<String, Trade, String>() {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        lastPriceState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("lastPrice", Double.class));
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // BTC的状态和ETH的状态是完全独立的
        // BTC的价格变化不会影响ETH的状态

        Double lastPrice = lastPriceState.value();  // 只获取当前key（如BTC）的状态

        if (lastPrice != null) {
            double change = trade.getPrice() - lastPrice;
            out.collect(trade.getSymbol() + ": " + change);
        }

        lastPriceState.update(trade.getPrice());  // 只更新当前key的状态
    }
});
```

### 数据流示例

```
输入数据:
  BTC交易1 (price=50000)  → Key=BTC → 更新BTC的状态: lastPrice=50000
  ETH交易1 (price=3000)   → Key=ETH → 更新ETH的状态: lastPrice=3000
  BTC交易2 (price=50100)  → Key=BTC → 更新BTC的状态: lastPrice=50100（不影响ETH）
  ETH交易2 (price=3010)   → Key=ETH → 更新ETH的状态: lastPrice=3010（不影响BTC）

状态存储:
  Key=BTC: lastPrice=50100
  Key=ETH: lastPrice=3010
  （完全独立）
```

## 关键要点

1. **状态按key隔离**：每个key有独立的状态空间
2. **不会相互影响**：BTC的状态变化不会影响ETH的状态
3. **并行处理**：不同key可以在不同并行任务中处理
4. **状态访问**：只能访问当前key的状态

## 与全局状态的区别

| 特性 | KeyedState（KeyedStream） | OperatorState（DataStream） |
|------|--------------------------|---------------------------|
| 作用域 | 每个key独立 | 整个算子共享 |
| 访问方式 | 通过key访问 | 全局访问 |
| 使用场景 | 按key聚合、窗口 | 全局配置、广播 |

## 什么时候你需要想到这个？

- 当你需要**为每个key维护独立状态**时（如每个交易对的最新价格）
- 当你需要理解 Flink 的**状态管理机制**时（状态如何隔离）
- 当你需要**实现按key聚合**时（每个key独立计算）
- 当你需要**调试状态问题**时（理解状态的作用域）
- 当你需要**优化状态使用**时（避免不必要的状态）

