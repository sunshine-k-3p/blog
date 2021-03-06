---
layout:     post
title:      "一套Flink实时计算架构设计思路"
date:       2020-10-25 00:00:00
author:     "k"
header-img: "img/post-bg-digital-native.jpg"
tags:
    - Flink
    - Hadoop
    - 
---

### 一、Flink 提供的机制说明 
##### 1. Checkpoint机制  
绘制分布式数据流和操作符状态的一致快照，以保证故障时，Flink可以回退到最近的检查点重启，不丢失数据。
以数据源kafka为例，Flink会为流数据添加barrier，将数据分片，在某一个算子都接收到barrier n时做checkpoint操作，如下图：  
  
![demo](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/stream_barriers.svg) 

```
注意：数据源必须要支持指定回退功能，如kafka便支持指定位置读取操作，所以checkpoint时会存储kafka的偏移量。
```

多流之间checkpoint操作存在一个对齐处理，如下图所示，意在末端算子收到所有算子的barrier之后才会发出快照，通知协调器，否则将会把该流中的后续数据放入缓冲区。  
![demo](https://ci.apache.org/projects/flink/flink-docs-release-1.7/fig/stream_aligning.svg)  

  已探知的知识点：
  1. 按算子和slot数来分单位生成快照数据。    
  2. 执行checkpoint是一个异步的过程，所以对主slot流的影响微乎其微。  
  3. 想要深入了解checkpoint机制的同学推荐从CheckpointCoordinator类为切入口进行深入了解。

##### 2. Savepoint机制  
主要用于恢复之前Job的运行状态。  
savepoint是使用命令主动发起checkpoint的操作，启动Flink的时候可以指定savepoint的保存点来恢复之前的运行情况。

##### 3. State机制    
State主要作用是状态存储。  
3.1 Keyed State   
必须使用在KeyedStream函数后的State。  
支持ValueState、ListState、MapState、ReducingState、AggregatingState。  
3.2 Operator State    
以算子为单位做存储，无需KeyBy操作，该State的数据量与Flink的并行度有关。  
仅支持ValueState、ListState。  

### 二、系统概述 
##### 1. 概述   

实时数仓基于Flink实时计算引擎搭建，其中使用到了Checkpoint机制、Keyed State机制等等，同时为了实现7 * 24不断线发版，我们基于Redisson实现了幂等操作，为保证开发效率我们做了两方面节省开发时间的事，第一点将7 * 24小时不断线复杂发版逻辑进行了抽象封装，第二点实现数据自动存储或者下发（推送kafka），使开发人员仅需要开发特征指标相关业务而无需关心Flink组件的使用，甚至保证一位Java开发人员（不了解大数据组件的开发人员）可以在此框架中完成业务指标开发任务，降低了对开发人员能力要求。以下是Flink实时计算系统UML图：   
![demo](/blog/img/post/flink-frame/1.png)   

### 三、系统原理及使用 
##### 1. 7 * 24小时不断线发版 
不断线发版分为新任务发版、扩展并行度发版、新特征发版三种情况，其中较为复杂的是新特征发版。  
1.1 新任务发版
针对全新的特征任务发版，可以理解为新事件所对应的新任务的发版。下图是发版的时轴图。  
![demo](/blog/img/post/flink-frame/3.png) 
其中historyEventTime启动参数是指定处理事件时间之后的数据。    
1.2 扩展并行度发版  
针对运行中的任务，数据量增大计算资源需要扩充的情况。下图是发版的时间轴图。
![demo](/blog/img/post/flink-frame/4.png)  
在job1执行savepoint之后，job2通过savepoint启动较大资源的相同任务，在job1与job2并行的过程中，通过幂等机制来保证数据处理的唯一性。  
1.3 新特征发版  
针对在运行任务中有新的特征需要加入的情况。下图是发版的时间轴图。 
![demo](/blog/img/post/flink-frame/5.png)
首先Job1执行savepoint，查看库中maxEventTime（当前处理数据的最新事件时间），然后根据此时间补齐state的历史数据和结果特征库中的历史数据，然后设置该maxEventTime且使用savepoint启动Job2任务（含新特征的任务）。此时两任务并行，数据通过幂等机制来保证数据处理的唯一性。Job2在收到事件时间小于maxEventTime的数据时是拒绝存state和入库操作，在大于maxEventTime之后才存储state和入库操作。  
当Job1和Job2两个任务处理Kafka数据的位置相近时，杀死Job1任务。


##### 2. 系统提供的设置参数   
参考类com.kuainiu.realtime.framework.SysConst类，如下图：
![demo](/blog/img/post/flink-frame/2.png)  

### 四、发展方向  
##### 1. 事件特征计算拆解  
利用Flink本身仅做了内存隔离，cpu是没有做隔离的特点，以及现有实时数仓数据量小的情况，实现支持多线程并行处理没有逻辑关联的特征。从而提高计算速度。

##### 2. 寻求实质性的性能突破  

2.1 存储体系的并行度  
现有实时数仓计算体系，性能瓶颈主要是计算过程中查询的并行度较低，因为现使用的是Mysql数据库。未来可以考虑提高并行度更换存储体系，相信性能可以大大增加。

2.2 寻求可落地的中间表的设计方案  
在实际指标开发过程中，我们发现一个复杂指标计算的中间依赖值往往是较为复杂和多的，导致中间数据存什么的问题很难去设计，就算勉强设计出来复杂度也很高，而且补齐历史数据（state、入库的数据）的工作量很大。最终选择了查询Mysql数据库以较低复杂度的问题。  

##### 3. 实时数仓界面配置化  
现有实时数仓计算架构仍然使用的算子逻辑开发实现的，实现可配置化的第一步是将架构完全支持Table API开发，即支持使用sql开发特征计算。从而为界面配置化打下基础。

