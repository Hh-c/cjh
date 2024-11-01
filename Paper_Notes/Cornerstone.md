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

问题：因为 K 只存储了叶节点的信息，没有内部节点的信息（虽然可以推出来），所以需要生成缺失的内部节点，使树可遍历。

解决：

1. Karras：可以根据 Karras 提出的方法，首先将一个 3D 物体的 SFC keys 作为一个二元基数树的叶节点（假设有 N 个），然后可以生成 N - 1 个二进制基数树的内部节点。然后再根据这棵二元基数树生成八叉树（实现方式类似 eq5-7 ）。

   猜测：Karras 的方法可能适用性更强，比如说当删除部分 K 中的空桶（即不包含粒子的节点）以提高树的紧凑性时，Karras 的方法仍可用。

   具体见论文 Maximizing Parallelism in the Construction of BVHs, Octrees, and k-d Trees

2. 本文：不构造二元基数树，当 K 是完整的时（即没有删除空桶），可以直接推导出K中的键和它们隐含包含的内部八叉树节点之间的解析映射。

本文 octree 数据结构解释：
1. n = n~i~ + n~l~（n~i~ 为内部节点数量，n~l~ 为叶节点数量）；

   n~i~ = (n~l~ - 1) / 7 （因为不删除空桶，所以每个节点孩子数为 0 或 8，所以满足等式）

2. NK[i] 存储了节点 i 的 SFC key，其实是存储了它的3D网格坐标；

3. CO[i] 存储了节点 i 的第一个子节点的索引，则节点 i 的子节点索引范围为 [CO[i], CO[i]+8)。如果 CO[i]=0，则 i 为叶节点；

4. LO[i] 存储了 octree 第 i 层的第一个节点索引。（root 是第 0 层，而最深是第 L 层，且需要一个结束标志，所以 1 + L + 1 = L+2）

示例解释：

1. 节点内数字对应 NK，格式为 1 oct-digit oct-digit...；
2. 节点左边数字为节点索引；
3. 节点右边数字对应 CO；
4. LO = {0, 1, 9, 17, . . . , 17}.

生成过程解释：
下面数字为 0-1 串的是 B，不是 0-1 串的，如果没标 ~O~，则为 D

1. 第（1）步中的转换，对倒数第 a 层（从 0 计数），最前面加 placeholder-bit，右移 3a bit。注意这里 NK 内部元素的顺序还是中间状态，要到第（3）步排序后才是最终的；

2. 第（2）步中根据 K[i] 和 K[i+1] 异或结果的前导零数量 d 来判断，此时只有在出现 011 和 100 （即某一层的中间）的时候才可能，此时把具有 011 的 K key 作为其父节点的 (NK?) key。

   其实感觉也不是直接拿来用，而是在经过第（1）步处理后才是父节点的 NK key，更进一步，其实倒数第 a 层（从 0 计数）任一 K key 右移 3(a+1) bit 后，剩下的就是其父节点的 NK key，所以只要是这层的 K key 都满足，只是本文使用的方法恰好找的是中间的 K key。比如对于 fig2 中，NK key 123~O~, 124~O~ 原来的 K key 分别为 19(010011), 20(010100)，此时 d = 3，满足 mod(d,3)=0，将 010011 右移 3 bit 为 2~O~(010)，加上 placeholder-bit 为 12~O~，即为其父节点的 NK key 12~O~，且在第 3/3=1 层。

3. 𝛿 函数还没看懂

### 6 DISTRIBUTED OCTREES AND DOMAIN DECOMPOSITION

1. DISTRIBUTED OCTREES（如何处理分布式情况）：为了处理分布式的情况，需要调整，𝑓~glob~  = 𝑓~𝑟~ ◦ 𝑓 ，其中 𝑓~𝑟~ 计算了 N 的元素级全局和（即每个处理器上分别处理一部分粒子，𝑓~𝑟~ 将同一个节点中的粒子数加起来）。通过不断交替使用 𝑓~glob~ 和 *𝑔* ，最终每个节点都会收敛，生成平衡的基石数组；
2. DOMAIN DECOMPOSITION（如何生成分布式情况）：使用全局域分解将粒子 SFC 分成多个部分交个多个处理器并行处理，其中需要的全局集体通信为 O(𝑁~tot~log𝑁~tot~) ；对 SFC 采样等方法则为 O(𝑁~tot~^2^)。

### 7 LOCALLY ESSENTIAL OCTREES
