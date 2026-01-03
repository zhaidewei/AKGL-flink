# ValueState.value()：读取状态值

## 核心概念

**`value()`** 是 ValueState 的方法，用于**读取状态中存储的值**。

### 方法签名

```java
T value();
```

### 核心特点

1. **读取值**：返回状态中存储的值
2. **可能为null**：如果状态未初始化，返回null
3. **类型安全**：返回类型与StateDescriptor中声明的类型一致

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
        // 读取状态值
        Double lastPrice = lastPriceState.value();

        if (lastPrice != null) {
            double change = trade.getPrice() - lastPrice;
            out.collect("Price changed: " + change);
        }

        lastPriceState.update(trade.getPrice());
    }
}
```

## 币安交易数据示例

### 读取最新价格

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<PriceChange> out) {
    // 读取状态：获取上次的价格
    Double lastPrice = lastPriceState.value();

    if (lastPrice != null) {
        // 计算价格变化
        PriceChange change = new PriceChange();
        change.setSymbol(trade.getSymbol());
        change.setLastPrice(lastPrice);
        change.setCurrentPrice(trade.getPrice());
        change.setChange(trade.getPrice() - lastPrice);
        change.setChangePercent((trade.getPrice() - lastPrice) / lastPrice * 100);
        out.collect(change);
    }

    // 更新状态
    lastPriceState.update(trade.getPrice());
}
```

## 处理null值

```java
// 方式1：检查null
Double lastPrice = lastPriceState.value();
if (lastPrice != null) {
    // 使用lastPrice
}

// 方式2：使用默认值
Double lastPrice = lastPriceState.value();
if (lastPrice == null) {
    lastPrice = 0.0;  // 默认值
}

// 方式3：使用Optional（Java 8+）
Double lastPrice = Optional.ofNullable(lastPriceState.value())
                          .orElse(0.0);
```

## 什么时候你需要想到这个？

- 当你需要**读取状态值**时（value()方法）
- 当你需要**处理空状态**时（value()可能返回null）
- 当你需要**使用状态进行计算**时（读取后使用）
- 当你需要理解**状态的基本操作**时（读取和更新）
- 当你需要**调试状态问题**时（检查状态值）


