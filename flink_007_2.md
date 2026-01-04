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

## Job、JobManager 和 JobMaster 的关系

### 重要概念澄清

**关键理解**：
1. **Job（作业）**：提交一个 JAR 文件就是一个 Job
2. **JobManager**：集群中通常**只有一个**（长期运行）
3. **JobMaster**：每个 Job 有**一个** JobMaster（在 Job 的生命周期内存在）

### 关系图

```
Flink 集群（长期运行）
│
├── JobManager（1 个，长期运行）
│   ├── Dispatcher（接收作业提交）
│   ├── ResourceManager（管理资源）
│   └── 为每个 Job 创建 JobMaster
│
└── 多个 Job（可以同时运行多个）
    ├── Job 1 → JobMaster 1（管理 Job 1 的生命周期）
    ├── Job 2 → JobMaster 2（管理 Job 2 的生命周期）
    └── Job 3 → JobMaster 3（管理 Job 3 的生命周期）
```

### 详细说明

**1. Job（作业）**
```bash
# 提交一个 JAR 文件 = 提交一个 Job
./bin/flink run /path/to/my-job.jar
```
- 一个 JAR 文件 = 一个 Job
- 集群可以**同时运行多个 Job**
- 每个 Job 是独立的

**2. JobManager（集群主节点）**
```
集群启动
  ↓
启动 1 个 JobManager（长期运行）
  ↓
JobManager 等待接收作业提交
  ↓
可以管理多个 Job
```
- 集群中通常**只有 1 个 JobManager**（可配置高可用）
- JobManager **长期运行**，不随 Job 的生命周期结束
- 一个 JobManager 可以**同时管理多个 Job**

**3. JobMaster（作业主控）**
```
Job 1 提交
  ↓
JobManager 为 Job 1 创建 JobMaster 1
  ↓
JobMaster 1 管理 Job 1 的整个生命周期
  ↓
Job 1 完成
  ↓
JobMaster 1 销毁
```
- 每个 Job 有**一个** JobMaster
- JobMaster 在 Job **提交时创建**，Job **完成时销毁**
- 多个 Job 可以**同时运行**，每个都有自己的 JobMaster

### 示例场景

**场景：集群中同时运行 3 个 Job**

```
集群状态：
├── JobManager（1 个，长期运行）
│
├── Job 1（正在运行）
│   └── JobMaster 1（管理 Job 1）
│
├── Job 2（正在运行）
│   └── JobMaster 2（管理 Job 2）
│
└── Job 3（正在运行）
    └── JobMaster 3（管理 Job 3）
```

**关键点**：
- ✅ **1 个 JobManager**：管理整个集群
- ✅ **3 个 Job**：同时运行
- ✅ **3 个 JobMaster**：每个 Job 一个

## JobManager 的关键组件

### 1. Dispatcher（调度器）

**职责**：
- 接收作业提交
- 为每个作业创建 JobMaster
- 提供 Web UI

**工作流程**：
```
作业提交
  ↓
Dispatcher 接收
  ↓
为作业创建 JobMaster
  ↓
JobMaster 管理作业生命周期
```

### 2. ResourceManager（资源管理器）

**职责**：
- 管理 TaskManager 资源
- 分配和释放 Slot
- 与外部资源管理器（如 YARN）交互

**工作流程**：
```
JobMaster 请求资源
  ↓
ResourceManager 查找可用 Slot
  ↓
分配 Slot 给 JobMaster
  ↓
JobMaster 将任务分配到 Slot
```

### 3. JobMaster（作业主控）

**职责**：
- 管理单个作业的生命周期
- 调度作业的任务
- 协调检查点

**生命周期**：
```
作业提交 → 创建 JobMaster → 作业运行 → 作业完成 → 销毁 JobMaster
```

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

### 错误1：混淆 JobManager 和 JobMaster

```java
// ❌ 误解：认为每个 Job 有一个 JobManager
// 实际上，集群中只有 1 个 JobManager，但每个 Job 有 1 个 JobMaster

// ✅ 正确：理解关系
// - JobManager：1 个，长期运行，管理整个集群
// - Job：可以有多个，同时运行
// - JobMaster：每个 Job 1 个，在 Job 生命周期内存在
```

### 错误2：认为 JobManager 也执行任务

```java
// ❌ 误解：认为 JobManager 也处理数据
// JobManager 只负责协调，不执行任务

// ✅ 正确：理解职责分工
// - JobManager：协调和管理
// - TaskManager：执行任务
```

### 错误3：在代码中直接连接 JobManager

```java
// ❌ 错误：在代码中硬编码 JobManager 地址
StreamExecutionEnvironment env = StreamExecutionEnvironment.createRemoteEnvironment(
    "jobmanager-host", 8081, "job.jar"
);

// ✅ 正确：使用 getExecutionEnvironment()，通过配置指定
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// JobManager 地址通过 flink run 命令或配置文件指定
```

### 错误4：忽略 JobManager 高可用配置

```java
// ❌ 错误：生产环境不配置高可用
// 如果 JobManager 失败，整个集群无法工作

// ✅ 正确：生产环境配置高可用
// 在 flink-conf.yaml 中配置高可用
```

