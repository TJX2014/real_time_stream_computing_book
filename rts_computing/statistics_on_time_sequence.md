## 时间序列上的统计
在金融风控或反欺诈场景下，经常需要使用一些关于维度和密集度的特征。
比如过去一周内在同一个设备上交易次数、过去一天同一用户的总交易金额、过去一周同一用户在同一IP C段的申请贷款次数等等。
如果用SQL描述上面的统计量，分别如下；

```
过去一周内在同一个设备上交易次数
COUNT(*) FROM stream
WHERE event_type = "transaction" 
AND timestamp >= 1530547200000 and timestamp < 1531152000000 
GROUP BY device_id;

过去一天同一用户的总交易金额
SUM(amount) FROM stream
WHERE event_type = "transaction"
AND timestamp >= 1531065600000 and timestamp < 1531152000000
GROUP BY user_id;

过去一周同一用户在同一IP C段申请贷款次数
COUNT(*) FROM stream 
WHERE event_type = "loan_application"
AND timestamp >= 1530547200000 and timestamp < 1531152000000
GROUP BY ip_seg24;
```

可以发现，这些指标都有一个共同的模式，即在一个时间窗口（timestamp window）内，从一个或多个维度(group by)，
统计事件(event type)发生的次数(count)或事件中某个变量的总和(sum)。

为了统一和方便，在接下来的章节中，我们用下面的模式来描述一个时间序列上的算子：

```
OPERATOR(time, event_type, target[=value], on1[=value], on2[=value], ...)
```

其中各个符号的含义如下：
OPERATOR表示要统计的类型，比如计数（COUNT）、求和(SUM)、最大（MAX）、最小（MIN）、均值(AVG)、方差（VARIANCE）等等。
time表示统计的窗口，比如"1d"表示过去一天，"5m"表示过去5分钟，"2h"表示过去2小时等等。
event_type表示事件类型，比如"transaction"表示交易事件，"loan_application"表示贷款申请等等。可以根据具体业务设置事件类型。
target就是要统计的目标变量，比如在前面的例子中，target分别是设备（device_id）、交易金额（amount）、IP C段（ip_seg24）。
on1、on2等则是对target更细维度的划分，比如"过去一天同一用户的总交易金额"中，target为交易金额，on为用户。而"过去一周同一用户在同一IP C段的申请贷款次数"中，target为IP C段，on为用户。
target和on后面可以通过等号指定一个值（value），含义就是指定变量为特定值进行计算。

比如，前面用SQL描述的特征，用上面的描述方式分别是：

```
过去一周内在同一个设备上交易次数
COUNT(7d, transaction, device_id)

过去一天同一用户的总交易金额
SUM(1d, transaction, amount, userid)

过去一周同一用户在同一IP C段申请贷款次数
COUNT(7d, loan_application, ip_seg24, userid)
```

而如果指定参数，则比如：

```
过去一周内在设备"d000001"上交易次数
COUNT(7d, transaction, device_id=d000001)

过去一天用户"ud000001"的总交易金额
SUM(1d, transaction, amount, userid=ud000001)

过去一周同一用户在IP C段"220.181.111"的申请贷款次数等等
COUNT(7d, loan_application, ip_seg24="220.181.111", userid)
```

### 在实时流上计算特征的关键点

在开始具体讲解实时流上各种特征的计算方式前，我们需要线分析下实时流上特征计算的几个关键点。

#### 时间戳
实时流上的时间通常有两类：事件发生时的时间戳和服务器处理事件时的时间戳。
由于本章特征提取关注的是对真实世界发生事件的描述，所以本章特征提取使用的时间戳是事件发生时的时间戳。

