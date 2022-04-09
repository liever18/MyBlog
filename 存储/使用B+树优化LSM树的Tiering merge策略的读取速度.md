# 使用B+树优化LSM树的Tiering merge策略的读取速度

## 内容简介

本博文主要内容为自己对一篇论文的理解，论文将LSM树和B+Tree这两种在数据存储中最为经典高效的数据结构进行结合，使用B+树优化LSM树的Tiering merge策略的读取速度。实验结果表明，所设计的数据结构和策略可以在与tiering merge策略达到相同抑制写放大的水平的情况下，提高其读取的吞吐量。

## 论文信息

**Jungle: Towards Dynamically Adjustable Key-Value Store by Combining LSM-Tree and Copy-On-Write B+Tree**

Jung-Sang Ahn (junahn@ebay.com), 
Mohiuddin Abdul Qader, Woon-Hak Kang, Hieu Nguyen, Guogen Zhang, and Sami Ben-Romdhane
Platform Architecture & Data Infrastructure, eBay Inc.
USENIX HotStorage 2019

## 背景

### LSM-Tree

LSM树首先在内存中缓冲所有写操作，然后将它们刷新到磁盘，并使用顺序I/O合并它们。这种设计带来了许多优点，包括优越的写性能、高的空间利用率、可调性以及简化并发控制和恢复。

#### 基本概念

![image-20220402155924366](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439443.png)

LSM-Tree中的数据采用了分层(level)存储的设计，每个level中有若干个范围不重合的run，每个run中的数据有序，并且每个run有最大存储空间的限制。通常来说，高层级的level中run的范围更小，level中含有更多的run。层级level存储的数据呈指数增长。



##### 写操作

以一个具体操作为例：

![image-20220402161326832](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439616.png)

当log中的数据存满后，将会写入到level中。写入操作为找到level-0中范围存在重叠的run，并与之进行归并操作，删除重复元素。

![image-20220402161414576](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439264.png)

由于当前level-0存在的run的数量超过了限制（4  > 2），因此需要选择一些run往高层级的level进行迁移。

![image-20220402162250041](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439170.png)

如选择了level-0中的run3，它将与level-1中与它范围区间有交集的run进行merge。

![image-20220402162540023](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439335.png)

merge后level-1的run也超出了限制，因此，选择run往高层level进行merge的操作还会继续。

通过这个例子能直观的感受到，当低层级run的数量和run中数据的数量接近或达到限制时，写操作可能会导致多个层级的run进行更新操作，并且越往高层级走，涉及到run的数量越多，造成了大量数据的读写操作。这个现象被称为写放大。

##### 读操作

![image-20220402163248608](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439540.png)

以上图为例，当查找key-6时，先会直接在内存中的log中进行查找，如果未找到，则进入level-0进行查找，由于一个level中各个run区间不重叠，因此key最多出现在一个run中，又因为run之间有序，因此可以用二分的思想在对数的时间复杂度下定位到可能存在key的run。因此会定位到{1, 5, 10, 12}这个run，再查找这个run中是否含有待查找的key。由于run的中数据一般存储在磁盘中，顺序读性能大于随机读性能，因此可能顺序扫描这个run，单次遍历搜索run为线性时间复杂度，与run中数据容量呈线性关系。

由于布隆滤波器的特性，查找结果为false则一定不存在，查找结果为true则大概率存在，因此可以使用布隆滤波器做一次初筛，当这个run使用布隆滤波器查找的结果为true时，再去做遍历查找。经典布隆滤波器参数 10 bits per key 仅有1%的假阳性率，因此使用布隆滤波器可以大大缩减查找的时间。在不使用的情况下，需要逐层搜索level中满足范围的run，直到找到为止，时间复杂度为 O(L*R) ，L为平均的深度，R为一个run的元素个数。而使用布隆滤波器之后，平均时间复杂度可近似为 O(L + R) ，极大的降低了查找的时间复杂度。

#### Tiering merge

由于经典的LSM-Tree存在严重的写放大问题，因此有人提出了延时merge的策略，通过将范围相同的run组成一个run stack，当stack中的run满了之后，再进行merge操作。

![image-20220402171101070](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439143.png)

本质上，这是一个使用空间和读取速度来换取写入速度的策略，对于一些写操作远多于读操作的数据库，这是一种较好的优化。

### Copy-On-Write B+Tree

