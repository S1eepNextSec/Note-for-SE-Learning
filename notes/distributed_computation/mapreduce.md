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

# Computation Graph

点代表计算任务/数据

边代表依赖关系



分布式训练

训练并行化

数据并行化

将训练数据批量切分

模型并行化

把模型的计算过程进行切分

同步/异步更新



数据并行化

可简单通过累加进行合并的计算，可以将数据进行切分，原本的计算图变成可以并行计算的计算图，每个子图可以在单独的设备上进行计算。子图并行计算完后还需要进行求和汇总。

在并行计算阶段，理论上切分成N部分数据就可以由N个设备来并行计算，效率提升到原本的N倍，但是最终需要将N个子图的计算结果进行合并。

通过同步的方式进行模型训练，必须保证前一轮迭代全部计算完才能开始下一个迭代，在单次迭代中采用数据并行化方式进行计算，如果其中一个并行的子图计算被很快完成，依然需要等待所有的并行计算完成后进行数据交换、汇总才能进入下一轮迭代。

进行合并就必须在所有设备之间进行全量的数据交换，所以最终汇总会成为性能瓶颈。ALLReduce

目标

降低并行计算后数据汇总的开销。

AllReduce包括数据计算 + 网络传输，其中计算开销较小，而网络传输开销很大，因此需要减少网络开销。 

* 需要减少单次数据传输的数据量
* 需要避免大量并行的网络数据传输到同一台设备

parameter server

专门一台机器进行allreduce的数据汇总计算，即为paramter server

每台机器计算完毕将对应的数据传输到parameter server

paramter server进行数据汇总计算

parameter server将结果广播到所有机器

假设单个计算节点需要传播的参数规模是P，进行并行计算的节点有N个

计算节点 → Parameter Server阶段：每个计算节点O(P)的网络流量，Parameter Server O(P*N)的网络流量

 Parameter Server→计算节点阶段：每个计算节点O(P)的网络流量，Parameter Server O(P*N)的网络流量



Co-located & sharded Parameter Server

将每个计算节点都变成一个“Parameter Server”

每个计算节点的计算结果规模为P，也就是要向外传播的数据规模是P，将P按照一定规则进行分区，集群中N个节点就分成N份，第0台机器负责Partition 0的数据汇总，第1台机器负责Partition 1的数据汇总……因此每台机器将自己的计算结果中Partition 0的数据发送给0号机器，Partition 1的数据发送给1号机器……

每台机器只需要承担O((N - 1) * P / N)的网络流量（不需要将自己的数据发送给自己）

只需要进行两次通信，数据收集 & 数据汇总后的广播。

high fan in问题

同一时间有 N 台机器与同一台机器通信。



De-Centralized All Reduce 

 

所有的机器都得与N台机器建立网络连接。





Ring Allreduce

问题

对不满足交换律的算子无法使用
