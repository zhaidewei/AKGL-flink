# 并行度如何影响执行

## 核心概念

并行度不仅决定了任务数量，还**影响数据的分区方式、执行模式和元素顺序**。理解这些影响对于正确使用 Flink 至关重要。

## 数据分区

并行度决定了数据如何被分割：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(4);  // 并行度为 4

DataStream<String> stream = env.fromElements("a", "b", "c", "d", "e", "f");

// 数据会被分配到 4 个并行任务中：
// Task 0: "a", "e"
// Task 1: "b", "f"
// Task 2: "c"
// Task 3: "d"
```

## 任务执行

每个并行度对应一个**独立的线程**（本地环境）或**TaskManager Slot**（集群环境）：

```java
// 并行度为 4 意味着：
// - 本地环境：创建 4 个线程并行执行
// - 集群环境：需要 4 个 TaskManager Slot
```

## 数据流在并行算子间的流动

**重要理解**：并行度会将数据流分割，但**没有"合并"操作**，而是通过**数据分区策略（Partitioning Strategy）**决定数据如何流向下游算子。

### 逻辑视图 vs 物理执行

**逻辑视图**（代码层面）：
```java
DataStream<String> stream = env.fromElements("a", "b", "c", "d", "e", "f");
stream.map(s -> s.toUpperCase())
      .filter(s -> s.length() > 3)
      .print();
```
- 逻辑上是一个**单维的数据流**
- 数据按顺序流动：Source → map → filter → print

**物理执行**（运行时）：
```
Source (并行度 4):
  Task 0: ["a", "e"] → map → filter → print
  Task 1: ["b", "f"] → map → filter → print
  Task 2: ["c"]      → map → filter → print
  Task 3: ["d"]      → map → filter → print
```
- 物理上是**多个并行任务**同时执行
- 每个任务独立处理一部分数据

### 数据如何流向下游算子？

**关键点**：数据不是"合并"，而是通过**分区策略**分发到下游的并行任务。

**示例1：Forward 分区（前向连接）**
```java
stream.map(s -> s.toUpperCase())  // 并行度 4
      .filter(s -> s.length() > 3)  // 并行度 4
```
- 如果两个算子并行度相同且可以链式连接，数据**直接前向传递**（不经过网络）
- 如果链式连接，数据在**同一个线程内传递**（最快）

**示例2：Rebalance 分区（重新平衡）**
```java
stream.map(s -> s.toUpperCase())  // 并行度 4
      .setParallelism(8)  // 改变并行度
      .filter(s -> s.length() > 3)  // 并行度 8
```
- 上游 4 个任务 → 下游 8 个任务
- 数据通过**网络传输**，根据轮询策略分发到下游 8 个任务
- **没有合并**，而是**重新分发**

**示例3：Sink 输出（如 print()）**
```java
stream.map(s -> s.toUpperCase())  // 并行度 4
      .print();  // 并行度 4
```
- 每个并行任务**独立输出**到控制台
- 输出顺序可能乱序（因为不同任务完成时间不同）
- **没有合并输出**，而是多个任务并行输出

### 数据分区策略

Flink 提供多种数据分区策略：

1. **Forward**：前向连接，数据直接传递（相同并行度，链式连接）
2. **Rebalance**：轮询分发，数据均匀分配到下游任务
3. **Rescale**：局部重新平衡，只在部分任务间重新分发
4. **Broadcast**：广播，数据复制到所有下游任务
5. **KeyBy**：按 key 哈希分区，相同 key 的数据到同一任务
6. **Shuffle**：随机分发

#### Rebalance、Rescale 和 Shuffle 的区别

这三种策略都用于**重新分发数据**，但分发方式和网络开销不同：

##### 1. Rebalance（轮询分发）

**工作原理**：
- 使用**轮询（Round-Robin）**方式分发数据
- 每个上游任务的数据**依次轮询**发送到所有下游任务
- 保证数据**均匀分布**

**示例**：
```java
stream.map(s -> s.toUpperCase())  // 并行度 4
      .rebalance()  // 显式指定 Rebalance
      .filter(s -> s.length() > 3)  // 并行度 8
```

**数据流**：
```
上游 4 个任务 → 下游 8 个任务
Task 0 → [0, 1, 2, 3, 4, 5, 6, 7] (轮询)
Task 1 → [0, 1, 2, 3, 4, 5, 6, 7] (轮询)
Task 2 → [0, 1, 2, 3, 4, 5, 6, 7] (轮询)
Task 3 → [0, 1, 2, 3, 4, 5, 6, 7] (轮询)
```

**特点**：
- ✅ **均匀分布**：数据均匀分配到所有下游任务
- ⚠️ **网络开销**：每个上游任务需要与所有下游任务通信（全连接）
- ✅ **负载均衡**：适合需要均匀负载的场景

##### 2. Rescale（局部重新平衡）

**工作原理**：
- **局部重新平衡**，只在**部分任务**间重新分发
- 上游任务只与**部分下游任务**通信（不是全部）
- 减少网络连接数

**示例**：
```java
stream.map(s -> s.toUpperCase())  // 并行度 4
      .rescale()  // 显式指定 Rescale
      .filter(s -> s.length() > 3)  // 并行度 8
