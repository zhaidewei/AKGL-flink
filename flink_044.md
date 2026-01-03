# WatermarkGenerator：生成Watermark

## 核心概念

**WatermarkGenerator** 是用于**生成 Watermark** 的接口。你可以通过实现这个接口来自定义 Watermark 的生成逻辑。

### 核心方法

```java
public interface WatermarkGenerator<T> {
    // 当收到数据时调用
    void onEvent(T event, long eventTimestamp, WatermarkOutput output);

    // 周期性调用（如果使用周期性生成）
    void onPeriodicEmit(WatermarkOutput output);
}
```

## 源码位置

WatermarkGenerator 接口在：
[flink-api/src/main/java/org/apache/flink/api/common/eventtime/WatermarkGenerator.java](https://github.com/apache/flink/blob/master/flink-api/src/main/java/org/apache/flink/api/common/eventtime/WatermarkGenerator.java)

## 最小可用例子

### 使用内置策略（推荐）

```java
// 使用内置的BoundedOutOfOrderness策略（推荐）
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

### 自定义 WatermarkGenerator

```java
public class CustomWatermarkGenerator implements WatermarkGenerator<Trade> {
    private long maxOutOfOrderness = 10000;  // 10秒
    private long currentMaxTimestamp = Long.MIN_VALUE;

    @Override
    public void onEvent(Trade trade, long eventTimestamp, WatermarkOutput output) {
        // 更新最大时间戳
        currentMaxTimestamp = Math.max(currentMaxTimestamp, eventTimestamp);
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // 周期性生成Watermark
        output.emitWatermark(new Watermark(currentMaxTimestamp - maxOutOfOrderness));
    }
}

// 使用
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.forGenerator(ctx -> new CustomWatermarkGenerator())
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

## 币安交易数据示例

```java
// 推荐：使用内置策略
DataStream<Trade> trades = env.addSource(new BinanceSource());

DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

## 标点式生成器

```java
public class PunctuatedWatermarkGenerator implements WatermarkGenerator<Trade> {
    @Override
    public void onEvent(Trade trade, long eventTimestamp, WatermarkOutput output) {
        // 每收到一条数据就生成Watermark
        output.emitWatermark(new Watermark(eventTimestamp - 10000));
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // 标点式生成不需要这个方法
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**自定义Watermark生成逻辑**时（复杂场景）
- 当你需要理解**Watermark生成机制**时（如何生成）
- 当你需要**优化Watermark策略**时（自定义生成器）
- 当你需要**实现标点式生成**时（基于数据生成）
- 当你使用**内置策略不满足需求**时（自定义实现）


