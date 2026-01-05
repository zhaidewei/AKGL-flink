# 多交易对处理：使用keyBy按交易对分组

## 核心概念

使用 `keyBy()` 按交易对分组，为每个交易对独立处理数据。这是处理多个交易对的标准方式。

### 核心流程

1. **接收所有交易对数据**：从币安WebSocket获取多个交易对的数据
2. **按交易对分组**：使用keyBy按symbol分组
3. **独立处理**：每个交易对独立处理（状态隔离）

## 最小可用例子

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Trade Source"
);

// 按交易对分组
KeyedStream<Trade, String> keyed = trades.keyBy(trade -> trade.getSymbol());

// 每个交易对独立处理
keyed.process(new KeyedProcessFunction<String, Trade, String>() {
    private ValueState<Double> lastPriceState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("lastPrice", Double.class);
        lastPriceState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 每个交易对（BTC、ETH等）有独立的状态
        Double lastPrice = lastPriceState.value();
        if (lastPrice != null) {
            out.collect(trade.getSymbol() + " changed: " + (trade.getPrice() - lastPrice));
        }
        lastPriceState.update(trade.getPrice());
    }
})
.print();
```

## 币安交易数据示例

### 处理多个交易对

```java
// 假设SourceFunction接收所有交易对的数据
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Trade Source"
);

// 按交易对分组
KeyedStream<Trade, String> keyed = trades.keyBy(trade -> trade.getSymbol());

// 每个交易对独立统计
keyed.window(TumblingEventTimeWindows.of(Time.minutes(1)))
     .sum("quantity")
     .print();

// 结果：
// Key=BTCUSDT: 每分钟交易量
// Key=ETHUSDT: 每分钟交易量
// Key=BNBUSDT: 每分钟交易量
// （每个key独立计算）
```

## 状态隔离示例

```java
// 每个交易对有独立的状态
// Key=BTCUSDT: lastPrice = 50000
// Key=ETHUSDT: lastPrice = 3000
// Key=BNBUSDT: lastPrice = 400
// （完全独立，互不影响）

keyed.process(new KeyedProcessFunction<String, Trade, PriceChange>() {
    private ValueState<Double> lastPriceState;

    @Override
    public void processElement(Trade trade, Context ctx, Collector<PriceChange> out) {
        // 只访问当前key的状态（如BTC的状态不影响ETH）
        Double lastPrice = lastPriceState.value();
        // ...
    }
});
```

## 什么时候你需要想到这个？

- 当你需要**处理多个交易对**时（BTC、ETH等）
- 当你需要**按交易对分组处理**时（keyBy）
- 当你需要**独立的状态管理**时（每个交易对独立状态）
- 当你需要理解**状态隔离**时（不同key的状态独立）
- 当你需要**构建多交易对处理系统**时（分组处理）


