# File System

## Block Layer

### Block

* 磁盘中的最小物理存储单位是扇区Sector，磁盘的读写操作以Sector为单位。
* 文件系统中的一个**块Block**一般由**多个连续的扇区Sector**组成，块是文件系统中存储的最小单位，块的大小由文件系统来决定。
* 将磁盘物理上的多个Sector聚合成固定大小的一个Block，使得文件系统对于上层屏蔽了底层硬件的存储细节，这一层抽象使得文件系统上层实现不必限制于底层细节。
* Block Layer提供的抽象使得在文件系统看来，底层磁盘是一个**巨大的Block数组**，每个Block具有固定字节的大小，对于磁盘中的**每一个Block都由一个Block ID来唯一标识**，当需要获取某一具体Block的数据时，只需要根据Block ID索引到具体的Block位置即可。Block~i~在磁盘中起始字节偏移量为 i * Block_Size(i ≥ 0)。

### Block Bitmap

* 用**Bitmap**结构来标记每一个Block ID对应的Block处于**空闲 / 使用状态**。
* 由于一个Block中最多只有$\ Block\_Size * 8$个位，即最多能映射$\ Block\_Size * 8$个Block的空闲状况，所以需要连续的Block块来存储 Block Bitmap。Bitmap本身要用Block来存储，Bitmap总共占用的Block数量为 $\lceil Total\_Block\_Amount / (Block\_Size * 8)\rceil$。
* Block ID为 i 的Block在连续的Block Bitmap块存储中位于第$\ i / (Block\_Size * 8) $个Block存储其空闲状况。
* Block ID为 i 的Block在Bitmap中逻辑上Index为$\ i \% (Block\_Size * 8)$，而逻辑上的Index依赖于Bitmap的实现将其转换为Block中的具体第几个字节以及该字节中的第几个位。

### Allocate A Block

* 利用Block Bitmap找到一个空闲的Block，根据其在Block Bitmap中的位置可以得知这个Block的Block ID，因此能获取到这块空闲Block。
* **Block Bitmap本身的起始块位置必须是文件系统预先设定好的。**

### Super Block

* 固定的一个Block用来存储关于文件系统的一系列元信息
  * Block大小
  * ...
* **Super Block本身的位置必须是文件系统预先设定好的。**

### Hint

* **Block Layer 提供 $\ Block\_ID \rightarrow Block $ 的映射**
* **Block Bitmap提供 $\ Block\_ID \rightarrow Block空闲状态 $ 的映射**

## File Layer

### Inode

* 文件系统中存储的文件本质上是**任意大小的字节序列**，其大小会**根据写操作进行伸缩**，为了满足文件存储的要求提供一层Inode抽象，将**一个文件**的内容按照Block的大小**切分成多个Block**，以Block为单位**离散地存储**在文件系统中。
* **Inode**是负责记录文件内容存储在哪些Block中的一个磁盘数据结构。
* **一个Inode对应于一个文件**，Inode记录文件的元信息以及存储文件内容的Blocks对应的Block ID(**Block ID List**)。

```C++
// Inode
class Inode {
  InodeType type;		// 文件类型
  FileAttr inner_attr;	// 文件元信息
  uint32_t block_size;	// 存储Inode的Block的具体大小
  uint32_t block_pointers_number;	// Inode所在的Block中存储的Block_id索引数量
public:
  block_id_t* blocks;
}
```

* **Inode中采用一个Block_id_t数组来索引到具体存储文件内容的Block。**即在Block_id_t Array中，下标为 i 的Block ID索引到存储第 i 部分文件内容的Block。
* Inode中索引Block会采用多级索引。

![image-20241113003900954](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113003900954.png)

* 一个Inode 最多能支持的文件大小为：$\ 一级Block指针数 * Block\_Size + ((二级Block指针数 * Block\_Size)/Block指针大小) * Block\_Size + ... $

### Hint

* **File Layer提供 $\ Inode \rightarrow File \ Data \ Blocks \  \& \ File \ Meta $的映射**

## Inode Number Layer

* 文件系统中会存储大量的文件，而**每一个文件被抽象成一个Inode**，**采用Inode ID来唯一标识每一个Inode**，为了能够根据Inode ID高效地查找到对应的Inode结构存储在哪一个Block，需要一个Inode Table来保证能够通过Inode ID获取到Inode。(Notice: **Inode数据结构存储在具体的Block上**)

### Inode Allocation Bitmap

