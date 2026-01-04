# TaskManager：任务的执行者

## 核心概念

**TaskManager** 是 Flink 集群的**工作节点（Worker）**，负责**执行实际的数据处理任务**。每个 TaskManager 提供多个 Slot，用于运行 Flink 作业的任务。

### 类比理解

这就像：
- **公司的员工**：执行具体的工作任务
- **餐厅的厨师**：制作菜品（处理数据）
- **Spark 的 Executor**：如果你用过 Spark，TaskManager 在 Flink 中扮演类似的角色

### 核心特点

1. **多个工作节点**：集群中可以有多个 TaskManager
2. **执行任务**：运行 Flink 作业的实际任务
3. **提供资源**：通过 Slot 提供计算资源
4. **数据交换**：在任务之间传输数据

## 源码位置

TaskManager 的主要组件在：
- `flink-runtime/src/main/java/org/apache/flink/runtime/taskmanager/`
- `flink-runtime/src/main/java/org/apache/flink/runtime/taskexecutor/`

## TaskManager 的核心职责

### 1. 执行任务（Task Execution）

**功能**：
- 运行 Flink 作业的任务（如 Source、Map、Filter、Sink 等）
- 执行用户定义的函数（MapFunction、FilterFunction 等）
- 处理数据流

**示例**：
```java
// 当你定义这样的作业时
stream.map(s -> s.toUpperCase())
      .filter(s -> s.length() > 5)
      .print();

// TaskManager 会：
// 1. 在 Slot 中运行 map 任务
// 2. 在 Slot 中运行 filter 任务
// 3. 在 Slot 中运行 print 任务
```

### 2. 数据交换（Data Exchange）

**功能**：
- 在任务之间传输数据
- 处理数据分区（如 keyBy、rebalance 等）
- 管理网络缓冲区

**示例**：
```
Task A (TaskManager 1) → 网络传输 → Task B (TaskManager 2)
```

### 3. 状态管理（State Management）

**功能**：
- 存储任务的状态数据
- 与状态后端交互（如 RocksDB、文件系统）
- 参与检查点的创建

**示例**：
```java
// 当任务使用状态时
ValueState<String> state = getRuntimeContext().getState(stateDescriptor);
state.update("value");

// TaskManager 会：
// 1. 存储状态数据
// 2. 在检查点时持久化状态
```

### 4. 资源提供（Resource Provisioning）

**功能**：
- 提供 Slot 资源
- 向 JobManager 报告可用资源
- 管理 Slot 的分配和释放

**示例**：
```
TaskManager 启动
  ↓
注册到 JobManager
  ↓
报告可用 Slot 数量（如 4 个 Slot）
  ↓
等待 JobManager 分配任务
```

## TaskManager 的组件

### 1. Task Slot（任务槽）

**作用**：
- 运行任务的资源单元
- 每个 Slot 可以运行一个任务
- 多个 Slot 可以共享 TaskManager 的资源

### 2. Network Stack（网络栈）

**作用**：
- 处理任务之间的数据交换
- 管理网络缓冲区
- 实现数据分区策略（如 keyBy、rebalance）

### 3. State Backend（状态后端）

**作用**：
- 存储和管理任务状态
- 支持不同的状态后端（Memory、RocksDB、文件系统）

## 最小可用例子

### 启动 TaskManager

```bash
# Standalone 模式
./bin/taskmanager.sh start

# 或使用 start-cluster.sh（同时启动 JobManager 和 TaskManager）
./bin/start-cluster.sh
```

### 配置 TaskManager

```yaml
# flink-conf.yaml
taskmanager.numberOfTaskSlots: 4  # 每个 TaskManager 的 Slot 数量
taskmanager.memory.process.size: 4096m  # TaskManager 的内存
```

### 查看 TaskManager 状态

```bash
# 访问 Web UI
http://localhost:8081

# 查看 TaskManager 列表和 Slot 使用情况
```

## TaskManager 与并行度的关系

### 资源计算

```
总并行度 = TaskManager 数量 × 每个 TaskManager 的 Slot 数量
```

**示例**：
- 3 个 TaskManager
- 每个 TaskManager 有 4 个 Slot
- 总可用 Slot = 3 × 4 = 12
- 作业最大并行度 = 12

### Slot 分配

```
作业需要并行度 8
  ↓
JobManager 查找 8 个可用 Slot
  ↓
可能分布在多个 TaskManager：
  - TaskManager 1: 3 个 Slot
  - TaskManager 2: 3 个 Slot
  - TaskManager 3: 2 个 Slot
```

## 常见错误

### 错误1：设置过少的 Slot

```yaml
# ❌ 错误：Slot 数量太少
taskmanager.numberOfTaskSlots: 1
# 如果作业并行度是 8，需要 8 个 TaskManager

# ✅ 正确：根据作业需求设置
taskmanager.numberOfTaskSlots: 4
# 如果作业并行度是 8，只需要 2 个 TaskManager
```

### 错误2：混淆 TaskManager 和 Slot

```java
// ❌ 误解：认为 1 个 TaskManager = 1 个并行度
// 实际上，1 个 TaskManager 可以提供多个 Slot

// ✅ 正确：理解关系
// - TaskManager：工作节点
// - Slot：资源单元
// - 1 个 TaskManager 可以提供多个 Slot
```

### 错误3：TaskManager 内存配置不当

```yaml
# ❌ 错误：内存配置太小
taskmanager.memory.process.size: 512m
# 可能导致任务 OOM

# ✅ 正确：根据任务需求配置
taskmanager.memory.process.size: 4096m
# 考虑：任务数量、状态大小、网络缓冲区等
```

## 什么时候你需要想到这个？

- 当你需要**理解任务如何执行**时
- 当你需要**配置 TaskManager 资源**时
- 当你需要**理解 Slot 和并行度的关系**时
- 当你需要**理解数据如何在任务间传输**时
- 当你需要**理解状态如何存储**时
- 当你需要**优化任务性能**时

