# ValueState.update()：更新状态值

## 核心概念

**`update()`** 是 ValueState 的方法，用于**更新状态中存储的值**。

### 方法签名

```java
void update(T value);
```

### 核心特点

1. **更新值**：将新值写入状态
2. **覆盖旧值**：新值会覆盖旧值
3. **立即生效**：更新后立即生效

## 最小可用例子

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private ValueState<Double> lastPriceState;

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 读取状态
        Double lastPrice = lastPriceState.value();

        if (lastPrice != null) {
            double change = trade.getPrice() - lastPrice;
            out.collect("Price changed: " + change);
        }

        // 更新状态：保存当前价格
        lastPriceState.update(trade.getPrice());
    }
}
```

## 币安交易数据示例

### 更新最新价格

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<PriceChange> out) {
    Double lastPrice = lastPriceState.value();

    if (lastPrice != null) {
        PriceChange change = new PriceChange();
        change.setChange(trade.getPrice() - lastPrice);
        out.collect(change);
    }

    // 更新状态：保存当前交易的价格
    lastPriceState.update(trade.getPrice());
}
```

### 累计更新

```java
public class VolumeTracker extends KeyedProcessFunction<String, Trade, Double> {
    private ValueState<Double> totalVolumeState;

    @Override
    public void processElement(Trade trade, Context ctx, Collector<Double> out) {
        // 读取当前累计量
        Double totalVolume = totalVolumeState.value();
        if (totalVolume == null) {
            totalVolume = 0.0;
        }

        // 累加
        totalVolume += trade.getQuantity();

        // 更新状态：保存新的累计量
        totalVolumeState.update(totalVolume);

        out.collect(totalVolume);
    }
}
```

## 更新模式

### 直接更新

```java
// 直接用新值更新
lastPriceState.update(trade.getPrice());
```

### 累加更新

```java
// 读取 → 计算 → 更新
Double current = state.value();
if (current == null) current = 0.0;
current += increment;
state.update(current);
```

## 什么时候你需要想到这个？

- 当你需要**更新状态值**时（update()方法）
- 当你需要**保存最新值**时（如最新价格）
- 当你需要**累计更新**时（如累计交易量）
- 当你需要理解**状态的基本操作**时（读取和更新）
- 当你需要**维护状态**时（持续更新状态）


