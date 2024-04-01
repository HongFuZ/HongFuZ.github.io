# 一句话总结
作者将决策树模型部署到可编程交换机上：（1）引入流级别的特征显著提升决策树模型推理的准确性；（2）设计CRC（consecutive range marking）算法来解决特征空间爆炸的问题。

# Introduction
在数据平面执行model inference有两个优点：线速、reaction time小。

Prior art分类：
1. 不将learning models部署到数据面，而是从数据面提取信息后在控制面进行模型推理，例如NetWarden，FlowLens；或者在数据面提取信息后在数据面通过预定义的流量过滤策略（如基于阈值的过滤器）来减缓DDoS攻击，例如Poseidon，Jaqen
2. 将learning models部署到数据面，这些工作都只考虑了per-packet features，没有考虑flow-level features，另外每个方案都存在一些问题：
  + pForest、SwitchTree将决策树的一层映射成一个match/action stage，此时stage的数量会限制决策树的深度（tofino1 最多支持12个stage）；
  + Planter提出了包含feature tables、code tables的在数据面表示决策树的架构，但是缺少一些关键的设计（比如如何同时多次使用一个特征）；
  + Mousika提出的二分决策树模型，面临着table entries几何爆炸的问题
    
NetBeacon的创新点：
1. 围绕多阶段顺序模型架构的数据平面感知学习模型设计。由于处于流的不同阶段的数据包携带不同的流级别状态，因此模型在流进行时在流的不同阶段执行动态分析，从而减少基于单个推理模型作出过早分类决策而引入的错误。同时，模型使用了精心设计的流级和逐包特征，这些特征可在数据平面上以线速计算，以确保可部署性。
2. NetBeacon提出了一种高效的模型表示机制，以解决将决策树或森林模型表达到数据平面匹配表时的条目爆炸问题。与最先进的Mosika相比，NetBeacon显著降低了表项消耗（在某些情况下可达75%）。
3. 进一步强化了NetBeacon处理并发流的可扩展性，通过区分短流和长流的处理逻辑，以及在观察到存储索引冲突时允许安全的存储多路复用。这可能允许NetBeacon处理比用于维护每个流状态的寄存器总数更多的并发流。

# Background
**PISA**: Protocol-Independent Switch Architecture

**PISA的灵活性**：在PISA交换流水线（图1）中，网络数据包首先进入解析器进行包头解析，然后进入多个匹配/动作阶段进行数据包处理，最后到达解解析器进行数据包序列化。解析器、匹配/操作和反解析器都可以通过编程来实现所需的协议。匹配/动作支持精确匹配、三元匹配和最长前缀匹配（LPM）。每个匹配对应于一个动作，其中可以执行特定的计算和存储修改。相互依赖的动作需要放在不同的阶段。数据包报头和元数据实例使用无状态存储进行存储，该存储在新数据包到达时重新初始化。PISA还提供状态和持久存储，如计数器、计量器和寄存器。最后，PISA提供了各种机制（如重新提交、重新循环、包生成）来进一步扩展编程能力。
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/f63ad9b1-a19b-4e74-b782-d8c33ecce288)

**PISA的限制**：
+ **计算方面**：它支持布尔运算、移位运算、加减运算，但不支持乘除运算。浮点运算、循环运算和复杂的条件运算也不支持。主要的计算逻辑是使用匹配/动作阶段来实现的，这些阶段不是无限的（例如，Tofino 1有12个阶段）。
+ **存储方面**：在Tofino 1上，每条流水线的SRAM为120MB,TCAM为6.2MB；当一个包通过交换流水线时，每个寄存器只能被访问一次，因此诸如读取然后更新寄存器之类的操作本身不受支持。

 # Motivation
 主要通过实验论证了flow-level features的重要性：图的上半部分对feature按照信息增益（即对决策树的重要程度）由大到小进行排序，可见flow-level features明显占据更重要的地位；图的下半部分对比了只考虑packet级别特征和结合packet+flow特征的准确率对比，在三中场景中准确率分别提升11%、43%、21%。
 ![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/7d4ff73e-2c51-44c9-ab4e-40dd28cd826d)