* 用**连续的Blocks**来存储**Inode Allocation Bitmap**，**表示某一Inode ID是否已经被使用**。

* 当要创建新的文件时，必须在Inode Allocation Bitmap中获取到一个**空闲的Inode ID来唯一映射用以抽象这个文件的Inode**。

* Inode Allocation Bitmap起始位置必须由文件系统预先设定好。

#### 静态 Inode Table

* Inode Table组织成一个Inode结构的数组，其中Inode ID为 i 的Inode结构就存储在Inode Table中第 i 个位置。

  Inode Table[i] = Inode ID为 i 的Inode

* Inode Table由文件系统预先分配好放在一个固定的位置，存储Inode Table的Blocks是连续的。

![image-20241113005645846](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113005645846.png)

#### 动态 Inode Table

* 根据文件系统最多能支持多少个Inode来生成一个Inode Table，Inode Table为一个Block_id_t的数组

  **Inode Table[i] = Block ID，指向负责存储Inode ID为 i 的Inode数据结构的Block**。

* Inode Table由文件系统预先分配好放在一个固定的位置，存储Inode Table的Blocks是连续的。

* 需要根据Inode ID获取到Inode时，根据Inode ID在动态Inode Table中找到Inode所在的Block的Block ID，获取到对应的Block，其中的内容就是Inode Structure。
* 当要存储新的文件时，在Inode Table中顺序遍历得到一个没有被分配的Inode ID即可（已经被使用的Inode其在Inode Table中对应的Block ID必定是有效的，即不是Invalid标记）。

![image-20241113152940039](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113152940039.png)

#### Hint

* **Inode Number Layer提供了$\ Inode \ ID \rightarrow Inode $的映射。**

## File Name Layer

### 目录

* 对于上层用户来说，并不知晓有关Inode等的具体信息，只知道**通过文件名（字符串）来获取到文件**，需要提供一个文件名到底层文件表示形式的映射，即**通过文件名来获取到Inode ID**。

* Directory中记录**<File Name , Inode ID>**的列表，表示文件名到Inode ID的映射。
* **Directory也作为文件的形式存储，因此文件系统中存储的文件包含普通文件、目录文件。**
* 给定一个目录Directory，读取Directory的全部内容，可以在其中搜索文件名到Inode ID的映射，具体方式就是顺序遍历Directory中的内容寻找匹配的字符串，最终得到Inode ID。

![image-20241118211419859](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241118211419859.png)

## Path Name Layer

* 将文件组织成目录树结构

## Absolute Path Name Layer

* 引入根目录

## 链接 Link

### Hard Link

* 直接在**目录中添加到一个具体文件的索引**，即在目录中添加一项$\ Hard \ Link \ Name \rightarrow Inode  \  ID$的**Directory条目**。在这个**目录中可以直接根据这个Hard Link的名称获取到对应文件的Inode，避免一层一层解析路径名、读取目录进行查找**。

* **Hard Link本质上是用多个文件名来索引同一个Inode ID。**

* Inode Structure中需要Reference_count字段来记录多少个文件名正在引用这个Inode，进行一个到该Inode的Link操作时，Ref_count++;进行一个到该Inode的UnLink操作时，Ref_count--，如果Ref_count = 0，意味着这个Inode不再被引用，即文件应该被删除，就会释放这个Inode Structure以及其所有Block Pointers指向的Blocks。

* Link意味着目录树会出现环，为了避免环，可以**禁止目录Link目录**。

  > ![image-20241113014600723](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113014600723.png)

* **为了避免丢失对Inode的引用，不能直接删除Directory，必须保证Directory中为空时才能删除Directory。**

### Soft Link / Symbolic Link

* 构建一个 $\ Soft\ Link \ Name \rightarrow File \ Path \ or \ File \ Name$的映射。
* 创建 Soft Link时，并不要求Soft Link链接到的文件名真正存在对应的文件，因为Soft Link本身只是建立一个$\ 字符串 \rightarrow 字符串$的映射。**当使用这个Soft Link时，会找出其映射的文件名，根据这个文件名进行文件查找。**

> ![image-20241113013246933](C:\Users\StrangeMoon\AppData\Roaming\Typora\typora-user-images\image-20241113013246933.png)

## 重命名

Rename(From_file_name, To_file_name)，操作会覆盖掉To_file_name对应的文件。

### Method 1

* UnLink(To_name)
* Link(From_name,To_name)
* UnLink(From_name)

### Method 2

* Link(From_name,To_name)
* UnLink(From_name)
