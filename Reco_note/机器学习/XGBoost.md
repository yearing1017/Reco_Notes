#### 简介

XGBoost与GBDT比较大的不同就是目标函数的定义，但这俩在策略上是类似的，都是聚焦残差（更准确的说， xgboost其实是gbdt算法在工程上的一种实现方式），**GBDT旨在通过不断加入新的树最快速度降低残差，而XGBoost则可以人为定义损失函数（可以是最小平方差、logistic loss function、hinge loss function或者人为定义的loss function），只需要知道该loss function对参数的一阶、二阶导数便可以进行boosting，其进一步增大了模型的泛化能力，其贪婪法寻找添加树的结构以及loss function中的损失函数与正则项等一系列策略也使得XGBoost预测更准确**

- GBDT是机器学习算法，XGBoost是GBDT的工程实现
- **精度更高**：GBDT只用到一阶泰勒， 而XGBoost对损失函数进行了**二阶泰勒展开， 一方面为了增加精度， 另一方面也为了能够自定义损失函数，二阶泰勒展开可以近似大量损失函数**
- 灵活性更强：GBDT以CART作为基分类器，而XGBoost不仅支持CART，还支持线性分类器，另外，Xgboost支持自定义损失函数，只要损失函数有一二阶导数
- **正则化**：XGBoost在目标函数中加入了正则，用于控制模型的复杂度。有助于降低模型方差，防止过拟合。**正则项里包含了树的叶子节点个数，叶子节点权重的L2范式**。这个东西的好处，就是XGBoost在构建树的过程中，就可以进行树复杂度的控制，而不是像GBDT那样， 等树构建好了之后再进行剪枝
- **Shrinkage（缩减**）：相当于学习速率。这个主要是为了**削弱每棵树的影响，让后面有更大的学习空间，学习过程更加的平缓**
- **列抽样**：这个就是在建树的时候，**不用遍历所有的特征了，可以进行抽样**，一方面简化了计算，另一方面也有助于降低过拟合
- **缺失值处理：**这个是XGBoost的稀疏感知算法，加快了节点分裂的速度， 传统的GBDT没有设计对缺失值的处理， 而XGBoost能自动学习出缺失值的处理策略
- 传统的GBDT在每轮迭代时使用全部的数据，XGBoost则采用了与随机森林相似的策略， **支持对数据进行采样**
- **并行化操作：块结构可以很好的支持并行计算**

XGBoost是好多弱分类器的集成，训练弱分类器的策略就是尽量的减小残差，使得答案越来越接近正确答案。 **xgboost的精髓部分是目标函数的Taylor化简，这样就引入了损失函数的一阶和二阶导数。然后又把样本的遍历转成了对叶子节点的遍历，得到了最终的目标函数。这个函数就是衡量一棵树好坏的标准。在建树过程中，xgboost采用了贪心策略，并且对寻找分割点也进行了优化**。基于这个，才有了后面的最优点切分建立一棵树的过程。 **xgboost训练的时候，是通过加法进行训练，也就是每一次只训练一棵树出来， 最后的预测结果是所有树的加和表示**

XGBoost算法是采用分步前向加性模型，XGBoost 是**由 k 个基模型组成的一个加法运算式：**
$$
\hat{y}_{i}=\sum_{t=1}^{k} f_{t}\left(x_{i}\right)
$$
其中$f_k$ 是第k个基模型，$\hat{y_i}$ 为第 i 个样本的预测值

**损失函数**可由预测值和真实值表示；n为样本的数量
$$
L=\sum_{i=1}^{n} l\left(y_{i}, \hat{y}_{i}\right)
$$
XGBoost算法通过**优化结构化损失函数（加入了正则项的损失函数，可以起到降低过拟合的风险）**来实现弱学习器的生成，并且XGBoost没有采用搜索方法，而是**直接利用了损失函数的一阶导数和二阶导数值，并通过预排序、加权分位数等技术来大大提高了算法的性能。**

#### XGBoost树的生成过程

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-1.png)

##### 定义目标函数

