# 创建StreamExecutionEnvironment：三种方式

## 核心概念

Flink 提供了**三种方式**来创建 StreamExecutionEnvironment，根据你的使用场景选择合适的方式。

### 三种创建方式

1. **`getExecutionEnvironment()`** - 智能选择（推荐）
2. **`createLocalEnvironment()`** - 本地执行环境
3. **`createRemoteEnvironment()`** - 远程集群环境

## 源码位置

这些方法定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/environment/StreamExecutionEnvironment.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/environment/StreamExecutionEnvironment.java)

### 1. getExecutionEnvironment() - 智能选择（推荐）

**这是最常用的方式**，Flink 会根据上下文自动选择：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

**工作原理（伪代码）**：
```java
public static StreamExecutionEnvironment getExecutionEnvironment() {
    // 1. 检查是否有预设的工厂（测试环境等）
    if (threadLocalContextEnvironmentFactory != null) {
        return factory.createExecutionEnvironment();
    }

    // 2. 检查是否在集群中运行
    if (在集群中) {
        return 集群环境;
    }

    // 3. 默认：创建本地环境
    return createLocalEnvironment();
}
```

**使用场景**：
- **开发和测试**：自动使用本地环境
- **生产部署**：在集群中自动使用集群环境
- **代码可移植**：同一份代码可以在不同环境运行

### 2. createLocalEnvironment() - 本地执行环境

**明确指定使用本地环境**：

```java
// 使用默认并行度（CPU核心数）
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// 指定并行度
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(4);

// 指定配置
Configuration config = new Configuration();
config.set(CoreOptions.DEFAULT_PARALLELISM, 4);
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(config);
```

**使用场景**：
- **本地调试**：明确要在本地运行
- **单元测试**：测试环境需要确定性
- **IDE 中运行**：确保在本地 JVM 中执行

### 3. createRemoteEnvironment() - 远程集群环境

**连接到远程 Flink 集群**：

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "flink-cluster-host",  // 集群地址
    8081,                  // JobManager端口
    "path/to/job.jar"      // 作业JAR文件
);
```

**使用场景**：
- **连接到已有集群**：已有 Flink 集群，需要提交作业
- **传统部署方式**：不使用新的 Application Mode

**注意**：这种方式在现代 Flink 中较少使用，推荐使用 Application Mode。

## 最小可用例子

```java
// 方式1：智能选择（推荐，适合大多数场景）
StreamExecutionEnvironment env1 = StreamExecutionEnvironment.getExecutionEnvironment();

// 方式2：明确本地环境（适合本地开发）
StreamExecutionEnvironment env2 = StreamExecutionEnvironment.createLocalEnvironment(4);

// 方式3：远程集群（适合连接到已有集群）
StreamExecutionEnvironment env3 = StreamExecutionEnvironment.createRemoteEnvironment(
    "localhost", 8081, "my-job.jar"
);

// 使用方式完全相同
env1.fromElements("hello").print();
env2.fromElements("world").print();
env3.fromElements("flink").print();
```

## 选择建议

- **99% 的情况**：使用 `getExecutionEnvironment()`
- **本地调试**：使用 `createLocalEnvironment()`
- **连接到集群**：使用 `createRemoteEnvironment()`（或更好的方式：Application Mode）

## 什么时候你需要想到这个？

- 当你**开始写 Flink 程序**时（第一步选择创建方式）
- 当你需要**在本地测试**时（使用 createLocalEnvironment）
- 当你需要**部署到集群**时（理解不同环境的区别）
- 当你遇到"环境不匹配"的问题时（可能用错了创建方式）
- 当你需要**编写可移植的 Flink 代码**时

