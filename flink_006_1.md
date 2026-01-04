# 并行度：基本概念和设置

## 核心概念

**并行度（Parallelism）** 是 Flink 作业中**同时执行的任务数量**。它决定了 Flink 如何将数据流分割成多个子任务并行处理。

### 类比理解

这就像：
- **餐厅服务员**：并行度 = 4 意味着有 4 个服务员同时服务顾客
- **工厂生产线**：并行度 = 8 意味着有 8 条生产线同时生产
- **多线程程序**：并行度决定了有多少个线程同时处理数据

### 核心特点

1. **提高吞吐量**：多个任务并行处理，可以同时处理更多数据
2. **充分利用资源**：在多核 CPU 或多节点集群上，并行度可以充分利用计算资源
3. **水平扩展**：通过增加并行度，可以处理更大的数据量

## 源码位置

并行度相关的配置在：
- `StreamExecutionEnvironment.setParallelism()` - 设置作业默认并行度
- `DataStream.setParallelism()` - 设置算子并行度
- `CoreOptions.DEFAULT_PARALLELISM` - 配置级别的默认并行度

## 并行度的设置级别

Flink 支持**三个级别**的并行度设置，优先级从高到低：

### 1. 算子级别（最高优先级）

为**单个算子**设置并行度：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(4);  // 作业默认并行度

DataStream<String> stream = env.fromElements("hello", "world");

// 为 map 算子单独设置并行度为 8
stream.map(s -> s.toUpperCase())
      .setParallelism(8)  // 算子级别并行度
      .print();
```

**使用场景**：
- 某个算子计算密集，需要更多并行度
- 某个算子需要特殊资源（如 GPU）
- 调整特定算子的性能瓶颈

### 2. 作业级别（中等优先级）

为**整个作业**设置默认并行度：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 设置作业默认并行度为 4
env.setParallelism(4);

// 所有算子默认使用并行度 4
DataStream<String> stream = env.fromElements("hello", "world");
stream.map(s -> s.toUpperCase())  // 使用并行度 4
      .filter(s -> s.length() > 5)  // 使用并行度 4
      .print();  // 使用并行度 4
```

**使用场景**：
- 为整个作业设置统一的并行度
- 大多数算子的计算量相似时

### 3. 配置级别（最低优先级）

通过**配置文件**或**命令行参数**设置：

```java
// 方式1：通过 Configuration 设置
Configuration config = new Configuration();
config.set(CoreOptions.DEFAULT_PARALLELISM, 4);
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(config);

// 方式2：通过 flink-conf.yaml 设置
// parallelism.default: 4

// 方式3：通过命令行提交时设置
// flink run -p 4 your-job.jar
```

**使用场景**：
- 集群级别的默认配置
- 通过命令行动态调整并行度（不需要修改代码）

## 如何选择合适的并行度？

### 1. 本地环境

```java
// ✅ 推荐：使用 CPU 核心数
int cores = Runtime.getRuntime().availableProcessors();
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(cores);

// ✅ 或者：根据测试需要设置较小的值（如 2-4）
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(2);
```

**原则**：
- 不超过 CPU 核心数（线程切换开销）
- 通常设置为 2-8 即可满足测试需求

### 2. 集群环境

```java
// 根据数据量和资源情况设置
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 方式1：作业级别设置
env.setParallelism(16);  // 假设集群有足够的 Slot

// 方式2：算子级别设置（更灵活）
DataStream<String> source = env.addSource(...);  // 并行度 16
source.map(...).setParallelism(32);  // map 算子并行度 32（计算密集）
source.filter(...).setParallelism(8);  // filter 算子并行度 8（计算简单）
```

**原则**：
- **数据源并行度**：通常等于 Kafka Partition 数量或其他数据源的分区数
- **计算密集型算子**：可以设置更高的并行度（如 map、flatMap）
- **I/O 密集型算子**：根据外部系统（如数据库）的连接数限制设置
- **总并行度**：不超过集群的 Slot 总数

### 3. 经验公式

```
推荐并行度 = 数据源分区数 × 2
```

例如：
- Kafka Topic 有 8 个 Partition → 并行度设置为 16
- 但最终需要根据实际资源调整

## 并行度与资源的关系

### 本地环境

```java
// 并行度 = 4 意味着：
// - 创建 4 个线程
// - 共享同一个 JVM 的内存和 CPU
// - 受限于单机资源
```

### 集群环境

```java
// 并行度 = 16 意味着：
// - 需要 16 个 TaskManager Slot
// - 每个 Slot 占用一定的 CPU 和内存
// - 可以分布在多个节点上
```

**资源计算**：
```
总资源需求 = 并行度 × 每个 Slot 的资源需求
```

例如：
- 并行度 = 16
- 每个 Slot 需要 1 CPU Core 和 2GB 内存
- 总需求 = 16 CPU Cores + 32GB 内存

## 并行度的限制

1. **不能超过 Slot 数量**：集群中可用的 TaskManager Slot 总数
2. **数据源限制**：不能超过数据源的分区数（如 Kafka Partition 数）
3. **资源限制**：受限于集群的 CPU 和内存资源

## 查看并行度

```java
// 获取作业的默认并行度
int parallelism = env.getParallelism();
System.out.println("默认并行度: " + parallelism);

// 获取算子的并行度（需要从 Transformation 获取）
// 通常在作业执行时，Flink Web UI 会显示每个算子的并行度
```

## 最小可用例子

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 设置作业默认并行度
env.setParallelism(4);

// 创建数据流
DataStream<String> stream = env.fromElements("hello", "world", "flink");

// 处理数据（使用默认并行度 4）
stream.map(s -> s.toUpperCase())
      .print();

// 为特定算子设置不同的并行度
stream.filter(s -> s.length() > 5)
      .setParallelism(8)  // filter 算子使用并行度 8
      .print();

env.execute("Parallelism Example");
```

## 常见错误

### 错误1：并行度设置超过数据源分区数

```java
// ❌ 错误：并行度超过 Kafka Partition 数量
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(16);  // 设置并行度为 16

// 但 Kafka Topic 只有 8 个 Partition
DataStream<String> stream = env.addSource(new FlinkKafkaConsumer<>(...));
// 结果：只有 8 个 Source 任务会工作，其他 8 个空闲

// ✅ 正确：并行度不超过数据源分区数
env.setParallelism(8);  // 等于或小于 Partition 数量
```

### 错误2：并行度设置超过集群 Slot 数量

```java
// ❌ 错误：并行度超过集群可用 Slot 数
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(100);  // 设置并行度为 100

// 但集群只有 20 个 Slot
// 结果：作业提交失败或部分任务无法执行

// ✅ 正确：先查询集群 Slot 数量，然后设置合理的并行度
// 通过 Flink Web UI 或命令行查看可用 Slot 数
env.setParallelism(16);  // 不超过可用 Slot 数
```

### 错误3：本地环境设置过高的并行度

```java
// ❌ 错误：设置过高的并行度（超过CPU核心数）
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(100);
// 本地环境使用线程模拟，过高的并行度没有意义

// ✅ 正确：根据CPU核心数设置合理的并行度
int cores = Runtime.getRuntime().availableProcessors();
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(cores);
```

## 什么时候你需要想到这个？

- 当你需要**配置作业的并行度**时
- 当你需要**设置算子级别的并行度**时
- 当你需要**理解并行度与资源的关系**时
- 当你需要**选择合适的并行度**时
- 当你遇到**性能瓶颈**时（可能需要调整并行度）
