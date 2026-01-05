# 窗口函数process()：全量窗口处理

## 核心概念

**`process()`** 是窗口函数中的**全量处理**方法。它可以访问窗口中的所有元素，提供最大的灵活性。

### 核心特点

1. **全量访问**：可以访问窗口中的所有元素
2. **最大灵活性**：可以实现任何复杂的处理逻辑
3. **性能较差**：需要保存所有数据，内存占用大

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

// process：全量处理，可以访问窗口中的所有元素
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .process(new ProcessWindowFunction<Trade, String, String, TimeWindow>() {
          @Override
          public void process(String key, Context ctx, Iterable<Trade> elements, Collector<String> out) {
              // 可以访问窗口中的所有元素
              int count = 0;
              double totalAmount = 0.0;

              for (Trade trade : elements) {
                  count++;
                  totalAmount += trade.getPrice() * trade.getQuantity();
              }

              out.collect(key + ": count=" + count + ", totalAmount=" + totalAmount);
          }
      })
      .print();
```

## 实现 ProcessWindowFunction

```java
public class TradeWindowProcessor extends ProcessWindowFunction<Trade, TradeSummary, String, TimeWindow> {
    @Override
    public void process(String key, Context ctx, Iterable<Trade> elements, Collector<TradeSummary> out) {
        // key: 交易对（如"BTCUSDT"）
        // ctx: 窗口上下文（可以获取窗口信息）
        // elements: 窗口中的所有交易
        // out: 输出收集器

        TradeSummary summary = new TradeSummary();
        summary.setSymbol(key);
        summary.setWindowStart(ctx.window().getStart());
        summary.setWindowEnd(ctx.window().getEnd());

        int count = 0;
        double totalQuantity = 0.0;
        double totalAmount = 0.0;
        double maxPrice = Double.MIN_VALUE;
        double minPrice = Double.MAX_VALUE;

        for (Trade trade : elements) {
            count++;
            totalQuantity += trade.getQuantity();
            totalAmount += trade.getPrice() * trade.getQuantity();
            maxPrice = Math.max(maxPrice, trade.getPrice());
            minPrice = Math.min(minPrice, trade.getPrice());
        }

        summary.setCount(count);
        summary.setTotalQuantity(totalQuantity);
        summary.setTotalAmount(totalAmount);
        summary.setMaxPrice(maxPrice);
        summary.setMinPrice(minPrice);
        summary.setAvgPrice(totalAmount / totalQuantity);

        out.collect(summary);
    }
}
```

## 币安交易数据示例

### 统计窗口详细信息

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .process(new TradeWindowProcessor())
      .print();
```

### 访问窗口信息

```java
@Override
public void process(String key, Context ctx, Iterable<Trade> elements, Collector<String> out) {
    // 获取窗口开始时间
    long windowStart = ctx.window().getStart();

    // 获取窗口结束时间
    long windowEnd = ctx.window().getEnd();

    // 获取当前Watermark
    long currentWatermark = ctx.currentWatermark();

    // 处理窗口中的所有元素
    for (Trade trade : elements) {
        // 处理逻辑
    }
}
```

## 与其他窗口函数的区别

| 函数 | 数据访问 | 内存占用 | 性能 | 灵活性 |
|------|---------|---------|------|--------|
| `reduce()` | 增量（两个元素） | 小 | 最好 | 低 |
| `aggregate()` | 增量（累加器） | 中等 | 好 | 中 |
| `process()` | 全量（所有元素） | 大 | 较差 | 最高 |

## 使用场景

### 场景1：复杂统计

```java
// 需要访问所有元素进行复杂计算
.process(new ComplexStatisticsProcessor())
```

### 场景2：排序、TopN

```java
// 需要访问所有元素进行排序
.process(new TopNProcessor())
```

### 场景3：窗口信息

```java
// 需要窗口的开始、结束时间等信息
.process(new WindowInfoProcessor())
```

## 什么时候你需要想到这个？

- 当你需要**访问窗口中的所有元素**时（全量处理）
- 当你需要**复杂计算**时（排序、TopN等）
- 当你需要**窗口信息**时（开始时间、结束时间）
- 当你需要**最大灵活性**时（process最灵活）
- 当你需要**理解窗口函数**时（最复杂的函数）


