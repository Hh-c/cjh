## Fast, Memory-Efficient Cell Location in Unstructured Grids for Visualization

Abstract 

问题: 对于在非结构网格上的快速定位单元, 一般方法使用各种空间细分方法, 但是一般是内存密集的, 且扩展性差.

解决: 本文提出了新的数据结构和方法来解决该问题, 且应用于变量插值问题, 该方法内存高效、速度快且数值鲁棒性强，适用于处理大型数据集，并在CPU和GPU上都有很好的应用前景。

### 1 INTRODUCTION

问题: 很多可视化算法都需要在 UG(Unstructured Grid) 上插值 (且数值积分也是依赖于插值), 但是在 UG 上插值存在两个问题: (1) 需要考虑现代 AUG 中大小差异很大的单元; (2) 内存开销. 现在的方法一般都没针对 UG 优化.

解决: 本文提出了 celltree 来解决这个问题. 其基于 bih (bounding interval hierarchy), 性能好, 内存效率高, 可以处理非常大的复杂数据集, 并且适用于CPU和GPU。

### 2 BACKGROUND AND SCOPE

setting: ug 以有序的顶点列表形式描述，不同类型的单元被定义为索引的元组。

cell location 影响因素: (1) 某些 ug 顶点的高度不均匀空间分布; (2) 某些 cell 有很大纵横比或非凸性, 使得从数值角度处理网格变得复杂;

目标: 高性能

#### 2.1 Cell Location

Cell Location 一般步骤: (1) 通过一个空间数据结构来快速缩小候选 cell 数量; (2) 依次判断候选 cell

目标: 第 (2) 步的开销比较大, 所以需要在第 (1) 步时尽量减少候选 cell 数量. 

方法演化:

1. 使用 octree, 但是在非均匀顶点分布情况下效果不好;

2. 使用 kd-tree, 虽然会有更深的树和内存开销的代价, 但是使用 *Matrix \*Trees* 来存储可以有效降低内存开销 (使用了 CSR, compressed sparse row) 

   上述两种树都存在 cell 索引冗余的问题, 即一个 cell 覆盖了多个叶节点, 此时需要重复存储索引

3. Langbein等人提出了新的方法, 其基于 ug 的顶点进行空间细分，以快速定位靠近插值点的网格顶点(不懂)

#### 2.2 GPU-based Vector Field Visualization

背景: 现有的基于GPU的矢量场可视化方法大多局限于 sg (structured grid), ug 很少被使用, 就算有也具有一定的局限

本文方法: celltree 直接应用于GPU, 可以处理任意 ug, 也具有良好的存储和鲁棒性. 且不针对特定的访问模式, 所以支持广泛的依赖于 ug cell 定位的可视化算法.

#### 2.3 Ray Tracing

空间细分数据结构在光线追踪领域也很重要 (虽然不是用于 cell location, 而是加速光线-三角相交测试的), 本文方法借鉴了光线追踪中使用的 skd 树方法和表面积启发式算法等思想.

#### 2.4 Contribution

1. 通用性: 支持通用的非结构化网格, 并且可以处理任意 cell 类型, 适用于各种数据集类型和格式;
2. 数值鲁棒性: cell location 不会受到单个单元格不良数值特性的影响;
3. 支持多种算法和并行计算;
4. 内存开销可控;

### 3 THE CELLTREE

下文中假设 ug 以大小为 N 的 cell 列表的形式给出, 对于 cell c~i~ (i = 1, ..., N), 每个维度 d = 0, 1, 2 的下限 cmin^d^~i~ 和上限 cmax^d^~i~ 是已知的或可以确定的. 

#### 3.1 Basic Structure

数据结构: 本质上是 BIH, 构建过程见下图

优点: 因为是对象划分, 所以不像空间划分有时需要存储重复的 cell 索引, 可以就地划分且内存开销更小.

遍历: 给定一点 x, 从根开始比较, 如果 x~d~ ≤ L~max~, 则遍历左子树; 如果 x~d~ ≥ R~min~, 遍历右子树. 遍历到叶节点时，依次检查叶节点内的 cell 是否包含点 x. 如果是, 则遍历终止并返回相应的索引. 否则继续遍历到其他叶节点. 如果找不到包含该点的 cell，则它不包含在由网格单元描述的域内.

#### 3.2 Construction

下面等式的序号和论文中不同, 这里不使用论文中的序号.

**成本函数 C:**

