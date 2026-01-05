# ProcessFunction.processElement()：处理每个元素

## 核心概念

**`processElement()`** 是 ProcessFunction 的**核心方法**，每条数据都会调用这个方法进行处理。

### 方法签名

```java
public abstract void processElement(
    T value,                    // 当前处理的元素
    Context ctx,                // 上下文（时间、定时器等）
    Collector<O> out            // 输出收集器
) throws Exception;
```

### 核心特点

1. **每条数据调用**：流中的每条数据都会调用
2. **访问上下文**：可以通过 ctx 访问时间、定时器等
3. **输出数据**：通过 out.collect() 输出结果

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

trades.keyBy(trade -> trade.getSymbol())
      .process(new KeyedProcessFunction<String, Trade, String>() {
          @Override
          public void processElement(Trade trade, Context ctx, Collector<String> out) {
              // 处理每条交易数据
              String result = trade.getSymbol() + ": " + trade.getPrice();
              out.collect(result);
          }
      })
      .print();
```

## 币安交易数据示例

### 处理交易数据

```java
trades.keyBy(trade -> trade.getSymbol())
      .process(new KeyedProcessFunction<String, Trade, TradeSummary>() {
          private ValueState<Double> lastPriceState;

          @Override
          public void open(Configuration parameters) {
              lastPriceState = getRuntimeContext().getState(
                  new ValueStateDescriptor<>("lastPrice", Double.class));
          }

          @Override
          public void processElement(Trade trade, Context ctx, Collector<TradeSummary> out) {
              // 1. 获取状态
              Double lastPrice = lastPriceState.value();

              // 2. 处理数据
              TradeSummary summary = new TradeSummary();
              summary.setSymbol(trade.getSymbol());
              summary.setPrice(trade.getPrice());
              summary.setQuantity(trade.getQuantity());

              if (lastPrice != null) {
                  summary.setPriceChange(trade.getPrice() - lastPrice);
              }

              // 3. 更新状态
              lastPriceState.update(trade.getPrice());

              // 4. 输出结果
              out.collect(summary);
          }
      })
      .print();
```

## 访问上下文

### 获取时间

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<String> out) {
    // 处理时间
    long processingTime = ctx.timerService().currentProcessingTime();

    // 事件时间（如果使用事件时间）
    Long eventTime = ctx.timestamp();

    // 当前Watermark
    long watermark = ctx.timerService().currentWatermark();
}
```

### 注册定时器

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<String> out) {
    // 注册处理时间定时器（5秒后触发）
    long processingTime = ctx.timerService().currentProcessingTime();
    ctx.timerService().registerProcessingTimeTimer(processingTime + 5000);

    // 注册事件时间定时器（事件时间+10秒后触发）
    if (ctx.timestamp() != null) {
        ctx.timerService().registerEventTimeTimer(ctx.timestamp() + 10000);
    }
}
```

## 处理流程

```
数据流: [交易1, 交易2, 交易3, ...]

处理过程:
  交易1 → processElement(交易1, ctx, out) → 输出结果1
  交易2 → processElement(交易2, ctx, out) → 输出结果2
  交易3 → processElement(交易3, ctx, out) → 输出结果3
  ...
```

## 什么时候你需要想到这个？

- 当你需要**实现 ProcessFunction** 时（核心方法）
- 当你需要**处理每条数据**时（processElement的作用）
- 当你需要**访问状态和时间**时（通过Context）
- 当你需要**输出处理结果**时（通过Collector）
- 当你需要**理解Flink的处理机制**时（元素处理流程）


