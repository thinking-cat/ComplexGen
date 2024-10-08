# ComplexGen

## ComplexGen的流水线

1. ComplexGen网络预测：网络将**点云**作为输入进ComplexGen的神经网络内，输出为{点、边、面、边点、面点、面边、嵌入空间}
2. Complex提取：
    1. 去除步骤*1*中冗余的输出
    2. 运用组合优化算法提取Complex结构，以形成BRep流形模型
    3. 输出：合法的BRep流形
3. 几何改良：
    1. 将*2*中输出的*点*和*拓扑邻接关系*作为输入
    2. 通过*拓扑提示*进行正则化，减少输入的点中存在的*噪声*
    3. 由输入的点与拓扑关系，构建BRep模型输出

![Pipeline](./images/pipeline.png)


## 1. 网络

### 输入
网络的输入是*3D点云*及其离散的*体素*

每一块*体素*都是一个边长为单位1的正方体，通过 $128^3$ 个小正方体组成整个模型所在的空间，因此每一个*体素*的数据是 $S_p \in [0, 127]^3$ 的位置坐标，一个模型的数据即为： $S_p \in \mathbb{R}^{S \times 3}$ ，其中 $S$ 为非空体素的数量。

![Voxels](./images/voxel.png)

> 图中灰色部分为非空体素

### 网络结构
网络分为两部分，Encoder和Tri-path transformer decoder

1. Encoder(Sparse CNN)：

    1. 将 $128^3$ 的空间网格信息通过卷积与池化操作提取至 $16^3$ 
    2. 提取出每一个**非空体素**Voxel的 $d=384$ 个特征。即输出为 $S_f \in \mathbb{R}^{S \times d}$ (额外信息)

2. Tri-path transformer decoder：
    1. 输入： 
        1.  $S_f$ ：Encoder的输出
        2.  $S^{'}_p$ ：每一个体素的正弦位置编码 $S^{'}_p=PE(S_p)$ ，体素在 $xyz$ 每一个方向上的坐标都被编码为了一个128维的向量，因此位置编码为 $3 \times 128=386$ 维向量。
        3.  $Q_v,Q_e,Q_f$ ：点、边、面经过嵌入层学习到的特征向量
    2. 网络子层：
        decoder网络是由许多一样的子网络所组成的，对于第 $t$ 个子网络
        1. 输入为 $H_t$ 和 $2.1$ 所述的输入
        2. 输出为 $H_{t+1}$
        3. 网络结构：
            ![Sublayer](./images/sublayer.png)
            > 其中 $LN(\cdot)$ 表示线性层， $CA(\cdot)$ 表示交叉注意力层（Cross Attention）， $SA(\cdot)$ 表示自注意力层（Self Attention）
    3. 网络输出：
        对于*点、边、面*三种类型的元素
        1. 某个元素合法性的概率 $F^{val}$
        2. 某个元素属于某个类型的概率（如：边的类型有直线、圆……） $F^{type}$
        3. 某个元素是否开放的概率 $F^{open}$ 
        4. 某个元素在各自类型的嵌入空间的坐标 $P$ 
        5. 某两个不同类型元素之间的邻接关系 $F^{topo}_{fe}, F^{topo}_{ev}, F^{topo}_{fv}$

![network](./images/network.png)

