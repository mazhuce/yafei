#### RDD (Resilient Distributed Dataset - 弹性分布式数据集)

这是 Spark 最早、最核心的数据抽象。
- **是什么**：一个不可变的、可分区的、可并行计算的分布式对象集合。你可以把它想象成一个分布在集群多台机器上的巨大数组。    
- **面试常问**：    
    - **RDD 有哪些特性？**        
        - **分区列表 (A list of partitions)**：每个 RDD 被分成多个分区，分布在集群的不同节点上，这是并行计算的基础。            
        - **计算函数 (A function for computing each split)**：每个分区上都有一个计算函数来处理其数据。            
        - **依赖关系 (A list of dependencies on other RDDs)**：记录了该 RDD 是如何从其他 RDD 转换而来的，这被称为“血缘关系(Lineage)”，是故障恢复的关键。            
        - **分区器 (Optionally, a Partitioner)**：对于键值对 (Key-Value) 类型的 RDD，可以有分区器来决定数据如何分布。            
        - **优先位置 (Optionally, preferred locations)**：为了优化计算，可以指定分区的优先计算位置（如靠近数据存储的节点）。            
    - **RDD 的不可变性有什么好处？** 不可变性使得数据和计算逻辑更简单、可预测。更重要的是，它配合“血缘关系”可以轻松实现故障恢复。如果一个分区的数据丢失，Spark 可以根据其血缘关系重新计算出来，而不用做复杂的数据复制和检查点。
#### DataFrame 与 Dataset

这是在 RDD 基础上更高级的 API，也是目前推荐使用的方式。
- **是什么**：    
    - **DataFrame**：可以看作是带有 Schema（列名和类型）的 RDD，类似于关系型数据库中的表。它支持更丰富的操作，并且可以通过 Catalyst 优化器进行查询优化。        
    - **Dataset**：是 DataFrame 的扩展，增加了编译时期的“类型安全”。它是强类型的，你可以在编译时就发现类型错误。在 Scala/Java 中，`DataFrame` 只是 `Dataset[Row]` 的一个特例。        
- **面试常问**：    
    - **RDD、DataFrame、Dataset 的区别和联系？**        
        - **RDD**：最底层，无结构化信息，不享受 Catalyst 优化，但提供了最大的灵活性。适用于非结构化数据或需要精细控制物理执行的场景。            
        - **DataFrame**：拥有 Schema，通过 Catalyst 优化器和 Tungsten 执行引擎获得极高的性能，但牺牲了编译时类型安全。是 Python 和 R 用户的首选。            
        - **Dataset**：结合了 RDD 的类型安全和 DataFrame 的性能优化。是 Scala 和 Java 开发的首选。
#### Driver、Executor 与 Cluster Manager

这是 Spark 的运行架构。
- **Driver (驱动程序)**：是你的 Spark 应用的 `main` 函数所在进程。它负责创建 `SparkContext`，将代码解析成逻辑计划（DAG），并调度任务到 Executor 上执行。    
- **Executor (执行器)**：是在工作节点 (Worker Node) 上启动的工作进程。它负责执行 Driver 分配的具体任务 (Task)，并将结果返回给 Driver。每个 Executor 都有自己独立的内存和 CPU 核心。    
- **Cluster Manager (集群管理器)**：负责分配和管理集群资源，如 YARN、Mesos 或 Spark 自带的 Standalone Manager。
#### 面试问题

- **Spark 为什么比 MapReduce 快？**    
    - **内存计算**：Spark 优先将中间数据存储在内存中，避免了 MapReduce 大量的磁盘 I/O。        
    - **DAG 执行引擎**：Spark 将计算过程表示为有向无环图 (DAG)，可以进行深度优化，减少不必要的计算步骤和数据 Shuffle。        
    - **Lazy Evaluation (惰性求值)**：Spark 的转换操作 (Transformation) 不会立即执行，而是构建 DAG。只有当遇到动作操作 (Action) 时，才会真正触发计算，这为优化提供了空间。
- **解释一下 Transformation 和 Action 的区别。**    
    - **Transformation (转换)**：是惰性执行的，它从一个 RDD 生成一个新的 RDD，例如 `map()`, `filter()`, `groupByKey()`。它们只是定义了计算的逻辑，并不会立即执行。        
    - **Action (动作)**：会触发实际的计算，将结果返回给 Driver 或写入外部存储。例如 `count()`, `collect()`, `saveAsTextFile()`。一个 Action 会启动一个 Job。
- **什么是宽依赖和窄依赖？**    
    - **窄依赖 (Narrow Dependency)**：父 RDD 的每个分区最多只被子 RDD 的一个分区使用。计算可以在单一节点上完成，无需数据混洗 (Shuffle)。例如 `map`, `filter`。        
    - **宽依赖 (Wide Dependency)**：子 RDD 的一个分区依赖于父 RDD 的所有或多个分区。这通常需要进行 Shuffle，即在网络间重新分发数据。例如 `groupByKey`, `reduceByKey`。宽依赖是划分 Stage (阶段) 的依据。
