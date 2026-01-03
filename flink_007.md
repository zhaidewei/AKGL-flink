# 集群执行环境：生产部署

## 核心概念

**集群执行环境（RemoteStreamEnvironment）** 将 Flink 作业提交到**远程 Flink 集群**执行。这是生产环境的标准部署方式。

### 类比理解

这就像：
- **Spark 集群模式**：提交作业到 Spark 集群
- **Kubernetes 部署**：将应用部署到 K8s 集群
- **云服务**：将应用部署到云端服务器

### 核心特点

1. **提交到远程集群**：作业在集群节点上运行
2. **分布式执行**：多个节点并行处理
3. **高可用性**：节点失败可以恢复
4. **资源隔离**：每个任务在独立进程中运行

## 源码位置

RemoteStreamEnvironment 的定义在：
[flink-runtime/src/main/java/org/apache/flink/streaming/api/environment/RemoteStreamEnvironment.java](https://github.com/apache/flink/blob/master/flink-runtime/src/main/java/org/apache/flink/streaming/api/environment/RemoteStreamEnvironment.java)

### 创建方式（传统方式）

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "flink-cluster-host",  // JobManager 地址
    8081,                  // JobManager 端口
    "path/to/job.jar"      // 作业 JAR 文件路径
);
```

### 现代部署方式（推荐）

**Application Mode**（推荐，不需要 createRemoteEnvironment）：

```bash
# 打包作业
mvn clean package

# 提交到集群（Flink 1.11+）
./bin/flink run \
  --target local \
  ./target/my-job.jar
```

作业代码中仍然使用：
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
```

Flink 会根据提交方式自动选择环境。

## 最小可用例子

### 传统方式（不推荐）

```java
// 连接到远程集群
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "192.168.1.100",  // JobManager 地址
    8081,             // 端口
    "/path/to/my-job.jar"
);

env.fromElements("hello", "world")
   .map(s -> s.toUpperCase())
   .print();

env.execute("Remote Job");
```

### 现代方式（推荐）

```java
// 代码中不需要指定集群地址
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.fromElements("hello", "world")
   .map(s -> s.toUpperCase())
   .print();

env.execute("Remote Job");
```

然后通过命令行提交：
```bash
flink run -c com.example.MyJob /path/to/my-job.jar
```

## 集群架构

```
┌─────────────────┐
│  JobManager     │  ← 作业调度和协调
│  (Master)       │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
┌───▼───┐ ┌───▼───┐
│TaskMgr│ │TaskMgr│  ← 执行任务
│(Node1)│ │(Node2)│
└───────┘ └───────┘
```

## 与本地环境的区别

| 特性 | 本地环境 | 集群环境 |
|------|---------|---------|
| 运行位置 | 本地 JVM | 集群节点 |
| 资源 | 本地机器资源 | 集群资源（可扩展） |
| 容错 | 低 | 高（检查点、故障恢复） |
| 适用 | 开发测试 | 生产 |

## 什么时候你需要想到这个？

- 当你需要**部署 Flink 作业到生产环境**时
- 当你需要**利用集群资源**处理大规模数据时
- 当你需要**高可用性**（容错、恢复）时
- 当你使用 **Flink on Kubernetes** 或 **Flink on YARN** 时
- 当你需要**水平扩展**处理能力时