#### 时间窗口
对于实时流上的特征（指标）计算，我们需要界定一个分析的时间窗口。
这样做的原因有两点。一方面"流"是无穷无尽的，对于不确定的未来而言，我们永远也不确定下一刻会发生什么，
因此在不确定时间窗口的情况下，我们不能定义这条"流"上的最大值是多少、最小值是多少、平均值是多少等等问题。
另一方面"流"是在不断更新的，"旧"的数据终会被淘汰，淘汰的方式可以是不计入当前分析，也可以是减小对当前分析的影响程度，
但不管怎样，只有指定了窗口我们才能界定哪些数据是"旧"的。
"窗口"的存在很多时候会使我们的问题变得复杂，但同时它也帮助我们确定了分析的时间边界。
因此"窗口"是个双刃剑，我们在实时流上做分析时，一定要考虑"窗口"的问题。

#### 特征计算的模式
笔者了解过不同的风控系统架构。总结下来，这些系统架构对于特征的使用方式，可以分为两种：同步方式和异步方式。
1. 所谓同步方式，是指在实时流上计算完特征后，这些特征接下来就输入到规则系统或模型系统，给出决策结果。
2. 所谓异步方式，是指在实时流上计算完特征后，这些特征被保存到数据库里，等到业务需要做决策时，
再从数据库中查出这些特征，然后调用规则系统或模型系统，给出决策结果。

这两种不同的使用方式，对我们特征计算的需求是不同的。
同步方式下，特征计算和结果获取是在一起的。特征计算完后输出计算结果即可。
异步方式下，特征计算和结果获取是分开的。在事件到达时，只需要做特征更新计算，而在查询时，再做特征查询计算。

为了同时支持这两种方案，我们将特征计算划分为两种：更新计算和查询计算。
1. 更新计算。当实时流中的事件到来时，只根据这个新到的事件，更新统计信息，这些统计信息通常保存在数据库中。
注意这些统计信息不一定就是特征计算的最终结果，它们可以是一些中间结果。
在查询计算时，通过这些中间结果进一步计算就可以获得最终的特征计算结果。
2. 查询计算。当需要查询特征时，从数据库中查询统计信息，然后根据这些统计信息直接或间接地计算出最终要查询的特征。
如果在更新计算时，已经计算出了特征计算结果，就可以直接返回这个结果。
而如果在更新计算时，仅仅是更新了中间结果，就还需要根据这些中间结果进一步计算出最终特征结果。

据此，我们将特征计算分成三种模式：update、get和upget，
其中update模式对应更新计算，get模式对应查询计算，
而upget模式则是update和get的合并，upget模式下会进行更新计算并获取特征计算结果。

下面我们就按照窗口和计算模式的思路来实现各种特征。

### 计数
#### 定义
实时流上的计数通常是为了统计符合指定条件的事件数。
比如过去一周内在同一个设备上交易次数、过去一周同一用户在IP C段"220.181.111"的申请贷款次数等等。

#### 计算方法

一种简单的计算方式是当数据到来时，把它保存到状态数据库，然后遍历窗口内的所有数据，过滤出符合指定条件的事件并统计数量。
但是大多数情况下这种简单的方式在事实流计算中会存在性能问题。
因为如果将每条消息都保存在状态数据库中，当窗口较长、数据量较大时，会占用过多内存。
而且每次的计算需要遍历所有的数据，这无疑会消耗过多的计算资源，同时还增加了处理的时延。

因此，我们需要尽可能地降低计算复杂度，只保留必要的聚合信息，而不需要保存所有的原始数据。

对于计数而言，我们只需要在每个时间窗口内，给变量的每一种可能的取值分配一个计数寄存器，
当数据到达时，将相应取值的计数寄存器加一即可。

#### 更新计算
记变量$$X$$的值域为集合$$S=\{E_1, E_2, E_3, \cdots\}$$，用`REG(E_i)`记录$$X=E_i$$发生的次数。
当新数据到达，并且其值为`E_i`时，更新方法如下：

```
REG(E_i) = REG(E_i) + 1
```

#### 查询计算
直接返回该变量取值对应的`REG(E_i)`即可。

```
return REG(E_i)
```