模型的预测精度由偏差和方差共同决定，**损失函数代表了模型的偏差，想要方差小则需要更简单的模型**， 所以目标函数最终由损失函数L与抑制模型复杂度的正则项Ω组成， 所以目标函数如下：
$$
O b j^{(t)}=\sum_{i=1}^{n} l\left(\hat{y}_{i}^{t}, y_{i}\right)+\Omega\left(f_{t}\right)
$$
**正则化项**
$$
\Omega\left(f_{t}\right)=\gamma T_{t}+\frac{1}{2} \lambda \sum_{j=1}^{T} w_{j}^{2}
$$
**前面的 $T_t$ 为叶子节点数，$w_j$  表示 j 叶子上的节点权重， γ，λ是预先给定的超参数。引入了正则化之后，算法会选择简单而性能优良的模型， 正则化项只是用来在每次迭代中抑制弱分类器$f_i(x)$过拟合，不参与最终模型的集成。**


##### 目标函数的化简

boosting模型是前向加法， **以第t步模型为例， 模型对第i个样本的预测**为：
$$
\hat{y}_{i}^{t}=\hat{y}_{i}^{t-1}+f_{t}\left(x_{i}\right)
$$
其中，$\hat{y}_{i}^{t-1}$是第t-1步的模型给出的预测值， 是已知常数，$f_{t}\left(x_{i}\right)$是我们这次需要加入的新模型，所以把这个$\hat{y}_{i}^{t-1}$代入目标函数化简：
$$
\begin{aligned} O b j^{(t)} &=\sum_{i=1}^{n} l\left(y_{i}, \hat{y}_{i}^{t}\right)+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n} l\left(y_{i}, \hat{y}_{i}^{t-1}+f_{t}\left(x_{i}\right)\right)+\Omega\left(f_{t}\right) \end{aligned}
$$
**这个就是xgboost的目标函数了，最优化这个目标函数，其实就是相当于求解当前的**$f_{t}\left(x_{i}\right)$

##### **目标函数的Taylor化简：**

我们首先看一个关于目标函数的简单等式变换， 把上面的目标函数拿过来：
$$
\begin{aligned} O b j^{(t)} &=\sum_{i=1}^{n} l\left(y_{i}, \hat{y}_{i}^{t}\right)+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n} l\left(y_{i}, \hat{y}_{i}^{t-1}+f_{t}\left(x_{i}\right)\right)+\Omega\left(f_{t}\right) \end{aligned}
$$
**泰勒公式的概念**

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-2.png)