假设一次 cell-point 包含测试的开销为单位常数, 则对于一个叶节点 (其 cell 索引列表为 I, cell 索引数为 N~I~), 其 cell location 的开销为
$C_I = N_I$
如果将 I 划分为 L 和 R (还有对应的孩子节点). 设 P(L) 表示插值点在 L 中的概率, 则 P(L)N~L~ 表示查找左边的开销期望. 设 C~trav~ 表示下降一级层次结构的开销. 总开销为
$C_I = P(L)·N_L +P(R)·N_R + C_{trav}$
如果点分布平均, 则可用体积代替概率
$C_I = vol(L)·N_L +vol(R)·N_R + C_{trav}$
我们的目标是最小化开销 C, 可以看到开销来自两方面: (1) 分割叶节点的局部开销; (2) 层次结构中下降一级的开销 C~trav~. 但是全局最小化相当于穷尽了, 不太现实, 所以忽略 C~trav~.

体积 vol(L/R) 其实与 L~max~/R~min~ 是成比例的, 所以在每个可能的分割维度d = 0,1,2上局部最小化成本函数
$C = L_{max}^d ·N_L −R^d_{min}·N_R$
**分割方法:**

由上可知在不考虑 C~trav~ 的情况下, 最小化 C 的关键在于怎么分割节点

1. *split-middle*: 选择 I 的包围盒在维度 d 上的中点来划分. 虽然这能保证容量的平衡, 但是容易导致 N~L~ 和 N~R~ 的极端不平衡;

2. *split-median*: 先将 cell 沿维度 d 排序, 然后将 I 分成 L 和 R 大小相等的两部分. 这虽然能使树平衡且深度最小, 但如果 L~max~ >> R~min~ , 可能导致过度的遍历, 因为很多叶节点重叠;

3. 贪心: 每次分割都选择能最小化 C 的位置, 但是这样对每个分割平面的位置计算都是计算密集的, 不可行;

4. 本文 (桶排序): 将 I 排序到一小组 n~b~ 个等距离的桶中 (这些桶沿 d 覆盖了 I). 排序时遍历 I 一次, 每个 cell i 根据以下公式分配到桶 b 中.  $I_{min}^d$ 和 $I_{max}^d$ 表示所有包含在 I 中的 cell 的边界. 每个桶记录了其中 cell 数量和插入其中的 cell 的包围盒.
   $b = ⌈n_b · \frac{cmin^d_i +cmax^d_i}{I_{min}^d+I_{max}^d}⌉-1$
   评估 n~b~ -1 个分割桶的平面 (见下公式), 选择最小化公式4的平面.
   $p^d_j = I_{min}^d+ \frac{j+1}{n_b}(I_{max}^d-I_{min}^d)$
   实际上就是使用桶的边界而不是 cell 的边界来分割, 这样可以减少评估 C 的成本. 具体过程如 fig. 3 所示



   特殊情况: 当所有的 cell 都落到一个桶时, 会出现所有 cell 被分配到 L, 而 R 为空的情况. 此时回退到 split-median 方法. 

   注意 n~b~=2 时就是 split-middle

5. 各种方法示例见fig.4. 在优化 eq3 时会导致一个节点的孩子节点重合最小


### 4 IMPLEMENTATION

为了提高空间效率，实施以下设计限制，旨在在典型工作站上使用本文 cell location 方案

#### 4.1 Data Layout and Construction

1. 内存布局: 树节点结构如下图, 大小为 12字节



   dim 表示 dimension, 即维度 (0,1,2), 使用 3 表示叶节点 

   如果是内部节点, 使用 Lmax 和 Rmin 表示对应孩子节点的边界平面

2. 并行

   原因: 子节点的分割可以独立进行, 且大部分计算时间用于扫描单元格列表以确定最佳分割

   实现: 必须分割的叶节点记录在队列中，多个线程从队列中获取叶节点，分析并分割相应的叶节点，如果需要进一步分割，则将子节点放回队列中. 注意为了保存结构的一致性, 队列访问和树数组修改需要实现线程间同步

   问题: 因为并行异步问题, 树数组中节点索引无序, 通过构建完后排序解决

3. 优化: 因为 3.2 Construction 过程严重依赖于 cell 的包围盒, 所以可以通过缓存来优化, 但是在处理大网格时会有问题, 所以该优化可选

#### 4.2 Traversal

简介: BIH 遍历采用深度优先遍历, 优先遍历边界平面与所寻求点距离较大的子树

#### 4.3 Point-in-Cell Test and Interpolation

当查找到叶节点时, 需要进行 Point-in-Cell Test, 此时可以将 point 的全局坐标转换为本地坐标系来简化 inclusion test. 

优点: 由于 celltree 数据结构不依赖于 point-in-cell test 的特定性质，因此它可以适应任意单元类型, 且该实现可以轻松扩展以支持例如高阶元素或其他依赖于单元定义的插值方案