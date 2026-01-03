# 状态TTL：设置状态的过期时间

## 核心概念

**TTL（Time To Live）** 用于设置状态的**过期时间**。状态在指定时间后会自动过期并被清理，避免状态无限增长。

### 核心特点

1. **自动清理**：状态在TTL到期后自动清理
2. **节省内存**：避免状态无限增长
3. **按key管理**：每个key的状态独立计算TTL

## 最小可用例子

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, String> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);

        // 设置TTL：状态在10分钟后过期
        StateTtlConfig ttlConfig = StateTtlConfig
            .newBuilder(Time.minutes(10))
            .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
            .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
            .build();

        descriptor.enableTimeToLive(ttlConfig);

        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        Double lastPrice = lastPriceState.value();
        lastPriceState.update(trade.getPrice());
    }
}
```

## 币安交易数据示例

### 设置价格状态的TTL

```java
public class PriceTracker extends KeyedProcessFunction<String, Trade, PriceChange> {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);

        // TTL配置：状态在5分钟后过期
        // 如果某个交易对5分钟内没有新交易，状态会被清理
        StateTtlConfig ttlConfig = StateTtlConfig
            .newBuilder(Time.minutes(5))
            .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
            .build();

        descriptor.enableTimeToLive(ttlConfig);
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

        // 更新状态会重置TTL
        lastPriceState.update(trade.getPrice());
    }
}
```

## TTL配置选项

### UpdateType - 更新时机

```java
// OnCreateAndWrite: 创建和写入时更新TTL
.setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)

// OnReadAndWrite: 读取和写入时更新TTL
.setUpdateType(StateTtlConfig.UpdateType.OnReadAndWrite)
```

### StateVisibility - 过期状态可见性

```java
// NeverReturnExpired: 过期状态不返回（返回null）
.setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)

// ReturnExpiredIfNotCleanedUp: 如果未清理，返回过期状态
.setStateVisibility(StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp)
```

## 完整配置示例

```java
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.minutes(10))  // TTL时间：10分钟
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)  // 创建和写入时更新
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)  // 过期不返回
    .cleanupFullSnapshot()  // 在完整快照时清理
    .build();
```

## 使用场景

### 场景1：清理不活跃的交易对

```java
// 如果某个交易对10分钟内没有新交易，清理其状态
StateTtlConfig.newBuilder(Time.minutes(10))
```

### 场景2：限制状态大小

```java
// 通过TTL自动清理旧状态，避免内存无限增长
StateTtlConfig.newBuilder(Time.hours(1))
```

## 什么时候你需要想到这个？

- 当你需要**自动清理状态**时（避免状态无限增长）
- 当你需要**节省内存**时（清理不活跃的状态）
- 当你需要**管理状态生命周期**时（TTL自动管理）
- 当你需要**处理不活跃的key**时（如不活跃的交易对）
- 当你需要**优化状态存储**时（TTL是重要手段）


