# Distributed File System

## Network File System(NFS)

* NFS系统包含**NFS Server** & **NFS Client**
* 多个NFS Client通过**RPC**连接NFS Server进行文件操作
* NFS Client对于上层应用提供POSIX文件操作接口
* NFS Client与NFS Server不采用POSIX接口而采用自定义的一套NFS Protocols

### NFS Protocols

#### Server API

NFS RPC并不提供类似于POSIX中的OPEN & CLOSE接口。

**OPEN & CLOSE操作涉及对内存中数据结构File Descriptor Table & File Table的维护**，每次打开一个文件需要创建File Table中的Entry维护文件的读写游标(Cursor)以及对应的Inode ID，打开文件的进程中需要维护File Descriptor Table来保证对File Table中Entry的引用。

**File Descriptor Table & File Table均为内存中结构**，在系统中如果NFS Server宕机重启，意味着内存中没有已经打开文件的任何信息。如果NFS Client是利用NFS Server端的File Descriptor来表明具体操作的文件，当Server宕机后重启，Client端记录的File Descriptor就没有任何意义。

因此NFS采用Stateless NFS Server的设计，每一次NFS Client调用RPC接口时就传递所有相关的信息，**NFS Server不记录客户端读取的任何状态（比如File Table、File Descriptor Table、Cursor...），而由NFS Client端来记录所有状态**。

#### File Handler

为了让NFS Client能够通过某种命名方式来完成RPC调用NFS Server，NFS Client采用**File Handler**作为唯一标识NFS Server端文件的命名方式。

File Handler主要包含**Inode Number(Inode ID) & Generation Number**。Generation Number是一个**全局递增的整数**，每次**新分配一个inode时就会附带一个Generation Number**，并且Generation Number计数器自增。以此来**唯一识别Server端的每一个文件**。

NFS Client通过RPC调用NFS Server中的文件操作接口时，就采用File Handler来标识要操作的文件。

NFS Client不能采用Server端的Inode ID 或 文件路径名来标识文件。

> * File Handler不能直接使用Inode ID。并发操作下某一端对文件删除后再创建会导致相同Inode ID被再次分配，其他端可能会因此读取错误的文件内容。
>
>   ![image-20241113193756837](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113193756837.png)
>
> * FIle Handler不能直接使用文件在Server端的路径名，即File Name。当目录中出现Rename操作后，会根据路径名读取到被重命名的其他文件。
>
>   ![image-20241113193942303](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113193942303.png)

NFS Client端对于上层应用提供的仍是POSIX接口，意味着**NFS Client端要维护File Table & File Descriptor结构**，而NFS Client是利用File Handler与NFS Server进行RPC通信，所以**NFS Client内存中需要维护本地生成的File Descriptor(fd)到NFS Server端传来的File Handler的映射**。

#### Offset

因为NFS Server端设计为无状态，所以NFS Server端不会维护文件游标状态，NFS Client需要向上层呈现POSIX接口，由NFS Client来维护操作文件的游标。而**NFS Client调用RPC选择向NFS Server端传递要进行读/写的起始位置在文件中的起始偏移量(Offset)**，Client端自己记录当前偏移量。Offset保证了NFS Protocol文件操作语义的幂等性，只要传递相同的参数，保证执行结果每次都一致，这样**利于RPC通信在发送失败时的重试**。

![image-20241113195406620](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113195406620.png)

------

## GFS

### Architecture

* **1 Master + N Chunk Servers**。
* Master存储文件的元信息。
* Chunk Servers负责存储文件的Chunk。

![image-20241118220018031](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241118220018031.png)

### Chunk

* 文件分割为一个个较大的Chunk，**一个文件不同的Chunk可以分布在不同的Chunk Server上**。
* 每个文件都有**多个副本**，**每个副本的Chunk分布在不同的Chunk Server上达到容灾的目的**。

### No-directory

* GFS放弃了目录结构
* 采用内存中的FileName → Chunk Locations映射（Hash Table / Other Key Value Store Structure）来替代目录。

### Master Server

