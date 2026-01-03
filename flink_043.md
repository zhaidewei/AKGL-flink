# TimestampAssigner：提取事件时间戳

## 核心概念

**TimestampAssigner** 用于从数据中**提取事件时间戳**。它是 `assignTimestampsAndWatermarks()` 中必须配置的部分。

### 核心作用

1. **提取时间戳**：从数据对象中提取事件时间
2. **返回毫秒数**：返回从1970-01-01 00:00:00 UTC开始的毫秒数
3. **支持事件时间**：为事件时间处理提供时间戳

## 源码位置

TimestampAssigner 接口在：
[flink-api/src/main/java/org/apache/flink/api/common/eventtime/TimestampAssigner.java](https://github.com/apache/flink/blob/master/flink-api/src/main/java/org/apache/flink/api/common/eventtime/TimestampAssigner.java)

## 最小可用例子

### 方式1：Lambda 表达式（推荐）

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

DataStream<Trade> withTimestamps = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getTradeTime())
);
```

### 方式2：实现接口

```java
public class TradeTimestampAssigner implements TimestampAssigner<Trade> {
    @Override
    public long extractTimestamp(Trade trade, long recordTimestamp) {
        return trade.getTradeTime();  // 返回事件时间（毫秒）
    }
}

// 使用
DataStream<Trade> withTimestamps = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner(new TradeTimestampAssigner())
);
```

## 币安交易数据示例

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

// 从Trade对象中提取tradeTime字段作为事件时间
DataStream<Trade> withEventTime = trades.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> {
            // 返回币安交易的实际发生时间
            return trade.getTradeTime();  // 毫秒时间戳
        })
);
```

## 时间戳格式

时间戳必须是**毫秒数**，从 1970-01-01 00:00:00 UTC 开始：

```java
// 币安返回的时间戳通常是毫秒
long tradeTime = trade.getTradeTime();  // 已经是毫秒

// 如果是秒，需要转换
long tradeTimeSeconds = trade.getTradeTimeSeconds();
long tradeTimeMillis = tradeTimeSeconds * 1000;  // 转换为毫秒
```

## 复杂提取逻辑

```java
public class ComplexTimestampAssigner implements TimestampAssigner<Trade> {
    @Override
    public long extractTimestamp(Trade trade, long recordTimestamp) {
        // 如果tradeTime为空，使用当前时间
        if (trade.getTradeTime() == 0) {
            return System.currentTimeMillis();
        }
        return trade.getTradeTime();
    }
}
```

## 什么时候你需要想到这个？

- 当你需要**从数据中提取时间戳**时（事件时间处理）
- 当你需要**设置事件时间**时（assignTimestampsAndWatermarks）
- 当你需要**处理时间戳格式**时（秒转毫秒等）
- 当你需要**自定义时间戳提取逻辑**时（复杂场景）
- 当你需要**理解事件时间处理**时（时间戳是基础）