#### 窗口合并
设$$REG(E_i)_{t1}$$是窗口$$t_1$$上，$$X=E_i$$发生的次数，
设$$REG(E_i)_{t2}$$是窗口$$t_2$$上，$$X=E_i$$发生的次数。
则合并两个窗口的计数是：

$$
REG(E_i)=REG(E_i)_{t1}+REG(E_i)_{t2}
$$

#### 算子
```
COUNT(time, event_type, target[=value], on1[=value], on2[=value], ...)
```

算子使用样例

```
过去一周内在同一个设备上交易次数
COUNT(7d, transaction, device_id)

过去一周同一用户在IP C段"220.181.111"的申请贷款次数等等
COUNT(7d, loan_application, ip_seg24="220.181.111", userid)
```

### 求和
#### 定义
设$$(X_1, X_2, \cdots, X_N)$$是时间序列上某个窗口$$W$$的随机变量样本，则求和定义为：

$$
SUM=\sum_{i=1}^N X_i
$$

#### 计算方法

与计数的情况一样，为了降低计算复杂度并提高性能，对于求和而言，
我们只需要保存窗口内变量总和即可，也就是$$SUM$$。

#### 更新计算
记录窗口内当前样本的均值为`SUM`，新到的数据值为`X_i`。更新计算的算法如下：

```
SUM = SUM + X_i
```

#### 查询计算
直接返回该窗口变量的总和值即可。

```
return SUM
```

#### 窗口合并
设$$SUM_{t1}$$是窗口$$t_1$$上变量的总和，
设$$SUM_{t2}$$是窗口$$t_2$$上变量的总和
则合并两个窗口的计数是：

$$
SUM=SUM_{t1}+SUM_{t2}
$$

#### 算子
```
SUM(time, event_type, target[=value], on1[=value], on2[=value], ...)
```

算子使用样例

```
过去一天同一用户的总交易金额
SUM(1d, transaction, amount, userid)

过去一天用户"ud000001"的总交易金额
SUM(1d, transaction, amount, userid=ud000001)
```

### 均值

#### 定义
设$$(X_1, X_2, \cdots, X_N)$$是时间序列上某个窗口$$W$$的随机变量样本，则均值定义为：

$$
\overline{X}=\frac{1}{N}\sum_{i=1}^N X_i
$$

#### 计算方法

一种简单的计算方式是当数据$$X_i$$到来时，把它保存到状态数据库，然后遍历窗口内的所有数据算一个均值。
但是大多数情况下这种简单的方式在事实流计算中会存在性能问题。
因为如果将每条消息都保存在状态数据库中，当窗口较长、数据量较大时，会占用过多内存。
而且每次的计算需要遍历所有的数据，这无疑会消耗过多的计算资源，同时还增加了处理的时延。

因此，我们需要尽可能地降低计算复杂度，只保留必要的聚合信息，而不需要保存所有的原始数据。

对于均值而言，我们只需要保存每个窗口的均值和样本数量即可，也就是$$\overline{X}$$和$$N$$。

#### 更新计算
记录窗口内当前样本的均值为AVG，数量为n。更新计算的算法如下：

```
AVG = (AVG * n + X) / (n + 1)
n = n + 1
```

#### 查询计算
直接返回该窗口的AVG值即可。

```
return AVG
```

#### 窗口合并
设$$\overline{X}_{t1}$$和$$N_{t1}$$分别是时间段$$t_1$$窗口上的样本均值和样本数量，
设$$\overline{X}_{t2}$$和$$N_{t2}$$分别是时间段$$t_2$$窗口上的样本均值和样本数量。
则合并两个窗口的均值是：

$$
\overline{X}=\frac{\overline{X}_{t1}N_{t1}+\overline{X}_{t2}N_{t2}}{N_{t1}+N_{t2}}
$$

### 中心矩

#### 定义
设$$(X_1, X_2, \cdots, X_N)$$是时间序列上某个窗口$$W$$的随机变量样本，则$$k$$阶中心矩定义为：

