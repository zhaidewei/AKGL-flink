# JobManager：集群的协调者

## 核心概念

**JobManager** 是 Flink 集群的**主节点（Master）**，负责**协调和管理**整个集群。它不执行实际的数据处理任务，而是负责作业调度、资源分配和故障恢复。

### 类比理解

这就像：
- **公司的 CEO**：制定战略、分配任务、协调各部门
- **餐厅的经理**：分配工作、协调厨房和服务员
- **Spark 的 Driver**：如果你用过 Spark，JobManager 在 Flink 中扮演类似的角色

### 核心特点

1. **单一主节点**：通常集群中只有一个 JobManager（可配置高可用）
2. **不执行任务**：JobManager 不处理数据，只负责协调
3. **全局视图**：拥有整个集群和所有作业的全局视图
4. **关键组件**：包含多个关键组件（Dispatcher、ResourceManager、JobMaster 等）

## 源码位置

JobManager 的主要组件在：
- `flink-runtime/src/main/java/org/apache/flink/runtime/jobmanager/`
- `flink-runtime/src/main/java/org/apache/flink/runtime/dispatcher/`
- `flink-runtime/src/main/java/org/apache/flink/runtime/resourcemanager/`

## JobManager 的核心职责

### 1. 作业调度（Job Scheduling）

**功能**：
- 接收作业提交请求
- 将作业的 Transformation 图转换为执行图（ExecutionGraph）
- 将任务分配到 TaskManager

**示例**：
```java
// 当你提交作业时
env.execute("My Job");

// JobManager 会：
// 1. 接收作业
// 2. 构建执行图
// 3. 分配任务到 TaskManager
```

### 2. 资源管理（Resource Management）

**功能**：
- 跟踪集群中可用的 Slot 资源
- 根据作业需求分配 Slot
- 管理 TaskManager 的注册和注销

**示例**：
```
作业需要 16 个并行度
  ↓
JobManager 查找可用的 Slot
  ↓
找到 16 个 Slot（分布在多个 TaskManager）
  ↓
分配任务到这些 Slot
```

### 3. 检查点协调（Checkpoint Coordination）

**功能**：
- 触发检查点的创建
- 协调所有 TaskManager 的检查点操作
- 管理检查点的元数据

**示例**：
```
JobManager 触发检查点
  ↓
通知所有 TaskManager 创建检查点
  ↓
收集检查点完成状态
  ↓
保存检查点元数据
```

### 4. 故障恢复（Failure Recovery）

**功能**：
- 监控 TaskManager 的健康状态
- 检测任务失败
- 从检查点恢复作业
- 重新调度失败的任务

**示例**：
```
TaskManager 失败
  ↓
JobManager 检测到失败
  ↓
从最近的检查点恢复
  ↓
重新分配任务到其他 TaskManager
```

## JobManager 的关键组件

### 1. Dispatcher（调度器）

**职责**：
- 接收作业提交
- 为每个作业创建 JobMaster
- 提供 Web UI

### 2. ResourceManager（资源管理器）

**职责**：
- 管理 TaskManager 资源
- 分配和释放 Slot
- 与外部资源管理器（如 YARN）交互

### 3. JobMaster（作业主控）

**职责**：
- 管理单个作业的生命周期
- 调度作业的任务
- 协调检查点

## 最小可用例子

### 启动 JobManager

```bash
# Standalone 模式
./bin/jobmanager.sh start

# 或使用 start-cluster.sh（同时启动 JobManager 和 TaskManager）
./bin/start-cluster.sh
```

### 查看 JobManager 状态

```bash
# 访问 Web UI
http://localhost:8081

# 或通过命令行
./bin/flink list
```

### 提交作业到 JobManager

```java
// 代码中
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.fromElements("hello", "world")
   .map(s -> s.toUpperCase())
   .print();

env.execute("My Job");  // 提交到 JobManager
```

```bash
# 命令行提交
./bin/flink run -c com.example.MyJob /path/to/my-job.jar
```

## JobManager 高可用（HA）

### 为什么需要高可用？

如果 JobManager 失败，整个集群无法提交新作业或恢复失败作业。高可用配置可以：

1. **自动故障转移**：主 JobManager 失败时，备用 JobManager 自动接管
2. **作业恢复**：从检查点恢复正在运行的作业
3. **服务连续性**：集群服务不中断

### 高可用配置

```yaml
# flink-conf.yaml
high-availability: zookeeper
high-availability.zookeeper.quorum: zk1:2181,zk2:2181,zk3:2181
high-availability.storageDir: hdfs:///flink/ha
```

## 常见错误

### 错误1：认为 JobManager 也执行任务

```java
// ❌ 误解：认为 JobManager 也处理数据
// JobManager 只负责协调，不执行任务

// ✅ 正确：理解职责分工
// - JobManager：协调和管理
// - TaskManager：执行任务
```

### 错误2：在代码中直接连接 JobManager

```java
// ❌ 错误：在代码中硬编码 JobManager 地址
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "jobmanager-host", 8081, "job.jar"
);

// ✅ 正确：使用 getExecutionEnvironment()，通过配置指定
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// JobManager 地址通过 flink run 命令或配置文件指定
```

### 错误3：忽略 JobManager 高可用配置

```java
// ❌ 错误：生产环境不配置高可用
// 如果 JobManager 失败，整个集群无法工作

// ✅ 正确：生产环境配置高可用
// 在 flink-conf.yaml 中配置高可用
```

## 什么时候你需要想到这个？

- 当你需要**理解作业如何被调度**时
- 当你需要**配置集群资源**时
- 当你需要**理解故障恢复机制**时
- 当你需要**配置 JobManager 高可用**时
- 当你需要**查看作业状态**时（通过 Web UI）
- 当你需要**理解检查点如何工作**时

