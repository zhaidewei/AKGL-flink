# 标点式Watermark生成：基于数据生成

## 核心概念

**标点式 Watermark 生成** 是基于**特定数据事件**生成 Watermark 的策略。当收到特定数据时，立即生成 Watermark。

### 核心特点

1. **事件驱动**：基于特定数据生成
2. **实时性好**：立即生成，延迟低
3. **灵活控制**：可以精确控制生成时机

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// 标点式Watermark生成：每收到一条数据就生成
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
        // 默认是周期性生成，需要自定义实现标点式生成
);

// 或者使用自定义WatermarkGenerator
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.forGenerator(ctx -> new PunctuatedWatermarkGenerator())
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

## 自定义标点式生成器

```java
public class PunctuatedWatermarkGenerator implements WatermarkGenerator<Trade> {
    @Override
    public void onEvent(Trade trade, long eventTimestamp, WatermarkOutput output) {
        // 每收到一条数据就生成Watermark
        output.emitWatermark(new Watermark(eventTimestamp - 10000));  // 延迟10秒
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // 标点式生成不需要这个方法
    }
}

// 使用
DataStream<Trade> withWatermark = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.forGenerator(ctx -> new PunctuatedWatermarkGenerator())
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

## 与周期性生成的对比

| 特性 | 周期性生成 | 标点式生成 |
|------|-----------|-----------|
| 生成时机 | 定时（每200ms） | 基于数据 |
| 实时性 | 稍差（最多延迟200ms） | 好（立即生成） |
| 性能 | 好（批量生成） | 较好 |
| 使用频率 | ⭐⭐⭐⭐⭐ | ⭐⭐ |

## 使用场景

### 场景1：需要低延迟

```java
// 需要立即生成Watermark，不能等待定时器
WatermarkStrategy.forGenerator(ctx -> new PunctuatedWatermarkGenerator())
```

### 场景2：基于特定数据生成

```java
// 只有特定数据才生成Watermark
public class ConditionalWatermarkGenerator implements WatermarkGenerator<Trade> {
    @Override
    public void onEvent(Trade trade, long eventTimestamp, WatermarkOutput output) {
        // 只有大额交易才生成Watermark
        if (trade.getPrice() * trade.getQuantity() > 10000) {
            output.emitWatermark(new Watermark(eventTimestamp - 10000));
        }
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**低延迟Watermark生成**时（立即生成）
- 当你需要**基于特定数据生成Watermark**时（条件生成）
- 当你需要理解**Watermark生成策略**时（周期性 vs 标点式）
- 当你需要**优化实时性**时（减少Watermark延迟）
- 当你需要**自定义Watermark生成逻辑**时（复杂场景）

