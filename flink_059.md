# 获取State：通过RuntimeContext获取状态

## 核心概念

在 ProcessFunction 中，通过 **RuntimeContext** 的 `getState()` 方法获取状态对象。这是访问状态的唯一方式。

### 核心流程

1. **创建 StateDescriptor**：声明状态的名称和类型
2. **获取 RuntimeContext**：通过 `getRuntimeContext()` 获取
3. **获取状态对象**：通过 `getState()` 等方法获取

## 最小可用例子

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        // 1. 创建StateDescriptor
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);

        // 2. 通过RuntimeContext获取状态
        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 3. 使用状态
        Double lastPrice = lastPriceState.value();
        lastPriceState.update(trade.getPrice());
    }
}
```

## 获取不同类型的状态

### ValueState

```java
ValueStateDescriptor<Double> descriptor =
    new ValueStateDescriptor<>("lastPrice", Double.class);

ValueState<Double> state = getRuntimeContext().getState(descriptor);
```

### MapState

```java
MapStateDescriptor<String, Double> descriptor =
    new MapStateDescriptor<>("priceMap", String.class, Double.class);

MapState<String, Double> state = getRuntimeContext().getMapState(descriptor);
```

### ListState

```java
ListStateDescriptor<Trade> descriptor =
    new ListStateDescriptor<>("tradeList", Trade.class);

ListState<Trade> state = getRuntimeContext().getListState(descriptor);
```

## 币安交易数据示例

### 跟踪最新价格

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, PriceChange> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        // 获取ValueState
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);
        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<PriceChange> out) {
        Double lastPrice = lastPriceState.value();

        if (lastPrice != null) {
            PriceChange change = new PriceChange();
            change.setChange(trade.getPrice() - lastPrice);
            out.collect(change);
        }

        lastPriceState.update(trade.getPrice());
    }
}
```

## 关键要点

1. **必须在 open() 中获取**：状态获取应该在 open() 方法中完成
2. **使用 RuntimeContext**：通过 `getRuntimeContext()` 获取
3. **需要 StateDescriptor**：必须先创建描述符
4. **状态按key隔离**：每个key有独立的状态

## 什么时候你需要想到这个？

- 当你需要**获取状态对象**时（通过RuntimeContext）
- 当你需要**初始化状态**时（在open方法中）
- 当你需要**理解状态访问机制**时（如何获取状态）
- 当你需要**使用不同类型的状态**时（getState、getMapState等）
- 当你需要**理解RuntimeContext**时（状态访问的入口）