根据Taylor公式， 我们把函数$f(x)$在$x_0$处二阶展开：
$$
f(x) \approx f\left(x_{0}\right)+f^{\prime}\left(x_{0}\right)\left(x-x_{0}\right)+\frac{1}{2} f^{\prime \prime}\left(x_{0}\right)\left(x-x_{0}\right)^{2}
$$
类比之下，我们将 $l\left(y_{i}, \hat{y}_{i}^{t-1}+f_{t}\left(x_{i}\right)\right)$ 在点 $l((y_{i}, \hat{y}_{i}^{t-1})$ 展开，**因为当损失函数已知时，前面t-1棵树的损失结果是已知的了**
$$
O b j^{(t)} \approx \sum_{i=1}^{n}\left[l\left(y_{i}, \hat{y}_{i}^{t-1}\right)+g_{i} f_{t}\left(x_{i}\right)+\frac{1}{2} h_{i} f_{t}^{2}\left(x_{i}\right)\right]+\Omega\left(f_{t}\right)
$$
其中 $ g_i $ 是损失函数$ l $对前面预测值的一阶导数，相当于 $ f'(x_0) $ , $h_i$ 是损失函数 $l$ 的二阶导， 注意这里的求导是对 $\hat{y}_{i}^{t-1} $求导
$$
g_{i}=\frac{\partial l\left(y_{i}, \hat{y}_{i}^{(t-1)}\right)}{\partial \hat{y}_{i}^{(t-1)}}, h_{i}=\frac{\partial^{2} l\left(y_{i}, \hat{y}_{i}^{(t-1)}\right)}{\partial \hat{y}_{i}^{(t-1)}}
$$
由于在第t步时 $\hat{y}_{i}^{t-1} $ 是一个已知的值，所以 $l((y_{i}, \hat{y}_{i}^{t-1})$ 是常数，对函数的优化不会产生影响，因此目标函数进一步写成：

$$
O b j^{(t)} \approx \sum_{i=1}^{n}\left[g_{i} f_{t}\left(x_{i}\right)+\frac{1}{2} h_{i} f_{t}^{2}\left(x_{i}\right)\right]+\Omega\left(f_{t}\right)
$$
**所以我们只需要求出每一步损失函数的一阶导和二阶导的值，然后最优化目标函数**，就可以得到每一步的 $ f(x)$ ，最后根据加法模型得到一个整体模型。但是还有个问题，就是我们如果是建立决策树的话，根据上面的可是无法建立出一棵树来。因为这里的$f_t(x_i)$ 是什么鬼？ 咱不知道啊！所以还得进行一步映射， **将样本x映射到一个相对应的叶子节点才可以**，看看是怎么做的？

##### **基于决策树的目标函数的终极化简**

我们这里先解决一下这个$ f_t(x_i) $ 的问题，这个究竟怎么在决策树里面表示呢？ 我们看看这个$ f_t(x_i)$ 表示的含义是什么，$ f_t $ 就是我有一个决策树模型， $x_i $ 是每一个训练样本，那么这个整体 $ f_t(x_i) $ 就是某一个样本$ x_i $ 经过决策树模型 $ f_t$ 得到的一个预测值， 那么，我如果是在决策树上，可以这么想， 我的决策树就是这里的 $ f_t $，然后对于每一个样本 $ x_i $， **我要在决策树上遍历获得预测值，其实就是在遍历决策树的叶子节点，因为每个样本最终通过决策树都到了叶子上去**（样本都在叶子上， 只不过这里**要注意一个叶子上不一定只有一个样本**）

所以，**通过决策树遍历样本，其实就是在遍历叶子节点**。这样我们就可以把问题进行转换，把决策树模型定义为
$$
f_t(x) = w_{q(x)}
$$
其中，**q(x) 代表了该样本在哪个叶子节点上，w表示该叶子节点上的权重，所以 $ w_{q(x)} $** **代表了每个样本的预测值**

那么这个样本的遍历，就可以这样化简：
$$
\sum_{i=1}^{n}\left[g_{i} f_{t}\left(x_{i}\right)+\frac{1}{2} h_{i} f_{t}^{2}\left(x_{i}\right)\right] = \sum_{i=1}^{n}\left[g_{i} w_{q\left(x_{i}\right)}+\frac{1}{2} h_{i} w_{q\left(x_{i}\right)}^{2}\right]=\sum_{j=1}^{T}\left[\left(\sum_{i \in I_{j}} g_{i}\right) w_{j}+\frac{1}{2}\left(\sum_{i \in I_{j}} h_{i}\right) w_{j}^{2}\right]
$$
**遍历所有的样本后求每个样本的损失函数，但样本最终会落在叶子节点上，所以我们也可以遍历叶子节点，然后获取叶子节点上的样本集合（注意第二个等式和第三个等式求和符号的上下标， T代表叶子总个数）**由于一个叶子节点有多个样本存在，所以后面有了 $\sum_{i \in I_{j}} g_{i}$  和 $ \sum_{i \in I_{j}} h_{i}$ 这两项，这里的$ I_{j}$ 它代表一个集合，集合中每个值代表一个训练样本的序号，整个集合就是某棵树第 j 个叶子节点上的训练样本 , $ w_j $ 为第 j 个叶子节点的取值

这时再回顾正则化项，**在决策树中，决策树的复杂度可由叶子数 T 组成，叶子节点越少模型越简单，此外叶子节点也不应该含有过高的权重 w （类比 LR 的每个变量的权重），所以目标函数的正则项可以定义为**
$$
\Omega\left(f_{t}\right)=\gamma T_{t}+\frac{1}{2} \lambda \sum_{j=1}^{T} w_{j}^{2}
$$
即**决策树模型的复杂度由生成的所有决策树的叶子节点数量( γ 权衡)，和所有节点权重( λ 权衡)所组成的向量的范式共同决定**

目标函数的前后两部分都进行了解决，那么目标函数就可以化成最后这个样子
$$
\begin{aligned} O b j^{(t)} & \approx \sum_{i=1}^{n}\left[g_{i} f_{t}\left(x_{i}\right)+\frac{1}{2} h_{i} f_{t}^{2}\left(x_{i}\right)\right]+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n}\left[g_{i} w_{q\left(x_{i}\right)}+\frac{1}{2} h_{i} w_{q\left(x_{i}\right)}^{2}\right]+\gamma T+\frac{1}{2} \lambda \sum_{j=1}^{T} w_{j}^{2} \\ &=\sum_{j=1}^{T}\left[\left(\sum_{i \in I_{j}} g_{i}\right) w_{j}+\frac{1}{2}\left(\sum_{i \in I_{j}} h_{i}+\lambda\right) w_{j}^{2}\right]+\gamma T \end{aligned}
$$
为了简化表达式，我们再定义： $ G_{j}=\sum_{i \in I_{j}} g_{i} \quad H_{j}=\sum_{i \in I_{j}} h_{i}$ ， 那么**决策树版本xgboost的目标函数(二次函数)**
$$
O b j^{(t)}=\sum_{j=1}^{T}\left[G_{j} w_{j}+\frac{1}{2}\left(H_{j}+\lambda\right) w_{j}^{2}\right]+\gamma T
$$
这里要注意 $ G_j $ 和 $ H_j $ 是前$ t-1 $步得到的求导结果， 其值已知，只有最后一棵树的叶子节点 $ w_j $ 的值不确定， 那么将目标函数对 $ w_j $ 求一阶导， 并令其等于0， $ \frac{\partial J\left(f_{t}\right)}{\partial w_{j}}=G_{j}+\left(H_{j .}+\lambda\right) w_{j}=0 $， 则可以求得叶子节点 j 对应的权值:
$$
w_{j}^{*}=-\frac{G_{j}}{H_{j}+\lambda}
$$
目标函数又可化简为：
$$
o b j=-\frac{1}{2} \sum_{j=1}^{T} \frac{G_{j}^{2}}{H_{j}+\lambda}+\gamma T
$$
**这个就是基于决策树的xgboost模型的目标函数最终版本了，这里的G和H的求法，就需要明确的给出损失函数来， 然后求一阶导和二阶导，然后代入样本值即得出**

例子：

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-3.png)

