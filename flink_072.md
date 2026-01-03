# 实时价格监控：使用窗口统计价格变化

## 核心概念

使用窗口操作统计币安交易对的实时价格变化，计算涨跌幅等指标。

### 核心流程

1. **接收交易数据**：从币安WebSocket获取交易数据
2. **设置事件时间**：提取交易时间作为事件时间
3. **窗口统计**：使用窗口统计价格变化
4. **输出结果**：输出价格变化、涨跌幅等

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 1. 获取交易数据
DataStream<Trade> trades = env.addSource(new BinanceTradeSource());

// 2. 设置事件时间
DataStream<Trade> withEventTime = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 3. 窗口统计：每分钟计算价格变化
withEventTime.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .aggregate(new PriceChangeAggregator())
              .print();

env.execute();
```

## 价格变化聚合器

```java
public class PriceChangeAccumulator {
    public double firstPrice = 0.0;
    public double lastPrice = 0.0;
    public int count = 0;
}

public class PriceChangeAggregator implements AggregateFunction<Trade, PriceChangeAccumulator, PriceChange> {
    @Override
    public PriceChangeAccumulator createAccumulator() {
        return new PriceChangeAccumulator();
    }

    @Override
    public PriceChangeAccumulator add(Trade trade, PriceChangeAccumulator acc) {
        if (acc.count == 0) {
            acc.firstPrice = trade.getPrice();
        }
        acc.lastPrice = trade.getPrice();
        acc.count++;
        return acc;
    }

    @Override
    public PriceChange getResult(PriceChangeAccumulator acc) {
        PriceChange change = new PriceChange();
        change.setFirstPrice(acc.firstPrice);
        change.setLastPrice(acc.lastPrice);
        change.setChange(acc.lastPrice - acc.firstPrice);
        change.setChangePercent((acc.lastPrice - acc.firstPrice) / acc.firstPrice * 100);
        return change;
    }

    @Override
    public PriceChangeAccumulator merge(PriceChangeAccumulator a, PriceChangeAccumulator b) {
        // 合并逻辑
        return a;
    }
}
```

## 币安交易数据示例

### 实时价格监控

```java
DataStream<Trade> trades = env.addSource(new BinanceTradeSource());

DataStream<Trade> withEventTime = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);

// 每分钟统计价格变化
withEventTime.keyBy(trade -> trade.getSymbol())
              .window(TumblingEventTimeWindows.of(Time.minutes(1)))
              .aggregate(new PriceChangeAggregator())
              .print();
```

## 什么时候你需要想到这个？

- 当你需要**实时监控价格变化**时（涨跌幅统计）
- 当你需要使用**窗口统计**时（时间范围内的变化）
- 当你需要**计算价格指标**时（变化、百分比等）
- 当你需要**理解窗口应用**时（实际场景）
- 当你需要**构建实时监控系统**时（价格监控）