$$
M_k=\frac{1}{N}\sum_{i=1}^N(X_i-\overline{X})^k, k = 1, 2, 3, \cdots
$$

当$$k=2$$时，二阶中心矩为$$X$$方差。
当$$k=3$$时，三阶中心矩为$$X$$偏度。
当$$k=4$$时，四阶中心矩为$$X$$峰度。

#### 计算方法
如果严格按照定义来计算中心矩，就需要保存窗口内的所有原始数据。
这种做法在实时流计算中并不是非常明智，它会严重降低实时计算的性能。
为此，我们需要将问题做一些转化。幸好随机变量的各阶中心矩和期望之间，都存在着一定的关系。具体如下。

记随机变量$$X$$的期望为$$E(X)$$，则有：

方差与期望的关系
$$
M_2=E[(X-E(X))^2]=E(X^2)-E(X)^2
$$

偏度与期望的关系

$$
M_3=E[(X-E(X))^3]=E(X^3)-3E(X)E(X^2)+2E(X)^3
$$

峰度与期望的关系

$$
M_4=E[(X-E(X))^4]=E(X^4) - 4E(X^3)E(X) + 6E(X^2)E(X)^2 - 3E(X)^4
$$

从上面的各个关系式可以看出，我们能够将中心矩的计算，全部转化为均值（样本均值是期望的无偏估计）的计算。
比如对于方差，可以分别计算原始数据（$$X$$）的均值和原始数据平方($$X^2$$)的均值，
然后经过简单的计算（平方再相减）即可得到方差（这个值是有偏的，但是当窗口内的样本数N较大时，差别并不大）。
对于偏度和峰度，与此类似也可以算出来。

#### 更新计算
采用前文所讲将中心矩转化为均值计算的方法，我们以方差为例来说明计算方法。
记录窗口内当前样本的均值为AVG_X，平方后的均值为AVG_X2，数量为n。更新计算的算法如下：

```
AVG_X = (AVG_X * n + X) / (n + 1)
AVG_X2 = (AVG_X2 * n + X^2) / (n + 1)
n = n + 1
```

#### 查询计算
查询指定窗口上的均值AVG_X和平方均值AVG_X2，然后做如下计算：

```
return AVG_X2 - AVG_X^2
```

#### 窗口合并
由于将中心矩转化为均值计算，故对于窗口合并后的中心矩计算，
只需要先计算窗口合并后的各个均值，然后通过公式即可以得到中心矩。

### 协方差

#### 定义
设$$X$$和$$Y$$分别为两个随机变量，$$E(X)$$与$$E(Y)$$分别是X与Y的期望，则协方差定义为：

$$
Cov(X, Y)=E[(X-E(X))(Y-E(Y))]
$$


#### 计算方法
以协方差的定义式为基础，进一步推导可以得到：

$$
Cov(X, Y)=E[(X-E(X))(Y-E(Y))]=E(XY)-E(X)E(Y)
$$

可以看出，与方差类似，协方差也可转化为均值的计算，即两个随机变量乘积的均值减去各自均值的乘积。

#### 更新计算
采用前文所讲将中心矩转化为均值计算的方法，我们以方差为例来说明计算方法。
记录窗口内随机变量X的均值为AVG_X，随机变量Y的均值为AVG_Y，
两个随机变量乘积均值为AVG_XY，数量为n。更新计算的算法如下：

```
AVG_X = (AVG_X * n + X) / (n + 1)
AVG_Y = (AVG_Y * n + Y) / (n + 1)
AVG_XY = (AVG_XY * n + X * Y) / (n + 1)
n = n + 1
```

#### 查询计算
查询指定窗口上的三个均值AVG_X、AVG_Y和AVG_XY，然后做如下计算：

```
return AVG_XY - AVG_X * AVG_Y
```

#### 窗口合并
由于将协方差转化为均值计算，故对于窗口合并后的协方差计算，
只需要先计算窗口合并后的各个均值，然后通过公式即可以得到协方差。