#### **最优切分点划分算法及优化策略**

上面是可以判断出来一棵树究竟好不好，那么建**立树的时候应该怎么建立呢**？一棵树的结构近乎无限多，总不能一个一个去测算它们的好坏程度，然后再取最好的吧（这是个NP问题）。所以，我们仍然需要采取一点策略，这就是逐步学习出最佳的树结构。这与我们将K棵树的模型分解成一棵一棵树来学习是一个道理，只不过从一棵一棵树变成了一层一层节点而已。 这叫什么？ emmm, 贪心（找到每一步最优的分裂结果）！**xgboost采用二叉树， 开始的时候， 全部样本在一个叶子节点上， 然后叶子节点不断通过二分裂，逐渐生成一棵树。**

那么**在叶子节点分裂成树的过程中最关键的一个问题就是应该在哪个特征的哪个点上进行分裂，也就是寻找最优切分点的过程。**

学过了决策树的建树过程，那么我们知道**ID3也好，C4.5或者是CART**，它们寻找最优切分点的时候都有一个计算收益的东西，分别是**信息增益，信息增益比和基尼系数**。 **而xgboost这里的切分， 其实也有一个类似于这三个的东西来计算每个特征点上分裂之后的收益**。

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-4.png)

最优切分点的划分算法：

- 从深度为 0 的树开始，**对每个叶节点枚举所有的可用特征**；
- 针对每个特征，把**属于该节点的训练样本根据该特征值进行升序排列**，通过**线性扫描的方式来决定该特征的最佳分裂点**，并**记录该特征的分裂收益**；（这个过程每个特征的收益计算是可以并行计算的，xgboost之所以快，其中一个原因就是因为它支持并行计算，而这里的**并行正是指的特征之间的并行计算**，千万不要理解成各个模型之间的并行）
- **选择收益最大的特征作为分裂特征，用该特征的最佳分裂点作为分裂位置，在该节点上分裂出左右两个新的叶节点，并为每个新节点关联对应的样本集**（这里稍微提一下，xgboost是可以处理空值的，也就是假如某个样本在这个最优分裂点上值为空的时候， 那么xgboost先把它放到左子树上计算一下收益，再放到右子树上计算收益，哪个大就把它放到哪棵树上。）
- 回到第 1 步，递归执行到满足特定条件为止
  ![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-5.png)

