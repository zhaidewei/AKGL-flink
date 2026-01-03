# 状态的生命周期：创建、更新、清理

## 核心概念

**状态的生命周期**包括状态的**创建、更新、清理**三个阶段。理解生命周期有助于正确使用状态。

### 生命周期阶段

1. **创建**：在 `open()` 方法中创建 StateDescriptor 并获取状态
2. **更新**：在 `processElement()` 中读取和更新状态
3. **清理**：状态可以手动清理，或在key过期时自动清理

## 最小可用例子

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private ValueState<Double> lastPriceState;

    // 阶段1：创建（在open中）
    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);
        lastPriceState = getRuntimeContext().getState(descriptor);
        // 状态已创建，但值为null
    }

    // 阶段2：更新（在processElement中）
    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 读取状态（可能为null）
        Double lastPrice = lastPriceState.value();

        if (lastPrice != null) {
            out.collect("Price changed: " + (trade.getPrice() - lastPrice));
        }

        // 更新状态
        lastPriceState.update(trade.getPrice());
    }

    // 阶段3：清理（可选，在onTimer或processElement中）
    // lastPriceState.clear();  // 手动清理
}
```

## 生命周期详解

### 1. 创建阶段（open方法）

```java
@Override
public void open(Configuration parameters) {
    // 创建StateDescriptor
    ValueStateDescriptor<Double> descriptor =
        new ValueStateDescriptor<>("lastPrice", Double.class);

    // 获取状态对象（状态已创建，但值为null）
    lastPriceState = getRuntimeContext().getState(descriptor);

    // 此时状态存在，但value()返回null
}
```

### 2. 更新阶段（processElement方法）

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<String> out) {
    // 第一次调用：value()返回null（状态未初始化）
    // 后续调用：value()返回上次更新的值

    Double lastPrice = lastPriceState.value();  // 读取

    // 处理逻辑...

    lastPriceState.update(trade.getPrice());  // 更新
}
```

### 3. 清理阶段（可选）

```java
// 方式1：手动清理
lastPriceState.clear();

// 方式2：通过TTL自动清理（如果设置了TTL）
// 状态会在TTL到期后自动清理
```

## 币安交易数据示例

### 完整生命周期

```java
public class TradeTracker extends KeyedProcessFunction<String, Trade, TradeSummary> {
    private ValueState<Double> lastPriceState;
    private ValueState<Integer> tradeCountState;

    // 创建：在open中初始化状态
    @Override
    public void open(Configuration parameters) {
        lastPriceState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("lastPrice", Double.class));
        tradeCountState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("tradeCount", Integer.class));
    }

    // 更新：在processElement中更新状态
    @Override
    public void processElement(Trade trade, Context ctx, Collector<TradeSummary> out) {
        // 读取状态
        Double lastPrice = lastPriceState.value();
        Integer count = tradeCountState.value();
        if (count == null) count = 0;

        // 处理逻辑
        TradeSummary summary = new TradeSummary();
        summary.setSymbol(trade.getSymbol());
        summary.setCurrentPrice(trade.getPrice());
        if (lastPrice != null) {
            summary.setPriceChange(trade.getPrice() - lastPrice);
        }
        summary.setTradeCount(count + 1);

        // 更新状态
        lastPriceState.update(trade.getPrice());
        tradeCountState.update(count + 1);

        out.collect(summary);
    }
}
```

## 状态初始化

```java
// 方式1：检查null并使用默认值
Double value = state.value();
if (value == null) {
    value = 0.0;  // 默认值
}

// 方式2：直接使用，让Flink处理null
Double value = state.value();
// 需要处理null的情况
```

## 什么时候你需要想到这个？

- 当你需要**理解状态的使用流程**时（创建→更新→清理）
- 当你需要**初始化状态**时（在open中创建）
- 当你需要**更新状态**时（在processElement中）
- 当你需要**清理状态**时（手动清理或TTL）
- 当你需要**调试状态问题**时（检查生命周期）


