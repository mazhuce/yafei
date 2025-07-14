#### DataStream API & Dataflow

这是 Flink 的核心编程模型。
- **是什么**：在 Flink 中，所有程序（包括批处理）都被建模为数据流（Dataflow）。你通过定义数据源（Source）、一系列转换操作（Transformation）、以及数据汇（Sink）来构建一个数据流图。这个图会被 Flink 优化并部署到集群上执行。
#### 有状态计算 (Stateful Computation)

这是 Flink 最强大、最核心的特性。
- **是什么**：State 是在处理数据流时，算子（Operator）需要“记住”的信息。例如，在计算一分钟内的交易总额时，那个不断累加的“总额”就是 State。Flink 提供了容错的、可扩展的状态管理机制。    
- **State Backends (状态后端)**：决定了 State 的存储方式。    
    - `MemoryStateBackend`：状态存储在 TaskManager 的 JVM 堆内存中，速度最快，适合开发和测试。        
    - `FsStateBackend`：状态存储在 TaskManager 的堆内存中，但会定时将快照写入远程文件系统（如 HDFS）。        
    - `RocksDBStateBackend`：状态存储在本地磁盘的 RocksDB 中（一种嵌入式 KV 数据库），支持增量快照，能管理远超内存大小的超大规模状态。
#### 时间概念 (Time) 与窗口 (Window)

处理无穷数据流的关键。
- **时间**：    
    - **事件时间 (Event Time)**：事件在源头实际发生的时间。这是保证结果准确性的关键。        
    - **处理时间 (Processing Time)**：事件被 Flink 算子处理时的机器系统时间。简单但不准确。        
    - **摄入时间 (Ingestion Time)**：事件进入 Flink Source 算子的时间。        
- **窗口 (Window)**：将无限的数据流切分成有限的“桶”来进行计算。    
    - **滚动窗口 (Tumbling Window)**：固定大小，无重叠。如每分钟的PV。        
    - **滑动窗口 (Sliding Window)**：固定大小，有重叠。如每10秒计算过去1分钟的PV。        
    - **会话窗口 (Session Window)**：根据活动的间歇期（gap）来切分。如用户在线会话分析。
#### 架构：JobManager & TaskManager

- **JobManager (Master)**：负责协调整个作业的执行。它接收作业，生成执行图，调度任务到 TaskManager，并协调 Checkpoint。    
- **TaskManager (Worker)**：实际执行计算的节点。它包含多个任务槽（Task Slot），每个槽可以执行一个并行任务（Sub-task）。

**Flink 和 Spark Streaming 有什么核心区别？**

- **处理模型**：Flink 是**真正的逐条处理**（Native Streaming），延迟可以达到毫秒级。而旧的 Spark Streaming 是**微批处理**（Micro-batch），它将流数据切成小时间段的批次来处理，延迟通常是秒级。
    
- **状态管理**：Flink 提供了非常强大和灵活的本地状态管理，并与 Checkpoint 机制深度集成。Spark 的状态管理相对较弱。
    
- **迭代计算**：Flink 原生支持数据流的迭代计算，在图计算和机器学习领域有优势。

**Flink 的 Exactly-Once 是如何实现的？** 通过**分布式快照（Distributed Snapshot）机制**，也称为 Checkpoint。

- Flink 在数据流中周期性地插入一种叫 **Barrier（栅栏）** 的特殊标记。
    
- 当一个算子收到所有上游输入的 Barrier 后，它会对自己当前的 State 做一个快照，并把这个快照存到配置好的 State Backend（如 HDFS）。
    
- 作业失败后，Flink 会从最近一次**完整成功**的 Checkpoint 中恢复所有算子的 State，并重置数据源的读取位置，然后重新处理，从而保证数据不多不少，恰好处理一次。

**Checkpoint 和 Savepoint 有什么区别和联系？**

- **目的不同**：`Checkpoint` 的主要目的是**故障恢复**，是 Flink 自动进行的。`Savepoint` 是用户**手动触发**的，主要用于计划性的**运维操作**，如升级 Flink 版本、修改作业逻辑、A/B 测试等。
    