**xgboost的切分操作和普通的决策树切分过程是不一样的。普通的决策树在切分的时候并不考虑树的复杂度，所以才有了后续的剪枝操作。而xgboost在切分的时候就已经考虑了树的复杂度**（obj里面那个 γ）。所以，它不需要进行单独的剪枝操作

这就是xgboost**贪心建树的一个思路了，即遍历所有特征以及所有分割点，每次选最好的那个**。 GBDT也是采用的这种方式， 这算法的确不错，但是有个问题你发现了没？ 就是计算代价太大了，**尤其是数据量很大，分割点很多的时候，计算起来非常复杂并且也无法读入内存进行计算**。 所以作者想到了一种近似分割的方式（可以理解为分割点分桶的思路），选出一些候选的分裂点，然后再遍历这些较少的分裂点来找到最佳分裂点。那么怎么进行分桶选候选分裂点才比较合理呢？我们**一般的思路**可能是根据特征值的大小直接进行等宽或者等频分桶， 像下面这样

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-6.png)

上面就是**等频和等宽分桶的思路**了（这个不用较真，我这里只是为了和作者的想法产生更清晰的对比才这样举得例子），这样选择出的候选点是不是比就少了好多了？但是**这样划分其实是有问题的，因为这样划分没有啥依据**啊， 比如我上面画的等频分桶，我是5个训练样本放一个桶，但是你说你还想10个一组来，没有个标准啥的啊。 即上面那两种常规的划分方式缺乏可解释性，

所以重点来了， 作**者这里采用了一种对loss的影响权重的等值percentiles（百分比分位数）划分算法**（Weight Quantile Sketch）， 我上面的这些铺垫也正是为了引出这个方式，下面就来看看作者是怎么做的，这个地方其实不太好理解，所以慢一些

作者进行候选点选取的时候，**考虑的是想让loss在左右子树上分布的均匀一些，而不是样本数量的均匀，因为每个样本对降低loss的贡献可能不一样，按样本均分会导致分开之后左子树和右子树loss分布不均匀**，取到的分位点会有偏差。这是啥意思呢？ 再来一个图

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-7.png)