```

**数据流**：
```
上游 4 个任务 → 下游 8 个任务
Task 0 → [0, 1] (只发送到部分下游任务)
Task 1 → [2, 3] (只发送到部分下游任务)
Task 2 → [4, 5] (只发送到部分下游任务)
Task 3 → [6, 7] (只发送到部分下游任务)
```

**特点**：
- ✅ **减少网络开销**：每个上游任务只与部分下游任务通信
- ⚠️ **分布可能不均匀**：如果上游任务数据量差异大，下游负载可能不均
- ✅ **适合同节点**：适合上游和下游任务在同一 TaskManager 的场景

##### 3. Shuffle（随机分发）

**工作原理**：
- 使用**随机算法**分发数据
- 每个元素**随机**发送到下游任务
- 不保证均匀分布（长期来看可能均匀）

**示例**：
```java
stream.map(s -> s.toUpperCase())  // 并行度 4
      .shuffle()  // 显式指定 Shuffle
      .filter(s -> s.length() > 3)  // 并行度 8
```

**数据流**：
```
上游 4 个任务 → 下游 8 个任务
Task 0 → [随机下游任务]
Task 1 → [随机下游任务]
Task 2 → [随机下游任务]
Task 3 → [随机下游任务]
```

**特点**：
- ⚠️ **不保证均匀**：短期可能不均匀，长期可能均匀
- ⚠️ **网络开销**：每个上游任务需要与所有下游任务通信（全连接）
- ⚠️ **不可预测**：分发结果不可预测
- ⚠️ **较少使用**：通常不推荐使用，Rebalance 更常用

#### 三种策略的对比

| 策略 | 分发方式 | 网络连接 | 分布均匀性 | 使用场景 |
|------|---------|---------|-----------|---------|
| **Rebalance** | 轮询 | 全连接（每个上游→所有下游） | ✅ 保证均匀 | 需要均匀负载，跨节点 |
| **Rescale** | 局部轮询 | 部分连接（每个上游→部分下游） | ⚠️ 可能不均匀 | 同节点优化，减少网络 |
| **Shuffle** | 随机 | 全连接（每个上游→所有下游） | ⚠️ 不保证均匀 | 较少使用，不推荐 |

#### 何时使用哪种策略？

**使用 Rebalance**：
- 需要**均匀分布**数据
- 上游和下游任务可能在不同节点
- 需要**负载均衡**

**使用 Rescale**：
- 上游和下游任务在**同一 TaskManager**
- 需要**减少网络开销**
- 可以接受**轻微的不均匀**

**使用 Shuffle**：
- ⚠️ **不推荐使用**
- 如果需要随机分发，通常 Rebalance 更好

### 为什么没有"合并"？

**原因**：
1. **流式处理特性**：Flink 是流式处理，数据是**无界的**，无法等待所有数据到达后再合并
2. **性能考虑**：合并需要等待和缓冲，会增加延迟
3. **分布式设计**：Flink 是分布式系统，数据分布在多个节点，合并需要网络传输
4. **逻辑抽象**：DataStream 在逻辑上是单维的，物理执行是并行的，这是 Flink 的抽象

**类比理解**：
- **不是**：4 条河流汇合成 1 条河流（合并）
- **而是**：4 条河流各自流向 4 个下游处理站（分区）

## 元素顺序的影响

**重要**：并行度会影响元素的顺序，但影响程度取决于操作类型：

### 1. 无 keyBy 的操作（map、filter、flatMap 等）

**在同一个并行任务内，元素顺序是保持的**，但**不同任务之间的顺序不保证**：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(4);

DataStream<String> stream = env.fromElements("a", "b", "c", "d", "e", "f");

// 数据分配到 4 个任务：
// Task 0: ["a", "e"]  → 处理顺序：a, e（顺序保持）
// Task 1: ["b", "f"]  → 处理顺序：b, f（顺序保持）
// Task 2: ["c"]       → 处理顺序：c（顺序保持）
// Task 3: ["d"]       → 处理顺序：d（顺序保持）

stream.map(s -> s.toUpperCase())
      .print();

// 输出顺序可能是：
// C (Task 2 先完成)
// A (Task 0 输出)
// D (Task 3 输出)
// B (Task 1 输出)
// E (Task 0 继续输出)
// F (Task 1 继续输出)
```

