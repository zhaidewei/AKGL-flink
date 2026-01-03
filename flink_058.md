# StateDescriptor：声明状态的元数据

## 核心概念

**StateDescriptor** 用于**声明状态的元数据**，包括状态的名称、类型等信息。在获取状态之前，必须先创建 StateDescriptor。

### 核心作用

1. **声明状态**：定义状态的名称和类型
2. **获取状态**：通过 StateDescriptor 获取状态对象
3. **状态管理**：Flink 根据描述符管理状态

## 源码位置

StateDescriptor 相关类在：
[flink-core-api/src/main/java/org/apache/flink/api/common/state/](https://github.com/apache/flink/blob/master/flink-core-api/src/main/java/org/apache/flink/api/common/state/)

## 最小可用例子

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        // 创建StateDescriptor：声明状态的名称和类型
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);

        // 通过StateDescriptor获取状态对象
        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        Double lastPrice = lastPriceState.value();
        lastPriceState.update(trade.getPrice());
    }
}
```

## 不同类型的状态描述符

### ValueStateDescriptor

```java
// 声明单个值的状态
ValueStateDescriptor<Double> descriptor =
    new ValueStateDescriptor<>("lastPrice", Double.class);

ValueState<Double> state = getRuntimeContext().getState(descriptor);
```

### MapStateDescriptor

```java
// 声明键值对状态
MapStateDescriptor<String, Double> descriptor =
    new MapStateDescriptor<>("priceMap", String.class, Double.class);

MapState<String, Double> state = getRuntimeContext().getMapState(descriptor);
```

### ListStateDescriptor

```java
// 声明列表状态
ListStateDescriptor<Trade> descriptor =
    new ListStateDescriptor<>("tradeList", Trade.class);

ListState<Trade> state = getRuntimeContext().getListState(descriptor);
```

## 币安交易数据示例

### 跟踪每个交易对的最新价格

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, PriceChange> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        // 声明状态：存储最新价格
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

### 存储多个交易对的价格

```java
public class MultiPriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private MapState<String, Double> priceMapState;

    @Override
    public void open(Configuration parameters) {
        // 声明MapState：存储多个交易对的价格
        MapStateDescriptor<String, Double> descriptor =
            new MapStateDescriptor<>("priceMap", String.class, Double.class);
        priceMapState = getRuntimeContext().getMapState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 存储每个交易对的价格
        priceMapState.put(trade.getSymbol(), trade.getPrice());
    }
}
```

## 状态名称

```java
// 状态名称必须唯一（在同一算子内）
ValueStateDescriptor<Double> descriptor1 =
    new ValueStateDescriptor<>("lastPrice", Double.class);

ValueStateDescriptor<Double> descriptor2 =
    new ValueStateDescriptor<>("maxPrice", Double.class);  // 不同名称
```

## 什么时候你需要想到这个？

- 当你需要**使用状态**时（必须先声明StateDescriptor）
- 当你需要**获取状态对象**时（通过StateDescriptor）
- 当你需要**理解状态管理**时（描述符的作用）
- 当你需要**声明不同类型的状态**时（ValueState、MapState等）
- 当你需要**初始化状态**时（在open方法中）


