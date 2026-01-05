# MapState：存储键值对集合

## 核心概念

**MapState** 用于存储**键值对集合**的状态。类似于 Java 的 Map，可以存储多个键值对。

### 核心特点

1. **键值对存储**：可以存储多个键值对
2. **按key访问**：通过key快速访问value
3. **动态大小**：可以动态添加、删除键值对

## 源码位置

MapState 接口在：
[flink-core-api/src/main/java/org/apache/flink/api/common/state/MapState.java](https://github.com/apache/flink/blob/master/flink-core-api/src/main/java/org/apache/flink/api/common/state/MapState.java)

## 最小可用例子

```java
public class PriceMapTracker extends KeyedProcessFunction<String, Trade, String> {
    private MapState<String, Double> priceMapState;

    @Override
    public void open(Configuration parameters) {
        MapStateDescriptor<String, Double> descriptor =
            new MapStateDescriptor<>("priceMap", String.class, Double.class);
        priceMapState = getRuntimeContext().getMapState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 存储每个交易对的价格
        priceMapState.put(trade.getSymbol(), trade.getPrice());

        // 读取特定交易对的价格
        Double btcPrice = priceMapState.get("BTCUSDT");
        if (btcPrice != null) {
            out.collect("BTC price: " + btcPrice);
        }
    }
}
```

## 币安交易数据示例

### 存储多个交易对的价格

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

trades.keyBy(trade -> "all")  // 所有交易对共享一个key
      .process(new KeyedProcessFunction<String, Trade, String>() {
          private MapState<String, Double> priceMapState;

          @Override
          public void open(Configuration parameters) {
              MapStateDescriptor<String, Double> descriptor =
                  new MapStateDescriptor<>("priceMap", String.class, Double.class);
              priceMapState = getRuntimeContext().getMapState(descriptor);
          }

          @Override
          public void processElement(Trade trade, Context ctx, Collector<String> out) {
              // 存储每个交易对的最新价格
              priceMapState.put(trade.getSymbol(), trade.getPrice());

              // 可以读取任意交易对的价格
              Double btcPrice = priceMapState.get("BTCUSDT");
              Double ethPrice = priceMapState.get("ETHUSDT");

              if (btcPrice != null && ethPrice != null) {
                  out.collect("BTC: " + btcPrice + ", ETH: " + ethPrice);
              }
          }
      })
      .print();
```

## 基本操作

### put() - 添加/更新

```java
// 添加或更新键值对
priceMapState.put("BTCUSDT", 50000.0);
priceMapState.put("ETHUSDT", 3000.0);
```

### get() - 获取值

```java
// 获取值
Double price = priceMapState.get("BTCUSDT");
if (price != null) {
    // 使用price
}
```

### remove() - 删除

```java
// 删除键值对
priceMapState.remove("BTCUSDT");
```

### contains() - 检查存在

```java
// 检查key是否存在
if (priceMapState.contains("BTCUSDT")) {
    Double price = priceMapState.get("BTCUSDT");
}
```

### iterate() - 遍历

```java
// 遍历所有键值对
for (Map.Entry<String, Double> entry : priceMapState.entries()) {
    String symbol = entry.getKey();
    Double price = entry.getValue();
    // 处理
}
```

## 与 ValueState 的区别

| 特性 | ValueState | MapState |
|------|-----------|----------|
| 存储内容 | 单个值 | 键值对集合 |
| 访问方式 | value() | get(key) |
| 适用场景 | 单个值 | 多个键值对 |

## 什么时候你需要想到这个？

- 当你需要**存储多个键值对**时（如多个交易对的价格）
- 当你需要**按key快速访问**时（Map的查找特性）
- 当你需要**动态管理多个值**时（添加、删除键值对）
- 当你需要理解**不同类型的状态**时（MapState vs ValueState）
- 当你需要**存储关联数据**时（key-value关系）