- **你如何对 Spark 作业进行性能调优？** 这是一个开放性问题，可以从以下几个方面回答：    
    - **使用高性能 API**：优先使用 DataFrame/Dataset API，以充分利用 Catalyst 优化器。        
    - **避免数据倾斜 (Data Skew)**：检查数据分布，对于倾斜的 key，可以采用加盐（salting）、局部聚合等方式打散数据。        
    - **优化 Shuffle 过程**：        
        - 用 `reduceByKey` 或 `aggregateByKey` 替代 `groupByKey`，因为前者会在 Map 端进行预聚合，大大减少 Shuffle 的数据量。            
        - 合理设置分区数 (`spark.sql.shuffle.partitions`)，避免分区过多（任务调度开销大）或过少（并行度不足）。            
    - **内存管理**：        
        - 使用 `cache()` 或 `persist()` 缓存被频繁使用的中间 RDD/DataFrame，避免重复计算。            
        - 对于小表 Join 大表的场景，使用广播变量 (`Broadcast Variable`) 将小表广播到每个 Executor，避免 Shuffle。            
    - **资源配置**：合理配置 Executor 的数量、内存 (`executor-memory`) 和 CPU 核心数 (`executor-cores`)，以充分利用集群资源。        
- **`repartition` 和 `coalesce` 有什么区别？**    
    - **`repartition(n)`**：用于增加或减少分区数量。它总是会触发完整的 Shuffle，代价很高，但能确保数据在分区中均匀分布。        
    - **`coalesce(n)`**：仅用于减少分区数量。它会尝试避免 Shuffle，通过合并相邻分区来实现，因此效率更高。但可能导致数据分布不均。
- **简述一个 Spark Job 的提交和执行流程。**    
    1. **提交**：用户通过 `spark-submit` 提交应用。        
    2. **Driver 初始化**：Driver 进程启动，创建 `SparkContext`。        
    3. **构建 DAG**：Driver 将代码中的 Transformation 构建成一个逻辑执行计划（DAG）。        
    4. **划分 Stage**：DAGScheduler 将 DAG 按宽依赖划分为多个 Stage（阶段）。        
    5. **生成 Task**：每个 Stage 内部根据分区数被划分为多个 Task。        
    6. **调度任务**：TaskScheduler 将 TaskSet（一组任务）发送给集群管理器。        
    7. **执行任务**：集群管理器在 Executor 上分配资源，Executor 启动并执行 Task。        
    8. **返回结果**：Task 执行结果返回给 Driver，或写入外部系统。        
- **Spark 是如何实现故障容错的？** 主要依靠 RDD 的“血缘关系”（Lineage）。由于 RDD 是不可变的，并且 Spark 记录了每个 RDD 是如何从其父 RDD 转换而来的，所以当某个 RDD 的一个分区数据丢失时，Spark 可以通过血缘关系找到它的父 RDD，重新进行计算以恢复该分区，而无需备份整个数据集。

### 转换算子 (Transformation) - 懒执行，返回新的 RDD
- **map(func)**: 将 RDD 中的每个元素通过一个函数转换成一个新的元素。    
- **filter(func)**: 筛选出 RDD 中满足条件的元素。    
- **flatMap(func)**: 与 map 类似，但每个输入元素可以被映射为零个或多个输出元素（返回一个序列）。    
- **groupByKey()**: 将键值对 RDD 中相同 key 的 value 聚合到一个序列中。    
- **reduceByKey(func)**: 将键值对 RDD 中相同 key 的 value 按照指定的函数进行聚合。    
- **sortByKey()**: 对键值对 RDD 按照 key 进行排序。    
- **union(otherRDD)**: 合并两个 RDD，返回包含两个 RDD 所有元素的新 RDD。    
- **join(otherRDD)**: 对两个键值对 RDD 进行内连接操作。    
### 行动算子 (Action) - 触发计算，返回值或写入外部系统

- **collect()**: 将 RDD 中的所有元素以数组形式返回到驱动程序中。    
- **count()**: 返回 RDD 中的元素个数。    
- **first()**: 返回 RDD 中的第一个元素。    
- **take(n)**: 返回 RDD 中前 n 个元素组成的数组。    
- **reduce(func)**: 通过指定的函数对 RDD 中的所有元素进行聚合。    
- **foreach(func)**: 对 RDD 中的每个元素执行指定的函数（通常用于打印或写入外部系统）。    
- **saveAsTextFile(path)**: 将 RDD 的内容保存为文本文件。