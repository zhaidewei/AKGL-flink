# 结果输出：将处理结果输出到控制台或文件

## 核心概念

将 Flink 处理后的结果输出，用于验证数据处理逻辑和查看结果。

### 常用输出方式

1. **print()** - 输出到控制台（开发测试）
2. **addSink()** - 输出到文件、数据库、Kafka等（生产环境）

## 最小可用例子

### 输出到控制台

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Trade Source"
);

// 简单输出
trades.print();

// 带标签的输出
trades.print("Trades");

// 处理后的结果输出
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .print("Volume per minute");
```

### 输出到文件

```java
// 输出到文件
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .sum("quantity")
      .writeAsText("output/volume.txt");

// 或使用Sink
trades.addSink(new FileSink<>(...));
```

## 币安交易数据示例

### 输出处理结果

```java
DataStream<Trade> trades = env.fromSource(
    new BinanceWebSocketSource("btcusdt"),
    WatermarkStrategy.noWatermarks(),
    "Binance Trade Source"
);

// 1. 输出原始数据
trades.print("Raw Trades");

// 2. 输出处理后的数据
trades.keyBy(trade -> trade.getSymbol())
      .window(TumblingEventTimeWindows.of(Time.minutes(1)))
      .aggregate(new PriceChangeAggregator())
      .print("Price Changes");

// 3. 输出累计统计
trades.keyBy(trade -> trade.getSymbol())
      .process(new VolumeTracker())
      .print("Total Volume");
```

## 不同输出方式

### print() - 控制台输出

```java
// 最简单的方式，用于开发测试
stream.print();
stream.print("Label");
```

### writeAsText() - 文本文件

```java
// 输出到文本文件
stream.writeAsText("output/result.txt");
```

### addSink() - 自定义Sink

```java
// 输出到Kafka
stream.addSink(new FlinkKafkaProducer<>("topic", ...));

// 输出到数据库
stream.addSink(new JdbcSink<>("INSERT INTO ...", ...));
```

## 什么时候你需要想到这个？

- 当你需要**验证数据处理结果**时（输出查看）
- 当你需要**调试Flink作业**时（print输出）
- 当你需要**保存处理结果**时（输出到文件）
- 当你需要**理解数据流**时（查看输出）
- 当你需要**构建完整流程**时（从输入到输出）