### 相关系数


#### 定义
设$$X$$和$$Y$$分别为两个随机变量，$$D(X)$$与$$D(Y)$$分别是X与Y的方差，则相关系数定义为：

$$
\rho_{XY}=\frac{Cov(X, Y)}{\sqrt{D(X)D(Y)}}
$$


#### 计算方法
在前面的方差和协方差计算中，我们都是将其转为了均值的计算。
根据相关系数的定义，它时由协方差和方差衍生而来。
因此我们在这里就不再详细展开了。因为只需要按照前述的方法计算出协方差和方差，就可以得到相关系数了。


### 直方图

#### 定义
直方图用于描述样本在一个或多个不相交区间上的分布情况。
如果用$$N$$表示样本总数，用$$m_i$$表示落在区间$$i$$上的样本数量，用$$k$$表示不相交区间的个数，则有：

$$
n=\sum_{i=1}^k m_i
$$

#### 计算方法
如果需要统计直方图的变量的取值范围已知，并且区间的个数已经给定，那么获取直方图并不困难。
只需要在每个时间到来时，计算出它落在哪个区间内，然后将对应区间的计数加一即可。

但是很多情况下，尤其在实时流处理中，我们可能并不能预先知道所研究对象的取值范围，
这样导致的结果就是我们不能预先划分出直方图各个区间的范围。

因此，针对这种取值范围未知的随机变量的直方图统计，就具备一定的技巧性了。
下面我们介绍一种实时流上的直方图统计方法。
1. 用$$K$$表示所要统计直方图的柱状条个数，记第i个柱状条为$$<X_i, M_i>$$，
其中$$X_i$$表示柱条取值，$$M_i$$表示柱条包含的样本数量。根据想要的直方图精细度，选择一个$$K$$值，比如$$K=64$$。
2. 当新数据$$X$$到达时，如果当前直方图的柱条数不足$$K$$，就新增一个柱条，记为$$<X, 1>$$。
如果当前直方图的柱条数已经为$$K$$，就需要将新到达的数据与某个现有柱条合并。合并步骤如下：

1) 记新增数据$$X$$为一个新柱条$$<X, 1>$$。

2) 如果$$X$$与某个柱条的值相同，则直接将这个柱条的计数加一。
否则遍历已有$$K$$个柱条，找到与$$X$$的值最相近的柱条，记为$$<X_{nearest}, M_{nearest}>$$。

3) 合并$$<X, 1>$$和$$<X_{nearest}, M_{nearest}>$$为新柱条，记为$$<X_{new}, M_{new}>$$。合并算法为：

$$
X_{new}=\frac{X_{nearest}M_{nearest} + X}{M_{nearest}+1}
$$

$$
M_{new}=M_{nearest}+1
$$

4) 将$$<X_{new}, M_{new}>$$加入直方图，同时还需要删除$$<X, 1>$$和$$<X_{nearest}, M_{nearest}>$$。

按照上述步骤，随着实时流上数据不断流入，就会逐渐形成一个描述数据分布状态的直方图。

#### 更新计算
直方图统计更新的方法在上一小节中已经描述，这里不再重述。
但需要注意的是，我们在做更新操作时，是针对一个窗口上的直方图而言的。
也就是说，每一个窗口我们都会用一个直方图来描述这个窗口上的数据分布。


#### 查询计算
由于更新计算的结果就是一个直方图，故直接返回直方图即可。

#### 窗口合并
由于我们是对每个窗口计算一个直方图，那么当查询的是跨多个窗口数据的直方图时，就会涉及到多个直方图的合并问题了。
多个直方图合并的算法其实与新增数据合并到已有直方图中的方法非常相似。
在新增数据合并到直方图时，我们将新增数据$$X$$视为一个计数为1的新柱条，即$$<X, 1>$$。
而两个直方图的合并，其实就是将其中一个直方图的每个柱条，逐一合并到另一个直方图的过程。
具体而言，对于两个直方图$$A$$和$$B$$，将直方图$$A$$的每个柱条$$<X_i, M_i>$$，逐个合并到直方图$$B$$。
合并时使用的公式也非常相似（本质上就是同一种算法），如下所示。

