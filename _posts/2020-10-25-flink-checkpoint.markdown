flink的checkpoint是由CheckpointCoordinator内部的⼀一个timer线程池定时产⽣生的，具体代码由 ScheduledTrigger这个Runnable类启动。
1. 环境的前置检查
![]()
上⾯面代码做⼀一些前置检查，包括如是否checkpoint数量量是否超过最⼤大值，如果是就创建队列列。
checkpoint间隔是否⼩小于最⼩小值，等等。如有有不不满⾜足情况，不不执⾏行行checkpoint实际动作。 2. ⽣生成pendingcheckpoint
![]()
![]()
pendingcheckpoint表示⼀一个待处理理的检查点，每个pendingcheckpoint标有⼀一个全局唯⼀一的递增 checkpointID，并声明了了⼀一个canceller⽤用于后续超时情况下的checkpoint清理理⽤用于释放资源。 上述⽅方法之后，pendingcheckpoint在正式执⾏行行前还会再执⾏行行⼀一遍前置检查，主要等待完成的检查点数量量 是否过多以及前后两次完成的检查点间隔是否过短等问题，这些检查都通过后，会把之前定义好的
 cancller注册到timer线程池，如果等待时间过⻓长会主动回收checkpoint的资源。
3. 启动checkpoint执⾏行行
发送这个checkpoint的checkpointID和timestamp到各个对应的executor,也就是给各个TaskManger发⼀一 个TriggerCheckpoint类型的消息。
其中，for (Execution execution: executions) 这⾥里里⾯面的executions⾥里里⾯面是所有的输⼊入节点，也就是flink source节点，所以checkpoint这些barrier 时间⾸首先从jobmanager发送给了了所有的source task
4. checkpoint处理理逻辑 TriggerCheckpoint消息进⼊入TaskManager的处理理路路径为 handleMessage -> handleCheckpointingMessage -> Task.triggerCheckpointBarrier
![]()
checkpoint具体执⾏行行逻辑分很多种，具体如下:
![]()
preformCheckpoint是具体执⾏checkpoint的逻辑，以StreamTask为例例:
![]()
在checkpointState⽅方法中调⽤用executeCheckpointing.
 StreamOperator的snapshotState(long checkpointId,long timestamp,CheckpointOptions checkpointOptions)⽅方法最终由它的⼦子类AbstractStreamOperator给出了了⼀一个final实现。
![]()
由此，可以看出checkpoint最终使⽤用keyed State或者Operator State通过算⼦子本身的性质去判断。
