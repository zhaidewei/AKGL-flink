# Flink 集群架构概述

## 核心概念

**Flink 集群**是由多个节点组成的分布式系统，用于执行 Flink 作业。集群采用**主从架构（Master-Worker）**，包含两种核心角色：**JobManager** 和 **TaskManager**。

### 类比理解

这就像：
- **公司的组织架构**：JobManager 是 CEO（决策和协调），TaskManager 是员工（执行工作）
- **餐厅运营**：JobManager 是经理（分配任务），TaskManager 是厨师（制作菜品）
- **Spark 集群**：JobManager 类似 Spark Driver，TaskManager 类似 Spark Executor

### 核心特点

1. **主从架构**：一个 JobManager（主节点）协调多个 TaskManager（工作节点）
2. **分布式执行**：任务分布在多个 TaskManager 上并行执行
3. **高可用性**：支持 JobManager 高可用（HA）配置
4. **资源管理**：通过 Slot 机制管理资源分配

## 集群架构图

```
┌─────────────────────────────────────────┐
│         Flink 集群                        │
│                                          │
│  ┌─────────────────┐                    │
│  │  JobManager     │  ← 主节点（Master）│
│  │  (协调者)        │                    │
│  └────────┬─────────┘                    │
│           │                              │
│    ┌──────┴──────┐                      │
│    │             │                       │
│ ┌──▼──┐      ┌──▼──┐                   │
│ │TaskM│      │TaskM│  ← 工作节点        │
│ │gr 1 │      │gr 2 │    (Worker)        │
│ └─────┘      └─────┘                   │
│                                          │
│  (可以有更多 TaskManager)                │
└─────────────────────────────────────────┘
```

## 两种核心角色

### 1. JobManager（主节点）

**职责**：
- **作业调度**：接收作业提交，构建执行图
- **资源协调**：分配任务到 TaskManager
- **故障恢复**：处理 TaskManager 失败，重新调度任务
- **检查点协调**：协调检查点的创建和恢复

**特点**：
- 通常只有一个（可以配置高可用）
- 不执行实际的数据处理任务
- 负责整个集群的协调和管理

### 2. TaskManager（工作节点）

**职责**：
- **执行任务**：运行 Flink 作业的实际任务
- **数据交换**：在任务之间传输数据
- **状态存储**：存储任务的状态数据
- **资源提供**：提供 Slot 资源

**特点**：
- 可以有多个（根据集群规模）
- 执行实际的数据处理
- 每个 TaskManager 提供多个 Slot

## 作业执行流程

```
1. 提交作业
   ↓
2. JobManager 接收作业
   ↓
3. JobManager 构建执行图
   ↓
4. JobManager 分配任务到 TaskManager
   ↓
5. TaskManager 执行任务
   ↓
6. TaskManager 向 JobManager 报告状态
```

## 最小可用例子

### 启动 Flink 集群

```bash
# 启动 JobManager
./bin/start-cluster.sh

# 或者分别启动
./bin/jobmanager.sh start
./bin/taskmanager.sh start
```

### 提交作业到集群

```java
// 代码中（不需要指定集群地址）
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.fromElements("hello", "world")
   .map(s -> s.toUpperCase())
   .print();

env.execute("Cluster Job");
```

```bash
# 通过命令行提交
./bin/flink run -c com.example.MyJob /path/to/my-job.jar
```

## 集群部署模式

Flink 支持多种部署模式：

1. **Standalone**：独立模式，Flink 自己管理集群
2. **YARN**：在 Hadoop YARN 上运行
3. **Kubernetes**：在 K8s 上运行
4. **Mesos**：在 Apache Mesos 上运行

## 与本地环境的区别

| 特性 | 本地环境 | 集群环境 |
|------|---------|---------|
| **节点数量** | 1 个（本地 JVM） | 多个（JobManager + TaskManager） |
| **资源** | 本地机器资源 | 分布式资源（可扩展） |
| **容错** | 低（JVM 崩溃=失败） | 高（节点失败可恢复） |
| **适用场景** | 开发、测试 | 生产 |

## 常见错误

### 错误1：混淆 JobManager 和 TaskManager 的职责

```java
// ❌ 误解：认为 JobManager 也执行任务
// JobManager 只负责协调，不执行任务

// ✅ 正确：理解角色分工
// - JobManager：协调和管理
// - TaskManager：执行任务
```

### 错误2：认为集群必须有多个 JobManager

```java
// ❌ 误解：认为必须有多个 JobManager
// 通常只有一个 JobManager（可以配置高可用）

// ✅ 正确：理解集群架构
// - 1 个 JobManager（主节点）
// - 多个 TaskManager（工作节点）
```

## 什么时候你需要想到这个？

- 当你需要**理解 Flink 集群如何工作**时
- 当你需要**部署 Flink 到生产环境**时
- 当你需要**理解作业如何在集群中执行**时
- 当你需要**配置集群资源**时
- 当你需要**理解 JobManager 和 TaskManager 的关系**时

