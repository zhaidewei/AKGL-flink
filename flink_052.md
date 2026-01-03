# 窗口函数aggregate()：自定义聚合逻辑

## 核心概念

**`aggregate()`** 是窗口函数中的**自定义聚合**方法。它提供了更大的灵活性，可以自定义聚合逻辑和输出类型。

### 核心特点

1. **自定义聚合**：可以定义自己的聚合逻辑
2. **类型可变**：输入和输出类型可以不同
3. **灵活性高**：支持复杂的聚合计算

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// aggregate：自定义聚合，计算平均价格
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .aggregate(new AggregateFunction<Trade, AvgPriceAccumulator, Double>() {
          @Override
          public AvgPriceAccumulator createAccumulator() {
              return new AvgPriceAccumulator();
          }

          @Override
          public AvgPriceAccumulator add(Trade trade, AvgPriceAccumulator acc) {
              acc.sum += trade.getPrice();
              acc.count++;
              return acc;
          }

          @Override
          public Double getResult(AvgPriceAccumulator acc) {
              return acc.sum / acc.count;
          }

          @Override
          public AvgPriceAccumulator merge(AvgPriceAccumulator a, AvgPriceAccumulator b) {
              a.sum += b.sum;
              a.count += b.count;
              return a;
          }
      })
      .print();
```

## 实现 AggregateFunction

### 累加器类

```java
public class AvgPriceAccumulator {
    public double sum = 0.0;
    public int count = 0;
}
```

### 聚合函数

```java
public class AveragePriceAggregator implements AggregateFunction<Trade, AvgPriceAccumulator, Double> {
    @Override
    public AvgPriceAccumulator createAccumulator() {
        return new AvgPriceAccumulator();
    }

    @Override
    public AvgPriceAccumulator add(Trade trade, AvgPriceAccumulator acc) {
        acc.sum += trade.getPrice();
        acc.count++;
        return acc;
    }

    @Override
    public Double getResult(AvgPriceAccumulator acc) {
        return acc.count == 0 ? 0.0 : acc.sum / acc.count;
    }

    @Override
    public AvgPriceAccumulator merge(AvgPriceAccumulator a, AvgPriceAccumulator b) {
        a.sum += b.sum;
        a.count += b.count;
        return a;
    }
}
```

## 币安交易数据示例

### 计算平均价格

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .aggregate(new AveragePriceAggregator())
      .print();
```

### 计算最大价格和最小价格

```java
public class PriceRangeAccumulator {
    public double maxPrice = Double.MIN_VALUE;
    public double minPrice = Double.MAX_VALUE;
}

public class PriceRangeAggregator implements AggregateFunction<Trade, PriceRangeAccumulator, String> {
    @Override
    public PriceRangeAccumulator createAccumulator() {
        return new PriceRangeAccumulator();
    }

    @Override
    public PriceRangeAccumulator add(Trade trade, PriceRangeAccumulator acc) {
        acc.maxPrice = Math.max(acc.maxPrice, trade.getPrice());
        acc.minPrice = Math.min(acc.minPrice, trade.getPrice());
        return acc;
    }

    @Override
    public String getResult(PriceRangeAccumulator acc) {
        return "Max: " + acc.maxPrice + ", Min: " + acc.minPrice;
    }

    @Override
    public PriceRangeAccumulator merge(PriceRangeAccumulator a, PriceRangeAccumulator b) {
        a.maxPrice = Math.max(a.maxPrice, b.maxPrice);
        a.minPrice = Math.min(a.minPrice, b.minPrice);
        return a;
    }
}
```

## 与其他窗口函数的区别

| 函数 | 处理方式 | 类型 | 灵活性 | 性能 |
|------|---------|------|--------|------|
| `reduce()` | 增量聚合 | 不变 | 低 | 最好 |
| `aggregate()` | 自定义聚合 | 可变 | 中 | 好 |
| `process()` | 全量处理 | 可变 | 最高 | 较差 |

## 什么时候你需要想到这个？

- 当你需要**自定义聚合逻辑**时（复杂计算）
- 当你需要**改变输出类型**时（Trade → Double）
- 当你需要**计算平均值、最大值等**时（需要累加器）
- 当你需要**灵活聚合**时（aggregate比reduce灵活）
- 当你需要**理解窗口函数**时（中等复杂度的函数）


