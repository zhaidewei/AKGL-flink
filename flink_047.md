# 窗口的概念：为什么需要窗口

## 核心概念

**窗口（Window）** 是将**无界数据流**划分为**有限的数据块**进行处理。因为流数据是无限的，我们需要窗口来定义"在什么时间范围内处理数据"。

### 类比理解

这就像：
- **时间桶**：将时间分成一个个桶，每个桶处理一段时间的数据
- **SQL的GROUP BY**：按时间分组统计
- **滑动平均**：计算一段时间内的平均值

### 为什么需要窗口？

1. **无界流处理**：流数据是无限的，需要划分成有限块
2. **时间范围统计**：需要统计"每分钟"、"每小时"的数据
3. **聚合计算**：在窗口内进行聚合（求和、平均等）

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 没有窗口：无法聚合（流是无限的）
// trades.sum("quantity");  // ❌ 无法对无限流求和

// 使用窗口：将流划分为1分钟的窗口
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))  // 1分钟窗口
      .sum("quantity")  // 对每个窗口内的数据求和
      .print();
```

## 窗口的作用

### 场景：统计每分钟交易量

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 问题：如何统计"每分钟"的交易量？
// 流是无限的，没有"每分钟"的概念

// 解决：使用窗口
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))  // 1分钟窗口
      .sum("quantity")  // 每个窗口内求和
      .print();

// 结果：
// [12:00:00, 12:01:00) - BTC交易量: 100
// [12:01:00, 12:02:00) - BTC交易量: 150
// [12:02:00, 12:03:00) - BTC交易量: 120
```

## 窗口类型

### 1. 滚动窗口（Tumbling Window）

```java
// 固定大小，不重叠
// [12:00:00, 12:01:00), [12:01:00, 12:02:00), [12:02:00, 12:03:00)
.window(TumblingEventTimeWindows.of(Time.minutes(1)))
```

### 2. 滑动窗口（Sliding Window）

```java
// 固定大小，可重叠
// [12:00:00, 12:01:00), [12:00:30, 12:01:30), [12:01:00, 12:02:00)
.window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(30)))
```

### 3. 会话窗口（Session Window）

```java
// 基于活动间隔
// 如果5分钟内没有数据，窗口关闭
.window(EventTimeSessionWindows.withGap(Time.minutes(5)))
```

## 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 统计每个交易对每分钟的交易量
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .print();

// 统计每个交易对每5分钟的平均价格
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(5)))
      .aggregate(new AveragePriceAggregator())
      .print();
```

## 什么时候你需要想到这个？

- 当你需要**统计时间范围内的数据**时（每分钟、每小时）
- 当你需要对**无界流进行聚合**时（求和、平均等）
- 当你需要**划分数据流**时（将无限流变成有限块）
- 当你需要**实现时间窗口计算**时（窗口是基础）
- 当你需要**理解流处理的核心概念**时（窗口是必须的）