# 后处理细节
## NMS(Non-Maximum Suppression)
NMS去重的步骤：
1. 提取足够可信的网络预测结果：规定只有预测的 $validness >= 0.5$ 的点、线、面算作**合理**的预测
2. 在合理的预测结果中，对于元素 $q$ ，如果存在重复，则满足：
    1. 存在同样类型的 $q^{'}$ ，即同样是点、线或者面。
    2. $q^{'}$ 有着相同的拓扑结构
    3. $q^{'}$ 有着相似的几何结构，可以利用*倒角距离*对输入进行计算，如果小于某个*阈值（0.05）*，则认为几何上相似

## Combinatorial Optimization

因为文章使用了Gurobi作为*组合优化*的软件，因此其输入是一个巨大的用来表示约束条件的稀疏矩阵，此处解释其变量意义：

1. `b_ub`: 约束上界
2. `b_lb`：约束下界
3. `data`：数据的系数
4. `rows`: 稀疏矩阵中**约束**所在行号
5. `cols`：稀疏矩阵中**数据**所在列号
6. `bkc`：mosek中的boundkey，用来表示约束形式

以下为例子 

假设约束条件为 $EV(i,j) < E(i)$

则实际上约束条件将被改写为 $-\inf < EV(i,j)-E(i) < 0$ 的形式。

因此
1. `b_ub` 须加入0， `b_lb` 须加入 `-inf` 。
2. `col`加入 $EV(i,j)$ 在变量 `c` 中所对应的下标，即`nv + nc + nf + nv*i + j`，而 `data` 加入`1`，即 $EV(i,j)$ 的系数，`rows` 因为表示现在是第几条约束条件，因此加入 `cur_row`
3. 对于 $E(i)$ ，类似的，在变量 `c` 中所对应的下标，即`nv + i`，而 `data` 加入`-1`，即 $E(i)$ 的系数，`rows` 因为表示现在是第几条约束条件，而现在还在构造同一条约束条件，因此加入 `cur_row`
4. `bkc` 中加入 `mosek.boundkey.up` ，表示这条约束是仅有上界，而有无穷下界的约束

构造完约束后，将会交给Gurobi去处理。

> 顺带一提， `c` 是一个巨长无比的单列表，其中存储的就是论文中的优化目标，一共有2个优化目标，其一是Combinational Optimization中直接列出的 $wF_{topo}+(1-w)F_{geom}$ ，另一条则是满足基本的水密条件，即——一个边连接两个面、一个开放/闭合的边有两个/一个端点、一个面有着闭合的边的回路。

# 组合优化约束条件

组合优化条件一共要满足如下条件：

$$ \begin{aligned} &\sum_i FE[i,j]=2E[j], \quad j\in[N_e] \\\\ &\sum_j EV[i,j]=2E[i]O[i], \quad i\in[N_e]  \\\\ &FE\times EV = 2FV  \\\\ &\begin{cases}FE[i.j]\le F[i] \; (\le \sum_j FE[i,j]), \quad \forall i,j \\\\ EV[i,j] \le V[j] \le \sum_k EV[k,j], \quad \forall i,j \end{cases}  \end{aligned}$$

## 合法的链复形
前三条约束是为了满足优化结果一定是一个**合法的**连复形，从数学语言转换过来就是：

1. 一条边一定与两个面相连
2. 一开*开放 / 闭合*的边一定有 一个/两个 端点
3. 一个面一定有一个闭合的边界环

其中，第三个条件之所以是这样描述的，是因为链复形一定满足边界算子 $\partial_1\circ \partial_2 = 0$ ，即既然一个面一定有*闭合*的边界，这个闭合的边界就是一组边组成的*闭合*循环，闭合的边的循环并不存在进一步的边界。这点与 (2) 不太一样。

## 二元与一元拓扑项的依赖关系
最下面的约束条件保证了网络的输出结果中，一元项（点、线、面、开放性的合法性）与二元项（各元素间的邻接关系）的相互依赖。

依据原论文解释，从直觉上，就是在说如果存在一个*面*，那么一定也就存在与某些边的邻接关系，并且如果存在一个边，那么一定存在与某一个（闭合），或者某些点的邻接关系，因此：

1. 如果一个面消失了，那么其所属的面边邻接关系一定会消失，因此需要保证面存在的合法性优于面边邻接关系的合法性( $FE[i,j] \le F[i]$ )
2. 如果一个点消失了，那么其所属的边点邻接关系一定会消失，因此需要保证点存在的合法性优于边点邻接关系的合法性( $EV[i,k] \le V[j]$ )
3. 如果一个面存在，那么一定存在对应的面边邻接关系，当然如果所有的面都是闭合的情况除外（ $F[i](\le \sum_j FE[i,j])$ ）
4. 如果一点存在，那么一定需要对应的的边点邻接关系，这一条理解上较为困难，必须结合链复形的合法性与流形的拓扑考虑，点不可能单独存在，边也不可能，因此有点必有边的邻接关系( $V[j] \le \sum_k EV[k,j]$ )

## 二次项约束条件的转化
上述的保证**合法的链复形**的约束条件中，存在二次项的约束条件，即：

$$\sum_j EV[i,j]=2E[i]O[i], \quad i\in[N_e] $$

$$FE\times EV = 2FV $$

二次项的优化是非常难做的，最好的方式是转化成一次项，此外，因为变量的取值都在 $[0,1]$ 之间，因此:

$$x\in[0,1],\;y\in[0,1],\;xy\in[0,1]$$

取一个中间变量 $z$ ，且 $z\in[0,1]$ ，则有：

$$z\le x, \quad z\le y, \quad z \ge x+y-1$$

因此带二次项的约束条件就被转化为了上述三个一次项的约束条件。

## 论文中的约束项
即便把非流形相关的约束全部删去，面对BSpline曲线，该乱的还是乱，这应该是网络本身对于曲线的生成与识别就不太行

尝试不存在BSpline曲线

如果不存在BSpline曲线，那**二元与一元拓扑项的依赖关系**这个额外约束本身并没有什么用。

如果一个点连接许多条边，也会导致结果很差

所有精细度高的，存在BSpline曲线的模型，其还原度都十分差劲
