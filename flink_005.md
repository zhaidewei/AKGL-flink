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
    return getExecutionEnvironment(new Configuration());
}

public static StreamExecutionEnvironment getExecutionEnvironment(Configuration config) {
    // 1. 首先检查 ThreadLocal 中的工厂（用于测试环境等）
    // 这允许测试框架或特定上下文设置自定义环境
    if (threadLocalContextEnvironmentFactory != null) {
        return threadLocalContextEnvironmentFactory.createExecutionEnvironment(config);
    }

    // 2. 检查全局上下文工厂（用于集群提交等场景）
    // 当通过命令行提交作业时，Flink 会设置这个工厂
    if (contextEnvironmentFactory != null) {
        return contextEnvironmentFactory.createExecutionEnvironment(config);
    }

    // 3. 默认情况：创建本地执行环境
    // 这适用于在 IDE 中直接运行或 standalone 模式
    return createLocalEnvironment(config);
}
```

**关键理解**：
- Flink 使用**工厂模式**来选择执行环境，而不是直接"检查是否在集群中"
- `threadLocalContextEnvironmentFactory`：用于测试框架等场景，优先级最高
- `contextEnvironmentFactory`：用于集群提交等场景，通过命令行提交时会设置
- 默认情况：创建本地环境，适合开发和测试
- 在集群中运行时，通常是通过 `flink run` 命令提交，此时会设置相应的工厂

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

## 常见错误

### 错误1：在生产环境使用 createLocalEnvironment()

```java
// ❌ 错误：在生产环境明确使用本地环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
// 这会在本地JVM运行，不适合生产环境

// ✅ 正确：使用getExecutionEnvironment()，自动选择
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 在集群中会自动使用集群环境
```

### 错误2：在本地开发使用 createRemoteEnvironment()

```java
// ❌ 错误：在IDE中开发时使用远程环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "remote-host", 8081, "job.jar"
);
// 需要连接远程集群，开发不方便

// ✅ 正确：使用getExecutionEnvironment()或createLocalEnvironment()
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 在IDE中自动使用本地环境
```

### 错误3：硬编码环境类型，代码不可移植

```java
// ❌ 错误：硬编码使用本地环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(4);
// 代码只能在本地运行，不能部署到集群

// ✅ 正确：使用getExecutionEnvironment()，代码可移植
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(4);  // 配置可以在任何环境使用
```

### 错误4：混淆 getExecutionEnvironment() 和 createLocalEnvironment()

```java
// ❌ 错误：认为getExecutionEnvironment()总是返回本地环境
// 实际上它会根据上下文自动选择

// ✅ 正确：理解getExecutionEnvironment()的智能选择机制
// - 在IDE中运行：自动使用本地环境
// - 通过flink run提交：自动使用集群环境
```

## 什么时候你需要想到这个？

- 当你**开始写 Flink 程序**时（第一步选择创建方式）
- 当你需要**在本地测试**时（使用 createLocalEnvironment）
- 当你需要**部署到集群**时（理解不同环境的区别）
- 当你遇到"环境不匹配"的问题时（可能用错了创建方式）
- 当你需要**编写可移植的 Flink 代码**时

