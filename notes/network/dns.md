# DNS

ip address本身具有结构性

host name

为什么不能只用ip 需要域名

user-friendly

distributing responsibility

域名划分若干段

root zone由ICANN管理 负责一级域名管理

dns hierarchy



解析域名 从后往前

域名global context-free

每个zone有多个name server

recursion

------

## Name Scheme

同一个对象，在不同的层次会有不同的名称。

### 作用

Retrieval 根据名称获取到某个对象

Sharing 通过名称来共享某个对象

Hiding 通过一个含有高层语义的名称来封装底层的语义

User-friendly Identifier：

Indirection：进行解耦，不需要与下层的细节进行绑定，拥抱变化。用一个名称可以映射到多个下层的对象。

### 组成

* Name-Mapping Algorithm
* Namespace
* Values
* Context/Global url不需要context inode就需要context

### 操作

* 绑定：创建一个Name到Value的映射
* 解绑：删除一个Name到Value的映射

### Name-Mapping Algorithm

* Table Lookup：按照表进行线性搜索。eg.文件系统中在Directory查找文件名到Inode Number的映射。
* Recursive Lookup：按递归的关系进行查找。eg.shell中执行 ls /usr/local/.....
* Multiple Lookup：并列地进行查找。

## Content Delivery Network(CDN)

## 