**关键点**：
- ✅ **同一任务内的顺序保持**：Task 0 中 "a" 一定在 "e" 之前
- ❌ **不同任务间的顺序不保证**：Task 2 的 "c" 可能比 Task 0 的 "a" 先输出
- ⚠️ **输出顺序可能乱序**：因为不同任务处理速度不同

### 2. keyBy 操作

**相同 key 的数据在同一个任务内，顺序是保持的**，但**不同 key 之间的顺序不保证**：

```java
DataStream<Trade> trades = env.fromElements(
    new Trade("BTC", 100),  // 事件时间: 12:00:01
    new Trade("ETH", 200),  // 事件时间: 12:00:02
    new Trade("BTC", 150),  // 事件时间: 12:00:03
    new Trade("ETH", 250)   // 事件时间: 12:00:04
);

KeyedStream<Trade, String> keyed = trades.keyBy(t -> t.getSymbol());

// keyBy 后数据分区：
// Key="BTC" (Task 0): [BTC@12:00:01, BTC@12:00:03]  → 顺序保持
// Key="ETH" (Task 1): [ETH@12:00:02, ETH@12:00:04]  → 顺序保持

keyed.map(t -> processTrade(t))
     .print();

// 输出顺序可能是：
// ETH@12:00:02 (Task 1 先完成)
// BTC@12:00:01 (Task 0 输出)
// BTC@12:00:03 (Task 0 继续，顺序保持)
// ETH@12:00:04 (Task 1 继续，顺序保持)
```

**关键点**：
- ✅ **同一 key 内的顺序保持**：BTC 的两个交易按时间顺序处理
- ❌ **不同 key 间的顺序不保证**：ETH 的交易可能比 BTC 先输出
- ✅ **状态一致性**：相同 key 的数据在同一任务，状态操作是安全的

### 3. 窗口操作

**窗口内的元素顺序取决于输入顺序**，但**窗口之间的顺序可能不保证**：

```java
keyed.window(TumblingEventTimeWindows.of(Time.seconds(10)))
     .process(new ProcessWindowFunction<...>() {
         @Override
         public void process(String key, Context ctx,
                           Iterable<Trade> elements, Collector<...> out) {
             // elements 中的顺序取决于输入顺序
             // 但不同窗口的输出顺序可能不保证
         }
     });
```

**关键点**：
- ✅ **窗口内元素顺序保持**：基于输入顺序
- ⚠️ **窗口输出顺序**：可能因为 Watermark 到达时间不同而乱序

### 4. 如何保证全局顺序？

如果需要保证全局顺序，有以下方法：

**方法1：设置并行度为 1**（不推荐，性能差）：
```java
env.setParallelism(1);  // 所有数据在单线程处理，保证顺序
```

**方法2：使用事件时间和 Watermark**（推荐）：
```java
// 使用事件时间，通过 Watermark 保证时间顺序
stream.assignTimestampsAndWatermarks(
    WatermarkStrategy.<Trade>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner((trade, timestamp) -> trade.getEventTime())
)
.window(TumblingEventTimeWindows.of(Time.seconds(10)))
.aggregate(...);
```

**方法3：使用 ProcessFunction 手动排序**：
```java
// 在 ProcessFunction 中维护有序队列
public class OrderedProcessFunction extends KeyedProcessFunction<String, Trade, Trade> {
    private ValueState<PriorityQueue<Trade>> queue;
    // 手动维护顺序
}
```

## 常见错误

### 错误：所有算子使用相同的并行度（不考虑计算复杂度）

```java
// ❌ 错误：所有算子使用相同的并行度，没有考虑计算复杂度
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(4);

DataStream<String> stream = env.addSource(...);
stream.map(s -> s.toUpperCase())  // 简单操作，并行度 4 可能过多
      .flatMap(s -> complexComputation(s))  // 复杂计算，并行度 4 可能不足
      .print();

// ✅ 正确：根据算子计算复杂度设置不同的并行度
// 注意：设置不同并行度会打破算子链，这是预期行为
stream.map(s -> s.toUpperCase())
      .setParallelism(2)  // 简单操作，较少并行度
      .flatMap(s -> complexComputation(s))
      .setParallelism(8)  // 复杂计算，更多并行度（会打破链）
      .print();
```

**说明**：
- 设置不同并行度会**自动打破算子链**，这是 Flink 的预期行为
- 不同并行度的算子之间需要**网络传输**（重新分区），会有一定开销
- 但如果性能提升（通过调整并行度）大于链式连接的好处，这是值得的

## 什么时候你需要想到这个？

- 当你需要**理解并行度如何影响作业执行**时
- 当你需要**理解元素顺序**时（并行度会影响顺序）
- 当你需要**理解数据流在并行算子间的流动**时
- 当你需要**理解 keyBy 对并行度的影响**时
- 当你需要**保证数据顺序**时

