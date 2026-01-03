# 交易量统计：使用状态累计交易量

## 核心概念

使用 ValueState 累计每个交易对的交易量，实现实时统计。

### 核心流程

1. **接收交易数据**：从币安WebSocket获取交易数据
2. **按交易对分组**：使用keyBy按交易对分组
3. **状态累计**：使用ValueState累计交易量
4. **输出结果**：输出累计交易量

## 最小可用例子

```java
DataStream<Trade> trades = env.addSource(new BinanceTradeSource());

trades.keyBy(trade -> trade.getSymbol())
      .process(new KeyedProcessFunction<String, Trade, TradeVolume>() {
          private ValueState<Double> totalVolumeState;

          @Override
          public void open(Configuration parameters) {
              ValueStateDescriptor<Double> descriptor =
                  new ValueStateDescriptor<>("totalVolume", Double.class);
              totalVolumeState = getRuntimeContext().getState(descriptor);
          }

          @Override
          public void processElement(Trade trade, Context ctx, Collector<TradeVolume> out) {
              // 读取当前累计量
              Double totalVolume = totalVolumeState.value();
              if (totalVolume == null) {
                  totalVolume = 0.0;
              }

              // 累加
              totalVolume += trade.getQuantity();

              // 更新状态
              totalVolumeState.update(totalVolume);

              // 输出
              TradeVolume volume = new TradeVolume();
              volume.setSymbol(trade.getSymbol());
              volume.setTotalVolume(totalVolume);
              out.collect(volume);
          }
      })
      .print();
```

## 币安交易数据示例

### 累计交易量

```java
public class VolumeTracker extends KeyedProcessFunction<String, Trade, TradeVolume> {
    private ValueState<Double> totalVolumeState;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Double> descriptor =
            new ValueStateDescriptor<>("totalVolume", Double.class);
        totalVolumeState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<TradeVolume> out) {
        Double totalVolume = totalVolumeState.value();
        if (totalVolume == null) {
            totalVolume = 0.0;
        }

        totalVolume += trade.getQuantity();
        totalVolumeState.update(totalVolume);

        TradeVolume volume = new TradeVolume();
        volume.setSymbol(trade.getSymbol());
        volume.setTotalVolume(totalVolume);
        out.collect(volume);
    }
}
```

## 与窗口统计的区别

| 方式 | 统计范围 | 数据保留 | 适用场景 |
|------|---------|---------|---------|
| **状态累计** | 从开始到当前 | 永久保留 | 累计总量 |
| **窗口统计** | 时间窗口内 | 窗口关闭后清理 | 时间段统计 |

## 什么时候你需要想到这个？

- 当你需要**累计交易量**时（从开始到当前的总量）
- 当你需要使用**状态进行累计**时（ValueState累计）
- 当你需要**实时统计总量**时（不是窗口统计）
- 当你需要理解**状态 vs 窗口**时（不同统计方式）
- 当你需要**构建累计统计**时（总量统计）


