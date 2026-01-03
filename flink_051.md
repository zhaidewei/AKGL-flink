# 窗口函数reduce()：增量聚合

## 核心概念

**`reduce()`** 是窗口函数中的**增量聚合**方法。它对窗口内的数据进行增量合并，每次只处理两个元素，性能好。

### 核心特点

1. **增量聚合**：每次合并两个元素，不需要保存所有数据
2. **性能好**：内存占用小，计算效率高
3. **类型不变**：输入和输出类型相同

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// reduce：增量聚合，计算总交易量
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .reduce((trade1, trade2) -> {
          // 合并两个交易：累加数量
          trade1.setQuantity(trade1.getQuantity() + trade2.getQuantity());
          return trade1;
      })
      .print();
```

## 工作原理

### 增量合并过程

```
窗口数据: [交易1(10), 交易2(20), 交易3(30)]

reduce过程:
  步骤1: reduce(交易1, 交易2) → 交易1(30)  [10+20]
  步骤2: reduce(交易1(30), 交易3) → 交易1(60)  [30+30]

结果: 交易1(数量=60)
```

## 币安交易数据示例

### 计算总交易量

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .reduce((t1, t2) -> {
          // 累加数量
          t1.setQuantity(t1.getQuantity() + t2.getQuantity());
          return t1;
      })
      .print();
```

### 计算总交易额

```java
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .reduce((t1, t2) -> {
          // 累加交易额
          double amount1 = t1.getPrice() * t1.getQuantity();
          double amount2 = t2.getPrice() * t2.getQuantity();
          t1.setQuantity((amount1 + amount2) / t1.getPrice());
          return t1;
      })
      .print();
```

## 实现 ReduceFunction

```java
public class TradeQuantityReducer implements ReduceFunction<Trade> {
    @Override
    public Trade reduce(Trade trade1, Trade trade2) throws Exception {
        // 累加数量
        trade1.setQuantity(trade1.getQuantity() + trade2.getQuantity());
        return trade1;
    }
}

// 使用
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .reduce(new TradeQuantityReducer())
      .print();
```

## 与其他窗口函数的区别

| 函数 | 处理方式 | 内存占用 | 性能 | 灵活性 |
|------|---------|---------|------|--------|
| `reduce()` | 增量聚合 | 小 | 最好 | 低 |
| `aggregate()` | 自定义聚合 | 中等 | 好 | 中 |
| `process()` | 全量处理 | 大 | 较差 | 最高 |

## 什么时候你需要想到这个？

- 当你需要**增量聚合**时（累加、求和等）
- 当你需要**高性能**时（reduce性能最好）
- 当你需要**简单聚合**时（类型不变的聚合）
- 当你需要**减少内存占用**时（增量处理）
- 当你需要**理解窗口函数**时（从最简单的开始）


