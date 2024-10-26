## Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees

High Performance Graphics (2012)

C. Dachsbacher, J. Munkberg, and J. Pantaleoni (Editors)

[链接](https://research.nvidia.com/sites/default/files/pubs/2012-06_Maximizing-Parallelism-in/karras2012hpg_paper.pdf) 

### 0 Abstract

传统方法沿着SFC的顺序构建树层次，限制了并行处理。而新方法通过并行构建整个树，提升了效率。

### 1 Introduction

背景：通用GPU计算技术带来了多种实时构建BVH、八叉树和k-d树的方法，实时构建树的方法可以分为两类：专注树的质量；专注构建树的速度。

问题：当前专注速度的方法的主要缺点是顺序生成节点层次，这不仅限制了并行度，而且其广度优先输出也没有利用数据局部性。

解决：本文方法并行度好，且结果是严格深度优先。方法先使用原地算法构建二进制基数树，再利用该树构建其他树。

### 2 Background

**Morton code.** 包含在三维单元立方体中的给定点的 Morton 码由位串 X_0Y_0Z_0X_1Y_1Z_1... 定义，其中点的 x 坐标表示为 0.X_0X_1X_2（y、z类似）。任意3D基元的 Morton 码可以根据其轴向边界框（AABB）的质心来定义。在实践中，Morton码可以被限制为30或63位，以便分别存储为32位或64位整数。

**Binary radix trees (Patricia tree).** 对于一组用 bit 串表示的 keys（k_0 ~ k_n-1），Binary radix trees 是它们共同前缀的层级表示，叶子节点表示 keys。内部节点表示其对应子节点的最长公共前缀。注意与 prefix tree 区分，prefix tree 对每个公共前缀都存储一个内部节点。有相同 keys 时也要注意，section 4讨论。

本方法考虑有序树，即每个节点的孩子满足字典序。使用 δ(i, j) 表示 key k_i 和 k_j 之间 keys 的最长前缀长度，显然从 k_i 开始的一些 keys 的这一位为0，后面到 k_j 的 key 的一位这为 1。记最后一个这位为 0 的 key 的索引为 split position ，记为 γ ∈ [i, j−1]，满足 δ(γ, γ+1) = δ(i, j)。


### 3 Parallel Construction of Binary Radix Trees

一般方法：从 root 开始逐位查找 spllit position ，查找到了就创建孩子节点，递归下去

本方法：

1. 简介：为了实现并行构建，在节点索引和 keys 之间建立联系。这样不需要之前节点的信息就可以查找内部节点的孩子；

2. 节点布局：使用 L 和 I 分别存储叶子节点和内部节点。root 存储在 I_0，每个节点的左孩子是 I_γ 或 L_γ，右孩子是 I_γ+1或 L_γ+1；

   特征：这种布局可以让每个内部节点的索引与它覆盖的第一个 key 或最后一个 key 的索引一致。

   注意 I 本身只是一个存储了 N-1 个元素的数组，只是因为人为定义的布局（数据结构），让 I 中元素的索引具有了特殊的意义，即对应其覆盖的叶节点范围的一端。

3. 算法：为了构建 Binary Radix Tree，需要知道每个内部节点覆盖的 keys 范围，而根据布局已经可知范围的一端。通过查找 I_i 附近的 k_i-1, k_i, k_i+1 来获取另一端。首先获取方向 d，d = +1 表示范围从 i 开始，d = -1 表示范围到 i 结束。则 k_i 和 k_i+d 属于 I_i，k_i-d 属于 I_i-d。 

   Line3：使用 δ(i,i+1) − δ(i,i−1) 来判断方向 d。因为划分是从上往下的，所以 δ 大的划分位置在下面，δ(i,i+1) − δ(i,i−1) < 0 表示 γ = i，将 k_i 和 k_i+1 分开，则 k_i 显然为左孩子，为节点 I_i 覆盖范围的末尾；反之 γ = i-1，将 k_i-1 和 k_i 分开，则 k_i 显然为右节点，为父节点 I_i 覆盖范围的起始；

   Line5：设置 I_i 覆盖范围内 keys 的公共前缀和的下界为 δ_min = δ(i,i−d)，则任意 k_j ∈ I_i，满足 δ(i, j) > δ_min。因为 k_i 和 k_i-d 先被分割，而分割顺序又是从上往下的，所以 δ(i,i−d) 肯定小于 I_i 覆盖范围内 keys 的公共前缀和长度；

   Line6-8：为了查找满足 δ(i,i+ld) > δ_min 的最大 l，首先查找满足大于 l 的最大二次幂上界 l_max；

   Line10-14：找到 l_max 后，在  [0,l_max −1] 上使用二分查找获取 l。思想是从最高位开始依次考虑 l 的每一位，满足 line12 的不等式赋1，否则赋0。最后得到另一边界为 j = i + ld；

   Line16-21：找到 split position。使用 δ(i, j) 可以获取 I_i 对应的前缀和长度，记为 δ_node。与上类似，可以使用二分查找依次判断步长 s ∈ [0,l − 1] 的每一位，直到找到满足 δ(i,i + sd) > δ_node 的最大 s。最后得到分位点 γ = i + s·d + min(d,0)；

   Line23-24：获取 I_i 覆盖 keys 的范围。给定 i，j，k，可以知道 I_i_ 的孩子节点覆盖了 [min(i, j), γ] 和 [γ + 1,max(i, j)]，对于每个孩子，比较它们的覆盖范围起始和结束来判断是否为叶子节点，然后在索引 γ 和 γ+1 处引用对应的节点（L_γ 或 I_γ）。

4. GPU 实现。该算法在GPU上通过单个内核启动实现，每个线程负责一个内部节点，使用逻辑异或和前导零位计算来评估键之间的差异；

5. 时间复杂度。对于覆盖了 l 个 keys 的节点，每个循环最多迭代 ⌈log_2l⌉ 次，而 l ≤ n ，且节点数为 n-1，所以最坏时间复杂度为 O(nlogn)，最坏时树高与 n 成正比，而一般情况下是与 logn 成正比。对树高 h，有更严格的边界 O(nlogh)。

### 4 BVHs, Octrees, and k-d Trees（等了解完这些树之后再补上）

**BVHs.** 为一组3D基元构建BVH的步骤如下：（1）根据每个基元的质心分配Morton码；（2）对Morton码进行排序；（3）构建二进制基数树；（4）为每个内部节点分配边界框。本文方法对步骤3和4进行了改进。