由于B+树具有超强的查找特性，因此常作为关系型数据索引的主要数据结构，由于B+树本身是一种内存非连续的链式结构，因此不适用于磁盘顺序海量写入操作的场景。

因此需要对B+树的存储结构进行改造，使其插入和修改操作对应为append写入磁盘。

下面先以不包含数据项（只有key）的B+树作为例子，之后再结合k-v结构进行分析。

![image-20220402172438131](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439573.png)

首先使得基础的存储变成顺序存储，仍可以通过指针进行跳跃查找。

接下来以更新操作（**Update E -> E'**）为例，介绍Cow B+Tree对应的修改方法。

![image-20220402172746551](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439132.png)



**step1**

![image-20220402172843531](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439542.png)

**step2**

![image-20220402172900262](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439991.png)

**step3**

![image-20220402172914324](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439458.png)

具体的操作见上面的图例。总结下来就是，会在disk中依次添加修改后的节点，以及它由近至远的祖先节点。由于B+树是一个矮胖树，树深很浅，每次更新或添加操作所需要修改的节点数量很小，因此磁盘IO开销较小。

之后再看看修改后叶子节点的k-v结构对应的更改。

![image-20220402201526093](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439875.png)

根据现代计算机体系，n次小规模的IO操作和合并成一次大规模IO操作相比，会产生更多的IO时间，因此可以将IO操作汇聚成batch一起进行写入。

![image-20220402204129717](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091439561.png)

![image-20220402204227050](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440281.png)

根据上两张图，可以看到，先是将修改后的value写入磁盘，再将所有叶子节点的key重新写入，并记录对应value的位置。

![image-20220402204638901](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440162.png)

上面是第三次write batch，逻辑仍然相同，先写修改的value，再写叶子节点的key。

可以看到，后两次write操作都使得B+Tree中产生了部分废弃数据，包括被修改的value和最新操作前的key，因此在必要时刻，可以对其进行压缩操作。

![image-20220402214748659](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440596.png)

## 本文工作

![image-20220402171410840](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440699.png)

基于RUM猜想，任何访问方法都不能同时是读最优、写最优和空间最优。

Tiering merge 是一个使用空间和读取速度来换取写入速度的策略，主要是通过使用栈结构，将多个run进行组合，使得merge的操作能够被延后。因为在level的一个run stack中存在若干个run，因此在一个level中，平均需要查找多个run，而原始策略仅需查询一次，所以Tiering merge带来的主要副作用为降低了读取速度。

本文使用Cow B+tree来替换run stack，目标是在不影响写入速度的前提下，优化在level中进行搜索的时间。

![image-20220402214956702](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440415.png)



![image-20220402215354846](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440211.png)

tiering stack和cow B+tree其实很像，一个是在时序上创建新的run，一个是创建新的write batch。

在run stack未满时，创建一个新run的开销比一个write batch要小，因为每次write batch需要修改B+树中的非叶子节点，而且还需要多储存叶子节点pointer to value的信息。

带来的好处为对一个run stack/Cow B+Tree进行查找时，可以将遍历所有run这样的线性时间复杂度优化到对数级别。相当于是使用B+树额外的储存空间，并且稍微增加写入的开销，换取了查找性能上的优化。

![image-20220403102532212](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440702.png)

这里有个小疑问，一个run stack里面的run会很多吗？如果run不多的话，这个查找时间应该也不会有数量级上的差异吧。



文章中还提到了原有run的一个不足，每个run有最大空间的限制，然而run实际的大小可能并没有达到最大空间，造成了空间上的浪费。

![image-20220403103044824](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440116.png)

虽然逻辑上使用了stack对run进行了分组，但实际上每个run都是占据了自己独立的储存空间，run stack里面的run实际上相互不干扰。

![image-20220403110427252](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440495.png)

当一个stack中run数量的上限M小于与下一级范围重叠stack的数量T，那么merge之后再分发给下一级level的run的平均大小会小于等于M / T * max_sizeof(run)，必然会造成空间上的浪费。

![image-20220403111257496](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440770.png)

之前提到了LSM-Tree的写放大问题，这里run空间上的浪费也会被放大。

因此较好的设计应该使得M约等于T。然而，即使在这种设计下，由于存储数据key的分布通常不均匀，可能存在热点数据范围，因此依然存在生成空间利用率较低的run的情况。

![image-20220403111832859](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440454.png)

