# ListState：存储元素列表

## 核心概念

**ListState** 用于存储**元素列表**的状态。类似于 Java 的 List，可以存储多个元素，支持添加、获取、删除等操作。

### 核心特点

1. **列表存储**：可以存储多个元素
2. **有序**：保持元素的添加顺序
3. **可重复**：可以存储重复的元素

## 源码位置

ListState 接口在：
[flink-core-api/src/main/java/org/apache/flink/api/common/state/ListState.java](https://github.com/apache/flink/blob/master/flink-core-api/src/main/java/org/apache/flink/api/common/state/ListState.java)

## 最小可用例子

```java
public class TradeListTracker extends KeyedProcessFunction<String, Trade, String> {
    private ListState<Trade> tradeListState;

    @Override
    public void open(Configuration parameters) {
        ListStateDescriptor<Trade> descriptor =
            new ListStateDescriptor<>("tradeList", Trade.class);
        tradeListState = getRuntimeContext().getListState(descriptor);
    }

    @Override
    public void processElement(Trade trade, Context ctx, Collector<String> out) {
        // 添加元素到列表
        tradeListState.add(trade);

        // 获取所有元素
        List<Trade> trades = new ArrayList<>();
        for (Trade t : tradeListState.get()) {
            trades.add(t);
        }

        out.collect("Total trades: " + trades.size());
    }
}
```

## 币安交易数据示例

### 存储最近N笔交易

```java
DataStream<Trade> trades = env.addSource(new BinanceSource());

trades.keyBy(trade -> trade.getSymbol())
      .process(new KeyedProcessFunction<String, Trade, List<Trade>>() {
          private ListState<Trade> recentTradesState;
          private static final int MAX_SIZE = 10;

          @Override
          public void open(Configuration parameters) {
              ListStateDescriptor<Trade> descriptor =
                  new ListStateDescriptor<>("recentTrades", Trade.class);
              recentTradesState = getRuntimeContext().getListState(descriptor);
          }

          @Override
          public void processElement(Trade trade, Context ctx, Collector<List<Trade>> out) {
              // 添加新交易
              recentTradesState.add(trade);

              // 获取所有交易
              List<Trade> allTrades = new ArrayList<>();
              for (Trade t : recentTradesState.get()) {
                  allTrades.add(t);
              }

              // 保持最近N笔
              if (allTrades.size() > MAX_SIZE) {
                  recentTradesState.clear();
                  // 只保留最近N笔
                  List<Trade> recent = allTrades.subList(
                      allTrades.size() - MAX_SIZE, allTrades.size());
                  for (Trade t : recent) {
                      recentTradesState.add(t);
                  }
              }

              out.collect(allTrades);
          }
      })
      .print();
```

## 基本操作

### add() - 添加元素

```java
// 添加元素到列表
tradeListState.add(trade);

// 可以添加多个
tradeListState.add(trade1);
tradeListState.add(trade2);
```

### get() - 获取所有元素

```java
// 获取所有元素（返回Iterable）
Iterable<Trade> trades = tradeListState.get();

// 转换为List
List<Trade> tradeList = new ArrayList<>();
for (Trade trade : tradeListState.get()) {
    tradeList.add(trade);
}
```

### update() - 更新整个列表

```java
// 更新整个列表（替换所有元素）
List<Trade> newList = Arrays.asList(trade1, trade2, trade3);
tradeListState.update(newList);
```

### clear() - 清空列表

```java
// 清空所有元素
tradeListState.clear();
```

## 与 ValueState 的区别

| 特性 | ValueState | ListState |
|------|-----------|-----------|
| 存储内容 | 单个值 | 元素列表 |
| 访问方式 | value() | get()（返回Iterable） |
| 适用场景 | 单个值 | 多个元素 |

## 什么时候你需要想到这个？

- 当你需要**存储多个元素**时（如最近N笔交易）
- 当你需要**保持元素顺序**时（List有序）
- 当你需要**存储历史记录**时（交易历史等）
- 当你需要理解**不同类型的状态**时（ListState vs ValueState）
- 当你需要**批量处理元素**时（获取所有元素处理）


