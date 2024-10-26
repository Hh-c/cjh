## Cornerstone: Octree Construction Algorithms for Scalable Particle Simulations

Cornerstone 是一种用于八叉树构建的新方法，适用于无网格数值模拟中的全局域分解和粒子相互作用。它基于 3D 计算机图形算法，扩展至高性能计算系统。Cornerstone 可以在 GPU 上运行，显著加快树构建速度，并消除 CPU 与 GPU 之间的数据传输。它在 N 体方法中处理短程和长程相互作用方面表现出色，比如 Barnes-Hut 和快速多极方法。

[链接](https://arxiv.org/pdf/2307.06345) 
### 3 SPACE FILLING CURVES AND OCTREES

定义：Space filling curves (SFCs) are continuous functions that map the unit interval into an 𝑛-dimensional hypercube.

三维粒子模拟更适合离散版的SFCs，下面以此为例。

映射：key space [0 *. . .* 2^3𝐿) -> a grid of 3D integer coordinates [0 . . . 2^𝐿) × [0 . . . 2^𝐿) × [0 . . . 2^𝐿) 
	L 表示细分的层数，2^L 即表示细分后有多少个小的空间

表示：在3D中使用 3L bits 表示一个 SFC key。如 Morton Z-curve 的一个 key 可以表示为 𝑘 = 𝑘_1𝑘_2 . . . 𝑘_𝐿，其中 𝑘_i 为8进制数，可以直观理解为第 i 次细分被分到哪了

位于第 i 细分层次的八叉树节点包含能够编码到键范围 [k, k + 8^{l-i}] 内的 3D 坐标，其中 k 满足 k mod 8^{l-i} = 0。其中 8^{l-i}=2^3(l-i)，表示其包含剩下 i+1 ---> l（即l-i）次细分的结果，最后每个方向有 2^{l-i} 个节点，一共 (2^{l-i})^3 个节点

### 4 LEAF NODES OF BALANCED OCTREES

通过不断的划分，使每个叶节点内粒子的数量小于 N_cirt 。

定义：

1. K 是存储了叶节点 SFC keys 的基石数组，长度为  n_l+1，定义了一个八叉树，有 n_l 个叶节点，最深 L 层；
2. 当 𝑁_𝑖 ≤ 𝑁_crit （即每个叶节点中粒子数量小于 𝑁_crit ），树被认为是平衡的
3. P is the sorted array of particle SFC keys obtained by encoding the particle coordinates into SFC keys followed by a radix-sort.
   即 P 是存储粒子 SFC keys 的数组
5. N is the histogram of particle SFC keys P for the 𝑛_𝑙_ bins defined by consecutive keys in K.
   N 中的元素 N_i 存储了叶节点 K_i 中粒子 SFC keys P 的数量

定义两个方法：其中 f 计算每个叶节点的粒子数，g 使树平衡

其中 𝑔 可以细分为 3 步：𝑔 = 𝑔_3 ◦ 𝑔_2 ◦ 𝑔_1 

𝑔_1：

定义：O is an auxiliary array that encodes the rebalance operation that is to be performed on each leaf node, which can be either removal, subdivision, or no change of the leaf node.
	即 O 存储了对每个叶节点的操作（用于使树平衡）

- O_i=8：𝑁_𝑖 > 𝑁_crit，K_i 需被细分
- O_i=0：叶节点合并后粒子仍少于 𝑁_crit，K_i 需被合并
- O_i=1：K_i 无需调整

根据每个叶节点 K_i 中的粒子数 𝑁_𝑖 ，判断进行什么操作 O_i 

𝑔_2：

定义：O′_𝑖 is equal to the index of the old node K_𝑖 in the rebalanced node array K′, and O′_𝑛𝑙 is equal to the total number of rebalanced leaf nodes in K′
	即 O′ 存储了原来叶节点 K_i 重平衡后在 K′ 中的索引

求前缀和：

假如有 O = [1,8,1,0,0,0,0,0,0,0,8]，则 O′ = [1,9,10,10,10,10,10,10,10,10,18]

𝑔_3：

两个式子是赋值的，从右边往左看，其实就是将原来叶节点 K_i 放到了 K′ 中，位置由 O′_i 决定

本文的八叉树构建方法相比于传统的自顶向下、层次顺序八叉树构建方法没有优势。主要是在多步中重复构建八叉树，且每一步中粒子位置仅发生微小变化时，本方法比较有效。

### 5 FULLY LINKED OCTREES

因为 K 只存储了叶节点的信息，没有内部节点的信息（虽然可以推出来），所以需要生成缺失的内部节点，使树可遍历。

首先将一个 3D 物体的 SFC keys 作为一个二进制基数树的叶节点（假设有 N 个），然后可以生成 N - 1 个二进制基数树的内部节点。然后再根据这棵二进制基数树生成八叉树（实现方式类似 eq5-7 ）。