cold区域很可能被很多实际存储数据不多的run给填满，导致进行merge操作，又生成很多小的run给下一个level，恶性传递。

![image-20220403112120168](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440739.png)



上述问题主要是因为每个run都是独立的，并且run生成之后就不再去修改它，因此问题会被不断的放大。



Cow B+tree中的多个write batch实际上是存储在一个连续的储存空间里，因此可以尽可能避免空间的浪费。

这里提出了两种来避免空间浪费的策略：

1. 当前cow b+tree中失效数据较多的时候: space usage / live (unique) data size > C.

   原地对其进行压缩。提前进行原地merge，避免对下一级level进行写入。

   ![image-20220403104315849](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440329.png)

   ![image-20220403104327688](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440751.png)

2. 当有效数据达到超过上限时，与tiering merge策略相似，将其进行merge，并按照区间范围划分成不同的write batch写入下一级level中的cow B+tree。

   ![image-20220403105638954](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440289.png)



由于Cow B+tree可以原地压缩，当前Cow B+tree空间满了需要将数据分发给下一级level时，分发的有效数据会比较多。缓解了像很多个小run merge后再分发，继续出现很多个小run的情况。并且也不用担心多次写入少量数据到cold range Cow B+tree中会有像写入小run一样的副作用，因为只有在整个Cow B+tree空间比较满的时候，才会进行往下级分发数据的操作。总的来说，对磁盘的利用率较高，也减少了往下级level传递的次数。

![image-20220403113106522](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440701.png)





最后，用实验来评估一下提出的策略

![image-20220403114631976](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440296.png)

C是压缩系数，C越大，意味着cow b+tree的容量越大，相当于用更大的空间去换更少的写入次数。

![image-20220403131215926](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440528.png)

下图主要比较的是三种策略，对应的对空间的需求程度和写放大问题的严重程度。

![image-20220403114747230](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440977.png)

通过上述结果可以得出结论，cow b+tree的应用可以缓解写放大问题，通过使用更多的空间，能够达到与tiering差不多的缓解效果。但是实验结果中并没有写操作吞吐量的对比数据，猜测是比tiering的写吞吐量差了不少。



接下来是读取操作的实验，主要关注吞吐量，可以看到cow b+tree应用后的效果是最好的，与主要比较的tiering策略有很大的提升。

![image-20220403114801423](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091440349.png)

## 个人总结与思考

论文的思想本质上是用空间来换读取的时间，由于空间相对来说是最容易扩展的（加大硬盘容量、分布式），因此牺牲空间换时间是我们乐于看到的做法。

接下来从读写两方面具体来说一下：

先是写操作，作者的意图是，要保证与原始tiering merge策略，有基本相同的性能。但是，论文只比较了写放大的程度，没有将写入数据吞吐量的比较放出来，让我感觉很疑惑。我感觉文章的做法应该是会带来额外的开销的，因为多维护了一个B+Tree的数据结构，每个write batch除了要修改key - pointer to value- value这三种数据，还要对这个结构中的非叶子节点进行修改。因此写入的性能很可能是有下降的。

从偏工程的角度来说，作者提到tiering merge策略很可能带来很多实际存储空间很小的run，使得merge操作变多。作者自己的做法是在cow-b+tree上原地进行压缩，压缩实际是存在时间开销的，要重写磁盘。实际上，对于很多小的run来说，工程上完全可以使用守护线程，定期去检查，提前先将run stack（实际不是stack）中若干个小的run合并成大的run，这样也能缓解空间上的浪费，应该能接近与作者思路相似的压缩效果。

然后是读取操作，作者是希望使用B+树，使得遍历查找能够优化为“二分”查找，但是在max size of max run stack仅为10的的情况下，两种查找的在时间上真的会有较大差异吗，特别是还有布隆滤波器的情况，也不用遍历每个run里面的所有元素，差异应该会更小。

这里我的猜测是，读数据吞吐量的提升主要是因为作者提前压缩的策略使得平均能到找到数据的level层级更小，降低了需要查找的B+Tree的数量。如果tiering merge策略定期后台收集run进行压缩，我觉得一样能比原始的tiering merge有较大的提升。

最后总结一下（根据自己的推测，不一定正确）：

作者通过在写入操作时提前进行了压缩操作，实际上是通过牺牲写入速度（实验没放出来）换取读取速度。在工程上，由于一个run stack的大小通常设置为10这个量级，因此本身B+tree的查找优化可能没太大用。
