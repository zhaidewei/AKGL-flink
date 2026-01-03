# 本地执行环境：开发和测试

## 核心概念

**本地执行环境（LocalStreamEnvironment）** 在**当前 JVM 进程**中运行 Flink 作业，使用多线程模拟分布式执行。这是开发和测试时最常用的环境。

### 类比理解

这就像：
- **本地 Spark**：`local[*]` 模式，在本地 JVM 中运行
- **单机数据库**：不需要网络连接，直接在本地运行
- **本地测试服务器**：开发时在本地运行，不需要部署到服务器

### 核心特点

1. **在本地 JVM 中运行**：不需要 Flink 集群
2. **多线程执行**：使用线程模拟并行任务
3. **适合开发测试**：快速迭代，不需要复杂环境

## 源码位置

LocalStreamEnvironment 的定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/environment/LocalStreamEnvironment.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/environment/LocalStreamEnvironment.java)

### 创建方式

```java
// 方式1：使用默认并行度（CPU核心数）
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// 方式2：指定并行度
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(4);

// 方式3：使用配置
Configuration config = new Configuration();
config.set(CoreOptions.DEFAULT_PARALLELISM, 4);
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(config);
```

### 默认并行度

本地环境的默认并行度是 **CPU 核心数**：
```java
private static int defaultLocalParallelism = Runtime.getRuntime().availableProcessors();
```

如果你的机器有 8 核，默认并行度就是 8。

## 最小可用例子

```java
// 创建本地环境，并行度为2
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(2);

// 创建数据流
DataStream<String> words = env.fromElements("hello", "world", "flink");

// 处理数据
words.map(s -> s.toUpperCase())
     .print();

// 执行（在本地JVM中运行）
env.execute("Local Test");
```

### 执行过程

1. **在本地 JVM 中启动**：不需要连接集群
2. **创建线程池**：根据并行度创建多个线程
3. **模拟分布式执行**：每个线程处理一部分数据
4. **输出到控制台**：结果直接打印到控制台

## 与集群环境的区别

| 特性 | 本地环境 | 集群环境 |
|------|---------|---------|
| 运行位置 | 本地 JVM | 集群节点 |
| 资源隔离 | 无（共享JVM） | 有（独立进程） |
| 容错性 | 低（JVM崩溃=作业失败） | 高（节点失败可恢复） |
| 适用场景 | 开发、测试 | 生产 |

## 使用场景

1. **本地开发**：在 IDE 中快速测试代码
2. **单元测试**：编写测试用例
3. **调试**：设置断点，单步调试
4. **学习 Flink**：不需要搭建集群就能学习

## 什么时候你需要想到这个？

- 当你在 **IDE 中开发 Flink 程序**时
- 当你需要**快速测试代码**时（不需要部署）
- 当你编写**单元测试**时
- 当你需要**调试 Flink 作业**时（设置断点）
- 当你学习 Flink 但**没有集群环境**时