# Overview of NetBeacon
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/c572a62d-6817-404f-8fd6-99f3420f823f)
如图3所示，在架构上，NetBeacon包括两个部分：感知数据平面的模型设计和高效模型部署。
+ 模型设计阶段：（1）特征工程只能依赖交换机pipeline以线速能够提取或者计算的特征；（2）考虑到流级别特征（例如，数据包大小的平均值）会随着流的进行而改变，NetBeacon提出了一种多阶段序列模型架构，该架构可以随着流的进行而做出多个推理决策，直到系统有足够的信心做出最终决策。
+ 模型部署阶段：将学习到的模型转换为数据平面上的多个特征表和一个模型表，其中特征表将特征值编码为名为range marks的数据结构，这些数据结构进一步映射到存储在模型表中的推理结果。NetBeacon设计了高效的编码机制，在表示模型时，大大减少了表项的消耗。
+ NetBeacon还设计了有状态存储管理模块，实现了数据平面的高效的每流状态管理：一方面，该模块使NetBeacon能够使用纯粹的per-packet特征（即不为短流维护per-flow状态）来处理短流，其中短流使用学习模型进行分类。另一方面，NetBeacon利用硬件哈希来实现存储复用。特别是，当新流的五元组被哈希到占用的寄存器（即存储冲突）时，如果存储的流是类确定的或超时，则新流可以使用此寄存器；否则，NetBeacon回退为对新流使用无状态的每数据包功能。如果数据包属于存储的流，则更新寄存器并计算特征用于模型推理，即查询特征表和模型表。一旦确定了对数据包的推理结果，用户可以根据结果设计自定义的后处理，例如做出丢弃或允许的二进制决策，或者相应地分配细粒度的不同服务优先级。

# Data Plane Aware Model Design
## Feature Engineering
NetBeacon使用的特征包括：
+ per-packet features：例如包大小、ttl、协议等
+ flow-level features
  - 聚合类（aggregate）特征，$F = aggr(a,c,d)$, 当属性$a$满足条件$c$时按照规则$d$来更新F中的feature值。例如特征$F=aggr(packet size, [96,112),+1)$记录的是大小在$[96,112)$范围的包的数量。
  - 总结类（summary）特征，例如max/min, mean, variance
  
## Multi-Phase Sequential Models
作者通过两个数据接分析了flow-level特征会随着时间（包的数量）动态变化（图4），从而说明仅仅通过单个模型来处理flow-level特征是不够的，所以作者提出了多阶段序列模型，在流的不同阶段应用不同的模型：在每个阶段，NetBeacon使用在该阶段计算的特征进行训练和推理，即基于前n个数据包计算第n个数据包的流级特征。触发模型进行推理决策的包称为推理点。推理点的精确排列取决于任务。特别地，每个推理点本质上代表了我们的模型在推理点之前处理了n个数据包之后流的分析结果。因此，推理点可以根据任务的不同而统一或具体地放置。特别地，如果模型使用方差作为特征，由于硬件的限制，推理点硬放在2的幂次位置。
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/2410fc1b-25f3-4421-b9cf-47d705cbabb0)

图5表明多阶段序列模型相较于单一模型推理准确率明显提升：
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/dac6cd5d-0d6b-4861-b86a-266f895070ff)

此外，NetBeacon设置了determination threshold，当某个阶段分类的可能性超过该阈值，则认为该流的类型可以提前确定，不需要后续的推理决策。

# Model Deployment
## Data Plane Model Representation
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/dd66ce50-9dd2-4969-9767-1c3feb90ff23)
决策树相当于将每个特征分层多个区间，不同特征之间的区间组合对应到具体的分类。而可编程交换机不支持range matching，所以首先要将range转换成ternary matching（每个比特可以去0,1或者*）。
如图6所示，按照经典的前缀编码，对于决策树最左边的路径，f1的[65,103)需要编码成8个互不相交的前缀，f2的[0,256)需要一个编码，f3的[10,256)需要6个编码，共有48中组合。因此ternary matching会存在严重的组合爆炸问题，即model table的entry数量爆炸。
NetBeacon的区间编码包括三部分：
+ feature table的range marking，按照决策树中涉及到的$N$个分界点将整个取值空间划分为$N+1$个basis区间，feature table就是将特征的取值空间映射到唯一编码的过程，同时还要求连续的取值区间合并成的associative ranges也能映射到唯一的编码。例如图6张节点4有个associative range [0,65)，它包含两个basis区间，即[0,25)和[25，65)。feature table的range marking规则如表3所示。
+ model table的range marking，某些key可能涉及到associative range，此时引入通配符\*，将feature table中的多个basis range收缩成model table中的一个range mark。model table的range marking规则如表3所示，此时associative range [0,65)可以映射为唯一编码(\*11).
+ feature table的range coding。简单来说就是针对一个具体的值，如何快速地找到对应的range mark。如果直接将每个basis range按照前缀匹配的方式拆成多个前缀则会带来存储效率的问题。作者提出CRC算法，有效地减少前缀的数量。其基本思路是将互不相交的不区分优先级的区间变成有可能相交的区分优先级的区间。前者，区间数量多，但是对于具体的数值只会命中一个ternary entry；后者，区间数量少，但是可能命中多个ternary entry，选择优先级最高的那个。
### CRC 算法
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/fae3c10f-6835-460a-9b2e-0edc8598c0e7)