## Flink 与 Spark 的对比

如果你熟悉 Spark，可以通过以下对比更好地理解 Flink 的架构：

### 架构对比

| 概念 | Flink | Spark |
|------|-------|-------|
| **集群主节点** | JobManager（1 个，长期运行） | Spark Driver（每个应用 1 个） |
| **作业管理** | JobMaster（每个 Job 1 个） | SparkContext（每个应用 1 个） |
| **工作节点** | TaskManager（多个） | Executor（多个） |
| **资源单元** | Slot | Core |
| **作业提交** | 提交到长期运行的集群 | 每个应用启动自己的 Driver |

### 关键区别

#### 1. 集群模式

**Flink**：
```
长期运行的集群
├── JobManager（1 个，长期运行）
└── TaskManager（多个，长期运行）

提交多个 Job 到同一个集群
├── Job 1 → JobMaster 1
├── Job 2 → JobMaster 2
└── Job 3 → JobMaster 3
```

**Spark**：
```
每个应用启动自己的集群
├── Application 1 → Driver 1 + Executors
├── Application 2 → Driver 2 + Executors
└── Application 3 → Driver 3 + Executors
```

**区别**：
- **Flink**：集群长期运行，多个 Job 共享集群资源
- **Spark**：每个应用启动自己的 Driver，资源隔离更好但开销更大

#### 2. JobManager vs Spark Driver

**Flink JobManager**：
- 集群中**只有 1 个**（长期运行）
- 可以管理**多个 Job**
- 类似于 Spark 的 **Cluster Manager**（如 YARN ResourceManager）

**Spark Driver**：
- 每个应用有**1 个 Driver**
- 只管理**1 个应用**
- 类似于 Flink 的 **JobMaster**

**类比**：
```
Flink:
  JobManager ≈ Spark 的 YARN ResourceManager（集群级别）
  JobMaster ≈ Spark 的 Driver（作业级别）

Spark:
  Driver ≈ Flink 的 JobMaster（作业级别）
  YARN ResourceManager ≈ Flink 的 JobManager（集群级别）
```

#### 3. JobMaster vs SparkContext

**Flink JobMaster**：
- 每个 Job 有 1 个 JobMaster
- 在 Job 提交时创建，Job 完成时销毁
- 管理单个 Job 的生命周期

**Spark SparkContext**：
- 每个应用有 1 个 SparkContext
- 在应用启动时创建，应用完成时销毁
- 管理单个应用的生命周期

**相似点**：
- 都是作业/应用级别的管理组件
- 都在作业/应用生命周期内存在
- 都负责调度和协调

#### 4. TaskManager vs Executor

**Flink TaskManager**：
- 长期运行的工作节点
- 提供多个 Slot
- 可以运行多个 Job 的任务

**Spark Executor**：
- 应用级别的工作节点
- 提供多个 Core
- 只运行一个应用的任务

**区别**：
- **Flink**：TaskManager 长期运行，可以运行多个 Job 的任务
- **Spark**：Executor 是应用级别的，只运行一个应用的任务

#### 5. Slot vs Core

**Flink Slot**：
- TaskManager 的资源单元
- 多个 Slot 共享 TaskManager 资源
- 可以运行不同 Job 的任务

**Spark Core**：
- Executor 的计算资源
- 每个 Core 可以运行一个任务
- 只运行一个应用的任务

### 作业提交流程对比

**Flink**：
```
1. 集群启动（JobManager + TaskManager 长期运行）
2. 提交 Job 1 → JobManager 创建 JobMaster 1
3. 提交 Job 2 → JobManager 创建 JobMaster 2
4. 两个 Job 共享同一个集群
```

**Spark**：
```
1. 提交 Application 1 → 启动 Driver 1 + Executors 1
2. 提交 Application 2 → 启动 Driver 2 + Executors 2
3. 两个应用各自独立的集群资源
```

### 资源利用对比

**Flink**：
- ✅ **资源共享**：多个 Job 共享 TaskManager 资源
- ✅ **资源利用率高**：资源可以动态分配给不同 Job
- ⚠️ **资源隔离较差**：Job 之间可能相互影响

**Spark**：
- ✅ **资源隔离好**：每个应用有独立的资源
- ⚠️ **资源利用率较低**：资源不能跨应用共享
- ⚠️ **启动开销大**：每个应用需要启动 Driver 和 Executors

### 使用场景对比

**Flink 适合**：
- 长期运行的流处理作业
- 需要资源共享的场景
- 多个作业共享集群资源

**Spark 适合**：
- 批处理作业（虽然也支持流处理）
- 需要资源隔离的场景
- 每个应用需要独立资源

## 什么时候你需要想到这个？

- 当你需要**理解作业如何被调度**时
- 当你需要**配置集群资源**时
- 当你需要**理解故障恢复机制**时
- 当你需要**配置 JobManager 高可用**时
- 当你需要**查看作业状态**时（通过 Web UI）
- 当你需要**理解检查点如何工作**时
- 当你需要**从 Spark 迁移到 Flink**时（理解架构差异）

