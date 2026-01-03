# 会话窗口（SessionWindow）：基于活动间隔

## 核心概念

**会话窗口（Session Window）** 是基于**数据活动间隔**动态创建的窗口。如果数据之间的间隔超过设定的时间（gap），就关闭当前窗口并开启新窗口。

### 类比理解

这就像：
- **用户会话**：用户活动时窗口开启，不活动一段时间后关闭
- **交易会话**：连续交易时窗口开启，没有交易一段时间后关闭
- **活动窗口**：基于活动动态创建窗口

### 核心特点

1. **动态大小**：窗口大小不固定，取决于数据活动
2. **基于间隔**：数据间隔超过gap时关闭窗口
3. **活动驱动**：窗口的开启和关闭由数据活动决定

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 会话窗口：如果5分钟内没有新数据，关闭窗口
trades.keyBy(trade -> trade.getSymbol())
      .window(EventTimeSessionWindows.withGap(Time.minutes(5)))
      .sum("quantity")
      .print();

// 窗口划分示例：
// 12:00:00 - 交易1 → 开启窗口1
// 12:00:10 - 交易2 → 窗口1继续
// 12:00:20 - 交易3 → 窗口1继续
// 12:05:30 - 交易4 → 窗口1关闭（间隔5分钟），开启窗口2
// 12:05:40 - 交易5 → 窗口2继续
```

## 窗口划分示例

```
时间线:
  12:00:00    12:00:10    12:00:20    12:05:30    12:05:40
    │            │            │            │            │
    └─ 交易1      └─ 交易2      └─ 交易3      └─ 交易4      └─ 交易5
    │            │            │            │            │
    └────────── 窗口1 ──────────┘            │            │
                                              │            │
                                              └────────── 窗口2 ──┘

规则：如果数据间隔 > 5分钟，关闭窗口
  交易3到交易4间隔5分10秒 > 5分钟 → 关闭窗口1，开启窗口2
```

## 币安交易数据示例

### 交易会话统计

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 会话窗口：如果3分钟内没有新交易，关闭窗口
// 用于识别"交易会话"（连续交易的时间段）
trades.keyBy(trade -> trade.getSymbol())
      .window(EventTimeSessionWindows.withGap(Time.minutes(3)))
      .aggregate(new SessionAggregator())  // 统计会话内的交易
      .print();
```

### 用户活动会话

```java
// 如果用户5分钟内没有活动，关闭会话窗口
DataStream<UserActivity> activities = env.addSource(new ActivitySource());

activities.keyBy(activity -> activity.getUserId())
          .window(EventTimeSessionWindows.withGap(Time.minutes(5)))
          .aggregate(new SessionAggregator())
          .print();
```

## Gap 设置

```java
// 短间隔：3分钟（适合高频交易）
.window(EventTimeSessionWindows.withGap(Time.minutes(3)))

// 中等间隔：5分钟（适合一般场景）
.window(EventTimeSessionWindows.withGap(Time.minutes(5)))

// 长间隔：10分钟（适合低频场景）
.window(EventTimeSessionWindows.withGap(Time.minutes(10)))
```

## 与其他窗口的区别

| 特性 | 滚动窗口 | 滑动窗口 | 会话窗口 |
|------|---------|---------|---------|
| 窗口大小 | 固定 | 固定 | 动态 |
| 窗口数量 | 固定 | 固定 | 动态 |
| 适用场景 | 固定间隔统计 | 滑动平均 | 活动会话 |

## 使用场景

### 场景1：交易会话识别

```java
// 识别连续交易的时间段（会话）
.window(EventTimeSessionWindows.withGap(Time.minutes(5)))
```

### 场景2：用户活动会话

```java
// 识别用户连续活动的时间段
.window(EventTimeSessionWindows.withGap(Time.minutes(10)))
```

## 什么时候你需要想到这个？

- 当你需要**识别活动会话**时（连续活动的时间段）
- 当你需要**动态窗口大小**时（窗口大小不固定）
- 当你需要**基于活动间隔**创建窗口时（gap机制）
- 当你需要理解**窗口类型**时（会话窗口的特点）
- 当你需要**分析用户行为**时（会话分析）