* **Master存储每个文件的各个Chunk所在的位置(Chunk ID → Chunk Location)，比如说在哪台Chunk Server，Chunk Server中的具体位置**。
* 所有元信息存储在**内存**中，保证快速访问。如果宕机，则重启后访问所有Chunk Server来重建Chunk ID → Chunk Location的映射，因此Chunk Server中也需要记录具体的Chunk属于哪个文件以及Chunk ID等信息。

### Chunk Server

* Chunk默认大小64MB，大的Chunk可以有效缓解频繁的网络传输数据，Client在进行写的时候不需要频繁到Master Server查询Chunk Location的映射。
* Chunk足够大，则描述文件的元信息（比如说Chunk Location）在Master Server就会相应的变小，保证元信息可以都存储在内存中。

### Client

* Client会与Master Server以及各个Chunk Server进行通信。
* Client与Master Server通信以获取文件元信息
* Client与Chunk Server通信以读写文件内容。

### Read Op

* Client连接到Master，从Master中获取到文件的元数据（文件的Chunk Location信息）。
* 因为文件会有多副本备份，Client可以向**任何一个有效的Chunk Server发送信息获取对应的Chunk**，或者向多个Chunk Server请求，只要有一个返回Chunk即可。
* 当获取过一次文件的元信息后，Client端就可以缓存住这部分元信息，之后读写可以不用去Master查询，除非文件元信息出现了更改（比如说删除Chunk、新分配Chunk）

### Write Op

GFS保证最终一致性，也就是在多副本的情况下，文件的写在过程中会出现副本之间信息不一致，但是保证最终会呈现一致的状态。

#### Primary & replicas

GFS中的存储每个文件会有多个副本，默认是会有三个数据副本，每个文件副本的Chunk可能存储在任一个Chunk Server上。**这就意味着当一个Chunk被写入的时候，其余的副本也应该被写入相同的信息以保证数据一致。**

为了保证文件的最终一致性以及副本之间的同步，**GFS采取了控制流与数据流分离的方式**。

在多台Chunk Server中，选定一台Chunk Server作为Primary，Primary来负责裁决如何对文件进行修改，其余Chunk Server上的Chunk副本的修改操作都必须与Primary指定的一致。

数据流与控制流分离的方式有效缓解了网络带宽的问题。

考虑这样一种形式，所有写操作必须发送到Primary来先决定写的顺序，然后写操作的内容以及Primary决定好的写顺序一起发送到所有含有该副本的Chunk Server中，时延取决于网络带宽，因为含副本的Chunk Server至少两个，意味着Primary Server的网络带宽会成为性能瓶颈，并且也使得Primary Server的效率低下，因为所有的写都必须由Primary转发。数据流与控制流分离的方式解决了以上问题，使得网络带宽不会成为性能瓶颈。

#### Primary Lease

Primary由Master来指定，一段时间内某一台Chunk Server被Master指定为Primary，因为考虑到宕机故障等因素，当通过心跳机制发现Primary下线时，Master可以指定其他机器为Primary。

#### 数据流

* 写入操作分为两阶段，第一阶段是**数据流**，要进行写入的Chunk内容在Chunk Server间传递，但并未真正写入，暂时存储在内存中。
* Client发送写入的Chunk内容到某一台Chunk Server，然后这个Chunk Server将Chunk内容转发到另一台存储对应副本的Chunk Server，而这一台Chunk Server又将Chunk内容继续传递给下一个存储对应副本的Chunk Server...这样形成一条数据链，最终所有存储对应Chunks内容的Chunk Server都会持有要写入的新内容。
* 每个Chunk Server转发一次写入的Chunk内容，不会像只由Primary转发到所有Chunk Server那样传输内容过大受网络带宽限制。

![image-20241118220036213](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241118220036213.png)

#### 控制流

* 写入操作第一阶段结束后，进入第二阶段的控制流，Client发送消息到Primary Server，要求Primary Server决定如何应用数据更改（在并发场景下决定写入的顺序）
* Primary Server将写入的决策发送给所有Secondary Chunk Servers。

* Primary Server中会记录为每个Chunk记录Chunk Version，用以检测Chunk Server中数据是否是最新的。

* 控制信息非常轻量级，意味着即使由Primary发送向所有Secondary Chunk Server也不会成为性能瓶颈。

![image-20241118220044781](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241118220044781.png)