- **生命周期**：`Checkpoint` 默认在作业停止后会被删除。`Savepoint` 会被永久保留在外部存储中，直到用户手动删除。
    
- **技术上**：`Savepoint` 就是一种特殊格式的 `Checkpoint`。

**如何选择合适的 State Backend？**

- **开发测试**或**状态极小**：使用 `MemoryStateBackend`。
    
- **状态较大**，有高可用要求，但**QPS不高**：使用 `FsStateBackend`。
    
- **状态非常大**（上百GB甚至TB级别），或需要**增量 Checkpoint** 以降低快照开销：必须使用 `RocksDBStateBackend`。

**Flink 如何处理数据乱序问题？** 通过 **Watermark (水印)** 机制。

- Watermark 是一个特殊的时间戳，它在数据流中流动，代表“时间戳小于此 Watermark 的事件已经全部到达”。
    
- 当一个窗口接收到一个 Watermark，如果该 Watermark 的时间超过了窗口的结束时间，窗口就会被触发计算。
    
- 通过设置一个**延迟时间** (`allowedLateness`)，可以允许窗口在触发后继续等待一段时间的迟到数据。

**什么是 Flink 的反压 (Backpressure) 机制？** 

反压是指下游算子的处理速度跟不上上游时，能够将压力向上游传递，最终减缓源头（Source）数据读取速度的机制。 Flink 基于其内部的**信贷控制（Credit-based）流控机制自动实现反压。下游 TaskManager 会向上游“申请”可用的网络缓冲区（Credit），只有拿到 Credit，上游才能发送数据。如果下游繁忙，不返回 Credit，上游就会被阻塞，从而实现压力的传导。
#### 算子
##### **1. 无状态转换 (Stateless Transformations)**
- `map(func)`: 一对一转换。    
- `flatMap(func)`: 一对多转换。    
- `filter(func)`: 过滤。    
- `project(indexes)`: (仅用于 Tuple 类型) 选择指定索引的字段，生成新的 Tuple。    
##### **2. 有状态转换 (Stateful Transformations)**
- `keyBy(keySelector)`: 逻辑上对流进行分区。所有具有相同 Key 的记录都会被发送到同一个下游任务实例。这是进行有状态计算前**必须**的一步。    
- **窗口操作 (Windowing)**:    
    - `.window(WindowAssigner)`: 将 `KeyedStream` 转换成 `WindowedStream`。`WindowAssigner`可以是 `TumblingEventTimeWindows`, `SlidingProcessingTimeWindows` 等。        
    - `.timeWindow(Time size)`: `.window()` 的一种简写。        
    - `.countWindow(size)`: 按数量切分窗口。        
    - `.apply(WindowFunction)` / `.reduce(ReduceFunction)` / `.aggregate(AggregateFunction)`: 在窗口上执行计算。`.apply` 最通用，但性能较低；`.reduce` 和 `.aggregate` 可以进行增量聚合，性能更好。        
- **连接操作 (Joins)**:    
    - `stream.join(otherStream).where(key1).equalTo(key2).window(...)`: 基于窗口的常规 Join。        
    - `stream.intervalJoin(otherStream).between(...)`: 基于时间区间的 Join，效率更高。        
- `process(ProcessFunction)`: Flink 中最底层的操作，可以访问事件、状态和时间（定时器），能实现任何复杂的业务逻辑，是 Flink 的“大招”。   
##### **3. 多流转换 (Multi-Stream Transformations)**
- `union(stream1, stream2, ...)`: 合并多个数据类型相同的流。    
- `connect(otherStream)`: 连接两个数据类型**可以不同**的流，生成一个 `ConnectedStreams`。后续需要用 `map` 或 `flatMap` 等算子来定义两个流各自的处理逻辑。是实现流与流之间状态共享的基础。    
- `broadcast(otherStream)`: 将一个流（通常是小数据量的配置流）广播到下游算子的所有并行实例中。   
##### **4. Sink**
- `addSink(sinkFunction)`: 将数据流写入外部系统，如 `FlinkKafkaProducer`, `JdbcSink` 等。
- `print()`: 打印到标准输出，主要用于调试。