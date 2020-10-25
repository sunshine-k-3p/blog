---
layout:     post
title:      "Flink Checkpoint机制"
date:       2020-10-25 00:00:00
author:     "k"
header-img: "img/post-bg-alitrip.jpg"
tags:
    - Flink
    - Hadoop
    - 
---

flink的checkpoint是由CheckpointCoordinator内部的⼀个timer线程池定时产⽣生的，具体代码由
ScheduledTrigger这个Runnable类启动。
1. 环境的前置检查  
![](/blog/img/post/flink-checkpoint/1_1.jpg)
![](/blog/img/post/flink-checkpoint/2_1.jpg)
上⾯面代码做一些前置检查，包括如是否checkpoint数量是否超过最大值，如果是就创建队列。
checkpoint间隔是否小于最小值，等等。如有不满⾜情况，不执行checkpoint实际动作。 
2. ⽣生成pendingcheckpoint   
![](/blog/img/post/flink-checkpoint/3_1.jpg)
![](/blog/img/post/flink-checkpoint/3_2.jpg)
pendingcheckpoint表示⼀一个待处理理的检查点，每个pendingcheckpoint标有一个全局唯一的递增
checkpointID，并声明了了一个canceller⽤用于后续超时情况下的checkpoint清理用于释放资源。
上述⽅方法之后，
pendingcheckpoint在正式执行前还会再执⾏⼀遍前置检查，主要等待完成的检查点数量
是否过多以及前后两次完成的检查点间隔是否过短等问题，这些检查都通过后，会把之前定义好的
cancller注册到timer线程池，如果等待时间过长会主动回收checkpoint的资源。
3. 启动checkpoint执⾏  
发送这个checkpoint的checkpointID和timestamp到各个对应的executor,也就是给各个TaskManger发一个TriggerCheckpoint类型的消息。
其中，for (Execution execution: executions) 这⾥面的executions⾥面是所有的输⼊入节点，也就是flink source节点，所以checkpoint这些barrier 时间首先从jobmanager发送给了所有的source task
4. checkpoint处理逻辑   
TriggerCheckpoint消息进⼊入TaskManager的处理路径为 handleMessage -> handleCheckpointingMessage -> Task.triggerCheckpointBarrier
![](/blog/img/post/flink-checkpoint/4_1.jpg)   
checkpoint具体执行逻辑分很多种，具体如下:    
![](/blog/img/post/flink-checkpoint/5_1.jpg)  
preformCheckpoint是具体执⾏checkpoint的逻辑，以StreamTask为例:  
![](/blog/img/post/flink-checkpoint/5_2.jpg)  
在checkpointState方法中调用executeCheckpointing.
 StreamOperator的snapshotState(long checkpointId,long timestamp,CheckpointOptions checkpointOptions)⽅方法最终由它的⼦类AbstractStreamOperator给出了一个final实现。
![](/blog/img/post/flink-checkpoint/6_1.jpg)
由此，可以看出checkpoint最终使用keyed State或者Operator State通过算子本身的性质去判断。