**从第一个问题开始**，揭开 $ h_i $ 的神秘面纱， 其实 $ h_i $ 上面已经说过了，损失函数在样本 i 处的二阶导数啊！还记得开始的损失函数吗
$$
O b j^{(t)} \approx \sum_{i=1}^{n}\left[g_{i} f_{t}\left(x_{i}\right)+\frac{1}{2} h_{i} f_{t}^{2}\left(x_{i}\right)\right]+\Omega\left(f_{t}\right)
$$
为啥$ h_i$就能代表第 i 个样本的权值啊？这里再拓展一下吧，我们在**引出xgboost的时候说过，GBDT这个系列都是聚焦在残差上面，但是我们单看这个目标函数的话并没有看到什么残差的东西对不对？ 其实这里这个损失函数还可以进一步化简**的（和上面的化简不一样，上面的化简是把遍历样本转到了遍历叶子上得到基于决策树的目标函数，这里是**从目标函数本身出发进行化简**）
$$
\begin{aligned} \mathcal{L}^{(t)} & \simeq \sum_{i=1}^{n}\left[g_{i} f_{t}\left(\mathbf{x}_{i}\right)+\frac{1}{2} h_{i} f_{t}^{2}\left(\mathbf{x}_{i}\right)\right]+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n}\left[\frac{1}{2} h_{i} \cdot \frac{2 \cdot g_{i} f_{t}\left(\mathbf{x}_{i}\right)}{h_{i}}+\frac{1}{2} h_{i} \cdot f_{t}^{2}\left(\mathbf{x}_{i}\right)\right]+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n} \frac{1}{2} h_{i}\left[2 \cdot \frac{g_{i}}{h_{i}} \cdot f_{t}\left(\mathbf{x}_{i}\right)+f_{t}^{2}\left(\mathbf{x}_{i}\right)\right]+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n} \frac{1}{2} h_{i}\left[\left(2 \cdot \frac{g_{i}}{h_{i}} \cdot f_{t}\left(\mathbf{x}_{i}\right)+f_{t}^{2}\left(\mathbf{x}_{i}\right)+\left(\frac{g_{i}}{h_{i}}\right)^{2}\right)-\left(\frac{g_{i}}{h_{i}}\right)^{2}\right]+\Omega\left(f_{t}\right) \\ &=\sum_{i=1}^{n} \frac{1}{2} h_{i}\left[\left(f_{t}\left(\mathbf{x}_{i}\right)+\frac{g_{i}}{h_{i}}\right)^{2}\right]+\Omega\left(f_{t}\right)+\text { Constant } \\ &=\sum_{i=1}^{n} \frac{1}{2} h_{i}\left(f_{t}\left(\mathbf{x}_{i}\right)-\left(-\frac{g_{i}}{h_{i}}\right)\right)^{2}+\Omega\left(f_{t}\right)+\text { Constant } \end{aligned}
$$
后面的每一个分类器都是在拟合每个样本的一个残差 $ -\frac{g_i}{h_i} $ , 其实把上面化简的平方损失函数拿过来就一目了然了。**而前面的 $ h_i $ 可以看做计算残差时某个样本的重要性, 即每个样本对降低loss的贡献程度**

**Xgboost引入了二阶导之后，相当于在模型降低残差的时候给各个样本根据贡献度不同加入了一个权重，这样就能更好的加速拟合和收敛，GBDT只用到了一阶导数，这样只知道梯度大的样本降低残差效果好，梯度小的样本降低残差不好，但是好与不好的这个程度，在GBDT中无法展现。而xgboost这里就通过二阶导可以展示出来，这样模型训练的时候就有数了**

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-8.png)

前面那一部分是围绕着**如何建立一棵树进行的，即采用贪心的方式从根节点开始一层层的建立树结构（每一层争取最优），然后就是建树过程中一个关键的问题：如何寻找最优切分点，给出了最优切分点算法，基于这个算法就可以建立树了。后面这一部分是一个优化的过程，提出了一种Weight Quantile Sketch的算法，这个算法可以将原来的分割点进行分桶，然后找到合适的候选分裂点，这样可以减少遍历时尝试的分裂点的数量，是xgboost相比于GBDT做出的切分点优化策略**，现在知道为啥xgboost要快了吧，因为xgboost寻找切分点的时候不用遍历所有的，而是只看候选点就可以了。而且**在特征上，xgboost是可以并行处理的**。这样xgboost的建树过程及优化策略基本上就是这些了

#### **利用新的决策树预测样本值，并累加到原来的值上**

若干个决策树是通过加法训练的， 所谓加法训练，本质上是一个元算法，适用于所有的加法模型，它是一种启发式算法。

**运用加法训练，我们的目标不再是直接优化整个目标函数，而是分步骤优化目标函数，首先优化第一棵树，完了之后再优化第二棵树，直至优化完K棵树。整个过程如下图所示**

![](https://blog-1258986886.cos.ap-beijing.myqcloud.com/%E6%9C%BA%E5%99%A8%E5%AD%A6%E4%B9%A0/19-9.png)

上图中会发现每一次迭代得到的新模型前面有个 η（这个是让树的叶子节点权重乘以这个系数）， **这个叫做收缩率，这个东西加入的目的是削弱每棵树的作用，让后面有更大的学习空间，有助于防止过拟合**。也就是，我不完全信任每一个残差树，**每棵树只学到了模型的一部分，希望通过更多棵树的累加来来弥补，这样让这个让学习过程更平滑，而不会出现陡变**。 

这个和正则化防止过拟合的原理不一样，**这里是削弱模型的作用，而前面正则化是控制模型本身的复杂度， 而这里是削弱每棵树的作用，都是防止过拟合，但是原理不一样。**