CRC从数值小的basis range开始编码。对于给定的basis区间$[r_{i-1},r_i)$，CRC将其扩展成parent区间$[p_{start},p_{end})$, 其中$p_{start}\in[0,r_{i-1}$, $p_{end}\in [r_i, r_{i+1})$. 为了找到最优的parent区间， CRC求解如下优化问题：
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/c972136b-88f4-469d-af18-5dbc356688dc)

其中$Prefix(R)$代表表征区间$R$需要的前缀数目。parent区间超出basis区间的头部部分由于会优先被上一个basis区间匹配到，所以并没有在公式中体现；parent区间超出basis区间的尾部部分如果存在的话，为了防止将下一个basis区间错误地匹配到当前basis区间，应该将其匹配顺序放在$Prefix([p_{start},p_{end}))$的entry前面。例如basis区间[65,103)被扩展成了[64,104), 后者需要[64,96)和[96,104)两个子区间前缀即可，而尾部超出的部分[103，104)还单独需要一个前缀。因此basis区间[65,103)在CRC算法下需要三个前缀entry来表征，而在原始的前缀分解算法下需要8个entry来表征。
需要注意的是，CRC算法并没有对最后一个basis区间进行编码，因为当前的range marking机制下最后一个basis range始终会被映射到0.因此如果feature找不到映射的值，将其映射值置为0即可。
图8定量地对比了经典前缀编码算法和CRC编码算法所需要的ternary entry数量，在三个case中entry数量都至少减少了约50%。
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/6d64c6f4-e4e9-433d-aa9a-98d2e2121e07)

### Handling Forest Models
+ 对于最终结果为所有决策树结果的majority的森林模型，如随机森林模型，可以每棵树单独表征和处理，最后聚合所有树的推理结果。此时相当于在独立树的model tables后面加了一个plurality table，它的key为所有树的推理结果的拼接，它的value为所有树推理结果的majority。
+ 对于gradient boosting tree models，如GBDT, XGBoost, LightGBM, 由于最终的推断概率是通过非线性函数（如sigmoid）作用于所有树的推断结果上而得到的，所以需要将所有子树合并成一颗大树，此时树的叶子节点相当于是原来单棵树叶子节点的非线性聚合，而这个非线性操作可以离线提前计算好.如下图所示。另外，在进行叶子节点聚合时可以根据其前置条件的冲突对不可能的组合进行剪枝。
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/3862ade0-cb62-4f54-89b1-a2e457aa0c86)

## Stateful Storage Management
为了利用流级别的功能，NetBeacon 依赖于有状态存储来维护每流的状态。为了实现线速流量分析， NetBeacon依赖于数据平面上的硬件哈希来分配存储索引。此时需要处理哈希冲突的问题：
+ 尽可能的减少冲突：NetBeacon引入长短流二分类模型（这个模型和任务相关），仅使用per-packet特征来判断数据包是否属于长流，然后只记录长流的状态信息。
+ 安全的存储覆盖：当存储索引发生冲突碰撞时，如果现有流的推理类别已经确定或者流已经完成（有流的推理类被确定或流已经完成（即其最后一个数据包到达时间超过预定义的超时） ,NetBeacon 允许新流使用占用的寄存器。否则， NetBeacon 将回退到为新流使用每数据包无状态功能。
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/6718bac1-1db7-4242-8bcf-9aceef35db6b)

## Integrated Data Plane Processing Logic
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/d60071f5-8667-40ed-bbb9-598c564eb543)

![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/6ffe963f-1e35-41d4-8fd9-567fa0cad66e)

## Control plane
在NetBeacon中，控制平面负责（i）在最开始时在数据平面上安装特征表和模型表，（ii）在确定流类别时，在接收到来自数据平面的请求（在Tofino交换机中）更新流类别表。请注意，更新流类表的延迟不会影响流量分析，因为未与流类表匹配的数据包将转而遍历常规模型推断管道。因此，控制平面在NetBeacon中脱离了数据包分类的关键路径，保证了线速流量分析。

# Implementation
以平均值和方差为例介绍了如何计算流级别的特征：
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/c86b6584-5d78-4def-a54f-78bd9e3b8732)

在线推理包括：一个阶段并行匹配特征表，一个阶段匹配（聚合）模型表。
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/d90d7e68-d140-444c-ab56-1b0227d90840)

# Evaluation
![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/c972cbb9-4564-4409-8abd-9134fd875d54)

![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/46902054-7d7e-4a52-a298-b984f9baf925)

![image](https://github.com/HongFuZ/HongFuZ.github.io/assets/14175384/1d0f6fdb-5dcc-41db-a777-5ca8cb18d69d)
