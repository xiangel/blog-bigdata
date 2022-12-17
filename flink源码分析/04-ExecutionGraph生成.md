
前面我们在介绍DataStream时，介绍了Flink任务提交时从StreamGraph->JobGraph->ExecutionGraph的过程，而如何生成ExecutionGraph并没介绍，本节来介绍在具体调度执行时使用的图结构ExecutionGraph。StreamGraph和JobGraph是在Client生成的。ExecutionGraph是在JobManager(Flink任务执行时的Master节点)端生成的，JobManager会根据提交的JobGraph来生成ExecutionGraph。

## 重要类
### DefaultExecutionGraph
ExecutionGraph的实现类，保存了具体的Graph结构信息、具体执行时的作业和任务相关信息以及作业执行中的中间结果信息等。相关重要属性如下
```
    // JobGraph的节点ID和ExecutionGraph的节点信息映射
    private final Map<JobVertexID, ExecutionJobVertex> tasks;
    // 按依赖顺序的Execution节点数据
    private final List<ExecutionJobVertex> verticesInCreationOrder;

    // 执行尝试信息
    private final Map<ExecutionAttemptID, Execution> currentExecutions;
    //中间结果数据信息
    private final Map<IntermediateDataSetID, IntermediateResult> intermediateResults;
    // 当前作业状态
    private volatile JobStatus state = JobStatus.CREATED;

    //执行拓扑结构
    private DefaultExecutionTopology executionTopology;
    // checkpoint处理协调器
    @Nullable private CheckpointCoordinator checkpointCoordinator;
```

### ExecutionJobVertex 
在ExecutionGraph中的节点信息，与JobGraph的JobVertex是一一对应的。其中存储了
```
    // 每个子任务节点信息
    @Nullable private ExecutionVertex[] taskVertices;

    // 产出数据集
    @Nullable private IntermediateResult[] producedDataSets;

    // 输入数据集
    @Nullable private List<IntermediateResult> inputs;

    // 并行度
    private final VertexParallelismInformation parallelismInfo;
```

### ExecutionVertex 
ExecutionJobVertex中根据并行度生成的单个子任务,包括具体的子任务的编号，执行信息等

### IntermediateResult
节点的每个输出链对应一个IntermediateResult，每个IntermediateResult下按ExecutionJobVertex的并行度对应有相应的IntermediateResultPartition。

### SlotSharingGroup
定义不同节点的任务可以部署到同一个slot中，对slot进行共享，更为有效的使用slot资源。

## ExecutionGraph生成
ExecutionGraph是Scheduler(JobManager中的负责调度处理的类)中实例化时通过调用createAndRestoreExecutionGraph方法来生成ExecutionGraph的。其最终调用的是DefaultExecutionGraphBuilder类中的buildGraph()方法。其具体流程如下：
1. 创建一个DefaultExecutionGraph的实例，这里主要是传入一些参数处理，并没有关联JobGraph的信息；
2. 初始化JobVertex,处理inputoutput的格式信息；
3. 将JobGraph的所有JobVertex进行按依赖顺序进行排序处理；
4. 调用ExecutionGraph的attachJobGraph方法将JobVertex列表信息绑定到ExecutionGraph
    每一个ExecutionJobVertex对应一个JobVertex，每一个IntermediateResult对应到一个JobVertex的IntermediateDataSet，再根据JobVertex的并行度生成对应数量的ExecutionVertex,用数组存储
    根据JobVertex的inputs信息初始化ExecutionJobVertex的inputs信息。
5. 配置statebackend和checkpoint信息，此部分留到介绍checkpoint时再详细介绍

## 总结
本篇接着01-DataStream基础介绍了JobGraph到ExecutionGraph的转换过程。首先介绍了ExecutionGraph中的相关核心概念，如ExecutionJobVertex、IntermediateResult等。后面介绍了ExecutionGraph的详细生成过程。在ExecutionGraph生成的最后会设置checkpoint等信息，此块后面单独介绍。ExecutionGraph生成好后，会通过DefaultScheduler的startScheduling()方法来触发进行调度(具体调度及运行后面介绍)。