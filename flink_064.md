# MapState的基本操作：get、put、remove

## 核心概念

MapState 提供了类似 Java Map 的基本操作：`get()`、`put()`、`remove()` 等，用于管理键值对集合。

### 核心操作

1. **put(key, value)** - 添加或更新键值对
2. **get(key)** - 获取值
3. **remove(key)** - 删除键值对
4. **contains(key)** - 检查key是否存在
5. **entries()** - 遍历所有键值对

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
        // put: 添加或更新
        priceMapState.put(trade.getSymbol(), trade.getPrice());

        // get: 获取值
        Double btcPrice = priceMapState.get("BTCUSDT");

        // contains: 检查存在
        if (priceMapState.contains("ETHUSDT")) {
            Double ethPrice = priceMapState.get("ETHUSDT");
        }

        // remove: 删除（如果需要）
        // priceMapState.remove("OLDUSDT");
    }
}
```

## 币安交易数据示例

### 管理多个交易对的价格

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<String> out) {
    String symbol = trade.getSymbol();
    Double price = trade.getPrice();

    // put: 存储每个交易对的最新价格
    priceMapState.put(symbol, price);

    // get: 读取特定交易对的价格
    Double btcPrice = priceMapState.get("BTCUSDT");
    Double ethPrice = priceMapState.get("ETHUSDT");

    // 计算价格比
    if (btcPrice != null && ethPrice != null) {
        double ratio = btcPrice / ethPrice;
        out.collect("BTC/ETH ratio: " + ratio);
    }
}
```

## 操作详解

### put() - 添加或更新

```java
// 添加新键值对
priceMapState.put("BTCUSDT", 50000.0);

// 更新已存在的key
priceMapState.put("BTCUSDT", 50100.0);  // 覆盖旧值
```

### get() - 获取值

```java
// 获取值（可能为null）
Double price = priceMapState.get("BTCUSDT");

if (price != null) {
    // 使用price
} else {
    // key不存在
}
```

### remove() - 删除

```java
// 删除键值对
priceMapState.remove("BTCUSDT");

// 删除后，get()返回null
Double price = priceMapState.get("BTCUSDT");  // null
```

### contains() - 检查存在

```java
// 检查key是否存在
if (priceMapState.contains("BTCUSDT")) {
    Double price = priceMapState.get("BTCUSDT");
}
```

### entries() - 遍历

```java
// 遍历所有键值对
for (Map.Entry<String, Double> entry : priceMapState.entries()) {
    String symbol = entry.getKey();
    Double price = entry.getValue();
    out.collect(symbol + ": " + price);
}
```

## 什么时候你需要想到这个？

- 当你需要**使用MapState**时（基本操作）
- 当你需要**添加、获取、删除键值对**时（put、get、remove）
- 当你需要**检查key是否存在**时（contains）
- 当你需要**遍历所有键值对**时（entries）
- 当你需要理解**MapState的使用**时（类似Java Map）


