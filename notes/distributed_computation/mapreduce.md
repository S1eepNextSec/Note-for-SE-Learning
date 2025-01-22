# MapReduce

## 容错 Fault tolerance

### worker failure

Re-Exectuion

map reduce函数保证函数式 没有副作用 当map reduce执行过程中宕机 只要再用相同的输入进行一次重新执行即可 reexecution本质上是时间上的冗余 出现问题重新执行一次来容错

步骤：

failure detection 

如果心跳失败 就认为检测到failure 重新将对应的任务调度到另一个worker上进行执行即可

心跳机制

master failure

checkpoint master定期将自己的运行状态写到gfs中 如果一个master宕机 可以另外启动一个master 根据checkpoint进行恢复 以此来避免master宕机后重新执行所有当前的任务 持久计划的状态要包含各个map reduce任务的执行状态以及中间文件的存储位置

问题

map/reduce函数作为用户传入的进行执行的函数，函数执行计算的过程中可能因为各种原因导致崩溃，导致某一个map/reduce worker执行的时候出现错误，对于这种bad record，MapReduce框架直接跳过这些bad record，当然会带来最终结果上计算准确度的损失。

局部性优化

从网络中读取大量数据很慢，MapReduce框架中文件读取可以大致认为是以下两种：Map函数从底层GFS文件存储中读取文件；Reduce函数从底层GFS中读取Map阶段产生的中间文件。Map阶段读取大量输入文件对网络带宽占用很大，效率受制于网络，因此基于GFS作为底层存储的MapReduce框架可以打破GFS底层文件系统的封装，使得MapReduce中的Master节点可以获取到关于GFS底层文件Chunk的分布状况，在调度任务的时候将任务分配给存储对应文件Chunks的机器上来执行Map/Reduce操作，使拥有Input Chunk的机器作为Map Worker。这样在Map阶段就可以直接在本地读取输入文件，减少网络的大量数据传输。

大量机器的集群中，总会有机器运行得比别的机器慢的多（规模效应），称之为Straggler。为了避免Straggler拖慢整个阶段，可以将对应得工作冗余地分配到多个工作节点上执行，最快完成的那一个节点的输出作为该任务的结果即可。但是这种解决方案只能适用于单个任务不需要长时间运行的情况，如果单个任务要执行很长的时间进行计算并且占用大量计算资源（e.g.大模型训练），这种方案显然不行。

## 局限性

* 非常依赖磁盘I/O，大量中间文件是存储在文件系统中的，要频繁地写入/读取。
* 编程上抽象非常有限，对复杂的工作场景支持差。e.g. Top K查询，需要多轮的Map/Reduce形成一个Chain。如果多个MapReduce互相依赖，无法保证容错，MapReduce的容错机制只保证单个MapReduce Stage内部的Map/Reduce执行是容错的。
* 针对批处理，无法支持实时计算
