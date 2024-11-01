## A Parallel Hashed Oct-Tree N-Body Algorithm

[A Parallel Hashed Oct-Tree N-Body Algorithm](https://www.cs.umd.edu/class/spring2021/cmsc714/readings/Warren-nbody.pdf) 

### 0 Abstract

本文介绍了一种自适应N体方法。

目的：计算任意分布体的力；

特点：（1）时间复杂度 NlogN（N为粒子数）；（2）精度高，且用户可定义参数调节；（3）使用 key 标识 cell，通过哈希表查找地址，而不是使用指针。

### 1 Introduction

N体模拟简介：N体模拟是研究复杂物理系统的基本工具。具体可以分为两种：（1）物理系统可以直接用N体建模；（2）不能直接用N体建模，此时是统计性的，表示系统的相空间密度分布。N越多，结果越精确，但是所需的计算资源往往超出当前的能力。

因为相互作用是两两发生的，所以为计算的时间复杂度为 O(N^2^) 。

优化：

1. 使用晶格（lattice）：多重网格法，傅立叶变换等方法降低时间复杂度到 O(N)，但是也存在动态范围受限的缺陷；

2. 不使用晶格：有的方法使用截断展开来近似多个体的贡献为单一相互作用，而不是引入晶格，结果复杂度通常为 O(N) 或 O(NlogN)。分析并保持相关变量不变会导致不同的放缩结果，但一般都显著好于 O(N^2^) 。

   基本思想：基于截断序列近似的N体算法的基本思想是将体划分为多个部分（常用空间树数据结构），使级数近似能够用于这些部分（目前还不知道咋用的），同时保持每个粒子的力（或其他量）的精度。

   MAC（multipole acceptance criterion）：选择与哪些单元进行交互以及哪些由于不够精确而被排除。

### 2 Background

#### 2.1 Multipole Methods

不同的多级法使用了不同的数据结构和不同的MAC控制策略。如Appel的方法使用二元树数据结构，根据交互的 cell 大小来控制 MAC；Barnes-Hut (BH) 算法使用了八叉树，根据粒子到 cell 质心的距离控制 MAC；Greengard 和 Rokhlin 的 fast multipole method (FMM) 有明确的最坏情况边界 𝜖，当多级展开到 𝑝=log~𝑧~(𝜖) 时，保证满足该界限。

#### 2.2 Analytic Error Bounds



### 3 Computational Approach

问题：并行树码在分布式内存机器上的应用存在一些问题，特别是在数据依赖的情况下，难以预先确定所需的非本地单元数据（并行算法需要在树遍历开始前确定本地必要的数据），此时 MAC 难以发挥作用。

解决：本文引入了 hashed oct-tree (HOT) 方法，本方法没有像传统的方法使用指针来表示一个分布式树数据结构，而是使用键和哈希表。为每个 cell 分配一个 key，通过简单的位运算就可以生成 cell 父/子节点的 key，通过哈希表查找可以将 key 转为内存地址，而且是统一编址。

拓扑对比：传统树数据结构使用指针表示树的拓扑，HOT 方法将树的拓扑隐含在将单元的空间位置和层级映射到键的过程中

#### 3.1 Key construction and the Hashing Function

key：

1. 定义：将 key 定义为将 d 个浮点数（d维空间中体的坐标）映射到一组 bit 的结果。
2. 映射：
   1. 将浮点数转为整数；
   2. 将 d 个整数的每位交错的组合为一个 key（具体见下 fig.4）。

除去原点和坐标系的选择，这种表示与Morton排序（也称为Z排序或N排序）是相同的。



3. place-holder bit：因为这些 key 是存储在相同大小的内存中，所以每个 key 的长度其实在内存上看是相等的，只是有效位长度不同，比如 key 的有效位为 00 ，而此时内存中别的 bit 也为 0，为了区分，在 key 前加个 1，变成 100。
4. 这种 key space 的好处：
   1. 容易获取子节点：将父节点 key 左移 d 位，然后加上 0～2^d^-1 即可得到子节点 key；
   2. 常数时间访问：通过 key 可以以 O(1) 的复杂度访问到对应的对象，如果是用指针，则需要从 root 开始向下遍历，需要 O(logN)。



hash：

1. hash function：本文使用非常简单的哈希函数 AND，将 k-bit key 与 bit-mask 2^h^-1 相于，得到最低 h bits。（实际上是将浮点坐标映射到键的过程中，执行了通常意义的哈希操作）；
2. 冲突处理：使用链式处理。对于较上层节点，k ≤ h，此时不会发生冲突；对于较下层节点，k > h，此时会发生冲突，但是对于并行机器，一般会导致冲突的节点会被分配给不同的处理器，这样也不会发生冲突。
3. 结构：哈希表条目大致包含指向数据的指针，key 等内容
4. 好处：
   1. 统一寻址：每个数据单元分配一个唯一的键，可以通过请求一个键来访问非本地数据；
   2. 缓存机制：可以使用哈希表来缓存非本地数据，提高内存性能。