$$
X_{new}=\frac{X_{nearest}M_{nearest} + X_iM_i}{M_{nearest}+M_i}
$$

$$
M_{new}=M_{nearest}+M_i
$$

其中，$$<X_i, M_i>$$是$$A$$中的柱条，$$<X_{nearest}, M_{nearest}>$$是在$$B$$中找到的与X_i最相近的柱条，
$$<X_{new}, M_{new}>$$就是新合并的柱条。当合并完$$A$$中所有的$$<X_i, M_i>$$时，也就完成了两个直方图的合并。

显然，当$$M_i=1$$时，就是新数据合并到直方图时的情形。

### 分位数

#### 定义
分位数（Quantile）是指将一个随机变量的概率分布范围分为几个等份的数值点。
常用的有中位数（即二分位数）、四分位数、百分位数等。

#### 计算方法
在前面我们已经讲述了直方图的计算方法。而分位数就可以在这个直方图的基础上计算而来。计算方法如下。

考虑直方图$$\{<X_1, M_1>, <X_2, M_2>, \cdots ,<X_K, M_K>\}$$，对于一个给定值$$X_b$$，且$$X_1<X_b<X_K$$。
我们按如下方法估计区间$$(-\infty, X_b]$$之间的样本数量。
1. 找到一个$$i$$，使得$$X_i \le {X_b}<X_{i+1}$$
2. 记s为

$$
s=\frac{M_i+M_b}{2}\cdot\frac{X_b-X_i}{X_{i+1}-X_i}
$$

  其中，

$$
M_b=M_i+\frac{M_{i+1}-M_i}{X_{i+1}-X_i}(X_b-X_i)
$$

则区间$$(-\infty, X_b]$$之间的样本数量估计值$$num(-\infty, X_b)$$为：

$$
num(-\infty, X_b) = \sum_{j=1}^{i-1}M_j+\frac{M_i}{2}+s
$$

下面，我们以第一四分位数（Q1）为例，来说明分位数的计算方法。

1. 记$$\{S_1, S_2, \cdots, S_n\}$$为区间$$[X_1, X_K]$$上的$$n+1$$等分点，记$$SM=\sum_{i}^KM_i$$。
则理论上第一四分位数应该满足$$\frac{num(-\infty, S_j)}{SM}=0.25$$。
2. 找到使得$$|\frac{num(-\infty, S_j)}{SM}-0.25|$$值最小的$$j$$，则$$S_j$$即是第一四分位数的近似值。
当然，$$n$$越大，$$S_j$$会越准确。

通过类似的方法，我们就可以得到各种分位数了。
由于分位数计算在直方图上很容易推导而来，而直方图的统计方法前面我们已经讲解过，这里就不再详细展开了。


### 差分
设$$(X_1, X_2, \cdots, X_N)$$是时间序列上某个窗口$$W$$的随机变量样本，则其一阶差分序列定义为

$$
(X_2-X_1, X_3-X_2, \cdots, X_N-X_{N-1})
$$

在一阶差分的基础上，再做一次一阶差分，即可以得到原始时间序列的二阶差分。

差分是一种在时间序列上经常使用的算子，比如关于序列平稳性的计算中就经常用到。

#### 计算方法
以一阶差分为例，只需记录了变量在上一次事件中的值，用当前事件中的值减去上一次的值，就是其差分值了。
不过需要注意的是，在实际实现中，由于多线程和并发的原因，事件处理的时间可能乱序。
因此不能简单地记录上一次的值，最好记录最近的多个值并把事件时间戳因素也考虑在内，这样才会得到正确的差分结果。
差分的计算相对简单，故此不再继续展开。
