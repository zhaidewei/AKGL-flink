# ProcessFunction：访问状态和时间的函数

## 核心概念

**ProcessFunction** 是 Flink 中用于**访问状态和时间**的函数。它提供了对每条数据进行处理的能力，同时可以访问状态、时间和服务。

### 核心特点

1. **访问状态**：可以读写 ValueState、MapState 等
2. **访问时间**：可以获取处理时间、事件时间、Watermark
3. **定时器**：可以注册定时器，在指定时间触发
4. **灵活性高**：可以实现复杂的处理逻辑

## 源码位置

ProcessFunction 接口在：
[flink-datastream-api/src/main/java/org/apache/flink/datastream/api/function/ProcessFunction.java](https://github.com/apache/flink/blob/master/flink-datastream-api/src/main/java/org/apache/flink/datastream/api/function/ProcessFunction.java)

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

KeyedStream<Trade, String> keyed = trades.keyBy(trade -> trade.getSymbol());

// ProcessFunction：处理每条数据，可以访问状态和时间
keyed.process(new KeyedProcessFunction<String, Trade, String>() {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        // 初始化状态
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);
        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 访问状态
        Double lastPrice = lastPriceState.value();

        // 访问时间
        long processingTime = ctx.timerService().currentProcessingTime();
        long eventTime = ctx.timestamp();

        if (lastPrice != null) {
            double change = trade.getPrice() - lastPrice;
            out.collect(trade.getSymbol() + " price changed: " + change);
        }

        // 更新状态
        lastPriceState.update(trade.getPrice());
    }
})
.print();
```

## 币安交易数据示例

### 跟踪价格变化

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Source"
);

trades.keyBy(trade -> trade.getSymbol())
      .process(new KeyedProcessFunction<String, Trade, PriceChange>() {
          private ValueState<Double> lastPriceState;

          @Override
          public void open(Configuration parameters) {
              lastPriceState = getRuntimeContext().getState(
                  new ValueStateDescriptor<>("lastPrice", Double.class));
          }

          @Override
          public void processElement(Trade trade, Context ctx, Collector<PriceChange> out) {
              Double lastPrice = lastPriceState.value();

              if (lastPrice != null) {
                  PriceChange change = new PriceChange();
                  change.setSymbol(trade.getSymbol());
                  change.setChange(trade.getPrice() - lastPrice);
                  change.setChangePercent((trade.getPrice() - lastPrice) / lastPrice * 100);
                  out.collect(change);
              }

              lastPriceState.update(trade.getPrice());
          }
      })
      .print();
```

## 关键方法

### processElement()

```java
@Override
public void processElement(Trade trade, Context ctx, Collector<String> out) {
    // trade: 当前处理的元素
    // ctx: 上下文（可以访问时间、定时器等）
    // out: 输出收集器
}
```

### 访问时间

```java
// 处理时间
long processingTime = ctx.timerService().currentProcessingTime();

// 事件时间
long eventTime = ctx.timestamp();

// 当前Watermark
long watermark = ctx.timerService().currentWatermark();
```

### 访问状态

```java
// 读取状态
Double lastPrice = lastPriceState.value();

// 更新状态
lastPriceState.update(trade.getPrice());
```

## 与 map/filter 的区别

| 函数 | 状态访问 | 时间访问 | 定时器 | 灵活性 |
|------|---------|---------|--------|--------|
| `map()` | ❌ | ❌ | ❌ | 低 |
| `filter()` | ❌ | ❌ | ❌ | 低 |
| `ProcessFunction` | ✅ | ✅ | ✅ | 最高 |

## 什么时候你需要想到这个？

- 当你需要**访问状态**时（ValueState、MapState等）
- 当你需要**访问时间**时（处理时间、事件时间）
- 当你需要**使用定时器**时（延迟处理、超时检测）
- 当你需要**复杂处理逻辑**时（ProcessFunction最灵活）
- 当你需要**理解Flink的状态处理**时（ProcessFunction是基础）


