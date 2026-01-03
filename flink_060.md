# ValueState：存储和更新单个值

## 核心概念

**ValueState** 用于存储**单个值**的状态。它是 Flink 中最简单的状态类型，适合存储单个标量值。

### 核心特点

1. **单个值**：只能存储一个值
2. **读写操作**：`value()` 读取，`update()` 更新
3. **按key隔离**：每个key有独立的值

## 源码位置

ValueState 接口在：
[flink-core-api/src/main/java/org/apache/flink/api/common/state/ValueState.java](https://github.com/apache/flink/blob/master/flink-core-api/src/main/java/org/apache/flink/api/common/state/ValueState.java)

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
        // 读取状态
        Double lastPrice = lastPriceState.value();

        if (lastPrice != null) {
            double change = trade.getPrice() - lastPrice;
            out.collect(trade.getSymbol() + " changed: " + change);
        }

        // 更新状态
        lastPriceState.update(trade.getPrice());
    }
}
```

## 币安交易数据示例

### 跟踪每个交易对的最新价格

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

trades.keyBy(trade -> trade.getSymbol())
      .process(new KeyedProcessFunction<String, Trade, PriceChange>() {
          private ValueState<Double> lastPriceState;

          @Override
          public void open(Configuration parameters) {
              ValueStateDescriptor<Double> descriptor =
                  new ValueStateDescriptor<>("lastPrice", Double.class);
              lastPriceState = getRuntimeContext().getState(descriptor);
          }

          @Override
          public void processElement(Trade trade, Context ctx, Collector<PriceChange> out) {
              Double lastPrice = lastPriceState.value();

              if (lastPrice != null) {
                  PriceChange change = new PriceChange();
                  change.setSymbol(trade.getSymbol());
                  change.setLastPrice(lastPrice);
                  change.setCurrentPrice(trade.getPrice());
                  change.setChange(trade.getPrice() - lastPrice);
                  out.collect(change);
              }

              lastPriceState.update(trade.getPrice());
          }
      })
      .print();
```

### 累计交易量

```java
public class VolumeTracker extends KeyedProcessFunction<String, Trade, Double> {
    private ValueState<Double> totalVolumeState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("totalVolume", Double.class);
        totalVolumeState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<Double> out) {
        // 读取当前累计量
        Double totalVolume = totalVolumeState.value();
        if (totalVolume == null) {
            totalVolume = 0.0;
        }

        // 累加
        totalVolume += trade.getQuantity();

        // 更新状态
        totalVolumeState.update(totalVolume);

        // 输出
        out.collect(totalVolume);
    }
}
```

## 基本操作

### 读取值

```java
Double value = lastPriceState.value();

// 如果状态为空，返回null
if (value == null) {
    // 处理空状态
}
```

### 更新值

```java
lastPriceState.update(trade.getPrice());
```

### 清空状态

```java
lastPriceState.clear();
```

## 状态隔离

```java
// 每个key有独立的状态
// Key=BTC: lastPrice = 50000
// Key=ETH: lastPrice = 3000
// （完全独立，互不影响）
```

## 什么时候你需要想到这个？

- 当你需要**存储单个值**时（最新价格、累计量等）
- 当你需要**跟踪状态变化**时（价格变化、数量变化）
- 当你需要**累计计算**时（累加交易量、累加金额）
- 当你需要理解**状态的基本使用**时（从ValueState开始）
- 当你需要**简单的状态管理**时（单个值最简单）


