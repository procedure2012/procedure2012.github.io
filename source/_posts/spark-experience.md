---
title: 大玩具的陨落和超大玩具的崛起
tags: 技术
categories: 科普
date: 2020-04-08 13:11:27
---

> 读一些无用的书，做一些无用的事，花一些无用的时间，都是为了在一切已知之外，保留一个超越自己的机会，人生中一些很了不起的变化，就是来自这种时刻。
> 梁文道

B站上各种各样拿树莓派组装集群的视频，看的手痒痒。所以我和同学一起也搞了一个。[安装](https://procedure2012.com/2020/03/30/spark-install-guide/)完spark和hadoop之后，我和天哥把上学期一门数据分析课程的作业移植到了我们的集群上。原先的作业是通过评论和评分的数据集训练出一个神经网络，然后根据评论预测评分。我们因为设备的能力限制，没有用神经网络的分布式架构tensorflow或者pytorch之类的，而是选择了主要用来做数据计算而不是机器学习的hadoop+spark的架构。然后将原来的神经网络换成了多层感知机（MLP），用spark自带的pyspark.ml库实现MLP的训练和预测。总的来说，效果并不是很好，不过我们本来也只是当组装了一个大玩具玩一玩，搞清这些分布式计算系统的一些基本用法和结构。至于效果好不好？谁在乎呢？\\(￣︶￣)/

## 代码

简单说一下代码结构，因为只有一个很简单的文件就不传github了。大致分成三个部分：
1. 处理数据
    1. 去掉没用的符号（数字、空格、标点……）
        ```sh
        word = eliminateEnter(word)
        word = eliminateNumber(word)
        word = eliminatePunctuation(word)
        ```
    2. 去掉stopwords
    3. 将所有词变成词根
        ```sh
        if (len(newWord) > 0) and (word not in stopwordsSet):
            newComment.append(porter.stem(newWord.lower()))
        ```
    4. word2vec
        ```sh
        from pyspark.ml.feature import Word2Vec
        word2Vec = Word2Vec(vectorSize=10, minCount=0, inputCol="comment", outputCol="features")
        model = word2Vec.fit(documentDf)
        result = model.transform(documentDf)
        ```
2. 训练MLP
    1. 将评论所有词的wordvec取平均作为评论的wordvec（这一步其实在pyspark.ml.feature的库里自动实现了，我们也是偷懒才用的这个方案）
    2. 分出训练集和测试集
        ```sh
        splits = result.randomSplit([0.7, 0.3], 1234)
        train_set = splits[0]
        test_set = splits[1]
        ```
    3. 用训练集训练MLP
        ```sh
        from pyspark.ml.classification import MultilayerPerceptronClassifier
        mlp_trainer = MultilayerPerceptronClassifier(maxIter=20, layers=[10, 5,4, 5], blockSize=1, seed=123)
        mlp_model = mlp_trainer.fit(train_set.select("label","features"))
        result = mlp_model.transform(test_set)
        ```
3. 评估模型
    1. 用测试集计算准确度
        ```sh
        from pyspark.ml.evaluation import MulticlassClassificationEvaluator
        predictionAndLabels = result.select("prediction", "label")
        evaluator = MulticlassClassificationEvaluator(metricName="accuracy")
        ```
stopwords和词根是利用python的nltk库处理的，词的word2vec和评论的word2vec是利用pyspark.ml.feature库的函数。最后的训练和测评也是利用pyspark.ml.classification库函数。

## spark运行模式

spark里面有两类节点，一类是master，它记录了所有节点的位置和各种信息，用作控制；另一种是worker，它只负责处理各种计算任务。master和worker只是对物理节点的抽象，一个物理节点可以同时用作master或者worker，也可以只运行其中之一。

完成源代码之后，可以将源代码通过`spark-submit`命令提交到spark集群上。提交操作会生成一个driver用来编译python源文件成为集群可以运行的版本，然后将其分成不同的job，job可以串行也可以并行；再将每个job分成许多不同的stage，stage间的关系类似于map和reduce，map->reduce->map->reduce->......每个stage会进一步细化为一系列的task，task是数据并行，每个task执行一样的代码，但是处理不同的数据。然后集群会将这些task分配到不同的节点上去执行。

执行这些划分任务的也是driver，driver可以在提交源代码的机器上运行，也可以放在集群里的某一台机器运行。这些task最终的运行是需要占用资源（内存，cpu核心……）。这些资源被封装在一个个的executor里，每一个节点可以运行多个executor。spark需要通过resource manager来协调各个节点的资源，将executor建立在各个节点之上。常用的resource manager有两种：standalone和yarn。

无论用哪一种resource manager，有三个参数最为重要：executor的数量，每个executor占用的核心数，每个executor占用的内存。每个executor可以并行执行多个task，每个task使用一个核心，共用内存。两种resource manager有不太一样的设置方式来设置这些参数。

### standalone

#### num-executors

standalone不能直接设定executor的数量，但是可以通过`--conf spark.cores.max`参数来设定最大允许分配的核心数，最大可分配的executor=最大核心数/每个executor占用的核心数。
```sh
spark-submit --master spark://192.168.178.176:7077 \
            --conf spark.cores.max=4 mlp.py
```
`--master`参数是用来指定当前使用的resource manager，`spark://192.168.178.176:7077`表示使用spark自己做资源调度。`--master yarn`表示使用yarn作为资源调度。

#### executor-memory

通过`--executor-memory`参数指定每个executor占用的内存。
```sh
spark-submit --master spark://192.168.178.176:7077 \
            --conf spark.cores.max=4 \
            --executor-memory 512M mlp.py
```
默认每个executor最小分配内存为512M，可以通过修改spark设置参数文件中的参数来调整，具体设置可以参考[官方文档](https://spark.apache.org/docs/latest/running-on-yarn.html)中各个参数的意义和[Troubleshooting](#Troubleshooting)中的内容。

#### executor-cores

通过`--executor-cores`参数指定每个executor占用的内核。
```sh
spark-submit --master spark://192.168.178.176:7077 \
            --conf spark.cores.max=4 \
            --executor-memory 512M \
            --executor-cores 2 mlp.py
```
如果同时设置了多个参数，最终生成的executor数量一定同时满足这些条件，并且满足物理资源的限制，num-executor=min(spark.cores.max/executor-cores, 总物理核心数/executor-cores, 总物理内存数/executor-memory)。
<a href="https://imgur.com/OpWTct8.jpg"><img src="https://i.imgur.com/OpWTct8.png" title="standalone运行" /></a>

### yarn

#### [配置yarn](https://www.jianshu.com/p/31d89cc4d279)

yarn并不是spark自带的resource manager，而是属于hadoop的resource manager，需要先启动hadoop才能使用。再使用yarn之前，需要先配置几个参数。

在`hadoop3.1.3/etc/hadoop/capacity-scheduler.xml`中修改
```sh
<property>
    <name>yarn.scheduler.capacity.resource-calculator</name>
    <!-- <value>org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator</value> -->
    <value>org.apache.hadoop.yarn.util.resource.DominantResourceCalculator</value>
</property>
```
这样在yarn在请求exectuor资源的时候才会考虑用户设定的内核限制。

在`yarn-site.xml`中添加如下内容
```sh
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>1024</value>
</property>

<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>4</value>
</property>

<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
</property>

<property>
    <name>yarn.scheduler.increment-allocation-mb</name>
    <value>512</value>
</property>
```
- `yarn.nodemanager.resource.memory-mb`规定当前配置文件所在节点可用的物理内存。yarn不能自动探测可用内存，需要用户自己指定，默认值为8G。
- `yarn.nodemanager.resource.cpu-vcores`规定当前节点可用的虚拟内核。虚拟内核可以视为对cpu计算能力的评估，不会因为设定值高于实际值导致运行错误，只会影响运行速度，一般设置为和物理内核一样。如果有些节点有更高的计算能力，可以调高虚拟内核数量，这样可以分派更多的executor，和运算较慢的节点可以协调运行，充分利用资源。
- `yarn.nodemanager.vmem-check-enabled`是否进行虚拟内存越界的检查。在yarn中还可以设定物理内存和虚拟内存的比值，默认为1:2.1（1M物理内存对应2.1M虚拟内存）。开启检查可以在虚拟内存超过限制（实际物理内存*2.1）时抛出异常。根据需求调整是否检查。
- `yarn.scheduler.increment-allocation-mb`规定实际分配给每个executor的内存数应该是这个数值的整倍数，不是倍数时向上补齐。

#### num-executors

通过`--num-executors`参数设置最多允许分配的executor总数量。
```sh
spark-submit --master yarn \
            --num-executors 8 mlp.py
```

#### executor-memory

通过`executor-memory`参数设每个executor需要占用的内存。
```sh
spark-submit --master yarn \
            --num-executors 8 \
            --executor-memory 512M mlp.py
```

#### executor-cores

通过`executor-cores`参数设置每个executor需要占用的内核数。
```sh
spark-submit --master yarn \
            --num-executors 8 \
            --executor-memory 512M \
            --executor-cores 2 mlp.py
```
如果同时设置了多个参数，yarn也会和standalone模式一样限制exectuor数量是的所有条件都被满足，并且不超出物理资源限制。
<a href="https://imgur.com/sa2qHgm.jpg"><img src="https://i.imgur.com/sa2qHgm.png" title="spark on yarn运行" /></a>

### 注意

yarn模式还有一个特殊的概念叫做container，每个executor和一个container一一对应，在yarn的web面板里显示的是container的数量。除了分配的用于计算的container，yarn还会为每个应用建立一个ApplicationMaster用于调度，负责调度当前应用的ApplicationMaster也需要占用一个container，所以面板里显示container会比设定的num-executor多一个（在其他限制依然被满足的情况下）。ApplicationMaster的内存和核心用量可以通过修改`spark.yarn.am.memory（默认512M）`和`spark.yarn.am.cores（默认1）`修改。

无论是ApplicationMaster的container还是用于计算的container，它们实际申请到的内存都是我们设定的值+max(设定值*0.1, 384M)，然后再根据`yarn.scheduler.increment-allocation-mb`的值上取整。其实上述调整standalone和yarn资源分配的参数并不唯一，都可以在`spark-submit -h`下找到足够充分的解释来自由调整，[官方文档](https://spark.apache.org/docs/latest/running-on-yarn.html)也给出了许多参数解释。

## Troubleshooting

### deploy-mode

之前提到的，我们提交源码所产生的driver会进行任务划分和初始化。这个driver可以运行在提交源码的机器上，也可以运行在集群里的某一台机器上。通过`--deploy-mode`参数来设置。
```sh
# 运行在提交源码的机器上（client）
spark-submit --master yarn \
            --deploy-mode client
            --num-executors 8 \
            --executor-memory 512M \
            --executor-cores 2 mlp.py
# 运行在集群中某台机器上（cluster）自动分配
spark-submit --master yarn \
            --deploy-mode cluster
            --num-executors 8 \
            --executor-memory 512M \
            --executor-cores 2 mlp.py
```

### master开启hadoop之后连接不上worker

我们遇到的情况是因为一开始以为不仅master需要运行`./start-all.sh`，每个worker也要运行。所以导致每个worker都有各自的cluster-ID，master和worker的cluster-ID不一致肯定连不上。检查master中`hadoop3.1.3/dfs/name/current/VERSION`这个文件中的`cluster-ID`和worker中`hadoop3.1.3/dfs/data/current/VERSION`这个文件中是否一致。不一致的话将`current`文件夹全部删除，然后再master重新启动hadoop。

### [Neither spark.yarn.jars nor spark.yarn.archive is set](https://lovebear.top/2020/02/23/Spark_Yarn_Warning_Sparkjars/)

`spark-submit`提交应用时如果是用yarn作为resource manager的时候会出现这个warning。spark在hdfs硬盘里找不到用于支持yarn运行的jar，即使不管也不会引起任何错误，但是会导致spark在每次有任务提交的时候都要上传一次jar到hdfs的临时文件里，很费时间。
```sh
# 打包jar文件
jar cv0f spark-libs.jar -C $SPARK_HOME/jars/ .
# 上传到hdfs
hdfs dfs -mkdir -p /spark/jar
hdfs dfs -put spark-libs.jar /spark/jar
# 配置spark
cd $SPARK_HOME/conf
vim spark-defaults.conf
# 在spark-defaults。conf里写入
spark.yarn.archive=hdfs:///spark/jar/spark-libs.jar
```
如果没有`spark-defaults.conf`，需要先`cp spark-defaults.conf.template spark-defaults.conf`生成一份。文中所有spark开头的配置变量（比如spark.yarn.am.memory）都可以通过这种方式定义。不过我生成`spark-defaults.conf`之后重启spark时会遇到错误，原因不明。也可以通过在`spark-submi`后面添加`--conf 变量名=值`的方式临时添加。

### Python相关

spark2.4.5默认使用的时python2.7，如果想用其他版本的python，可以先安装好新的版本，然后在`spark2.4.5/conf/spark-env.sh`中添加希望使用的python的路径。
```sh
export PYSPARK_PYTHON=/usr/bin/python3
```
`--deploy-mode client`模式下，spark时是用本地python编译完成后再将task分发给其他节点。所以只要源码中用到的库在本地是安装好的，就可以顺利完成计算，即使集群中有节点没有安装这个库。`--deploy-mode cluster`模式不太清楚……

## 我回来了

正经的说完了，说一说我们搞出来的集群。总共有5块树莓派

| 主机名 | 树莓派 | 核心数 | 内存 |
| ---    | ---   |  ---  |  --- | 
|Master  | 4     | 4     | 4G   |
|worker1 | 4     | 4     | 4G   |
|worker2 | 3B    | 4     | 1G   |
|worker3 | 3B    | 4     | 1G   |
|brotherhood | 4 | 4     | 2G   |

最后一块brotherhood是我的，其他都是天哥大力支持的。可以看出其实性能非常差，造价还极高……当个大玩具看起来还是挺漂亮的，真要做点什么只怕是还不如电脑上多开几个虚拟机……
<a href="https://imgur.com/shMlLK4.jpg"><img src="https://i.imgur.com/shMlLK4.png" title="大号玩具，前面炫酷的是天哥的，后边几乎看不见的是我的" /></a>

我们一开始准备的数据就是上课作业给的数据，400M的csv文件，包含评价和对应的评分。做作业时得到的预测准确率大概在50%左右，运行时间大概在15-20分钟左右。然而，同样的文件用我们的集群跑的时候直接因为内存不足崩溃了……即使只用了200M的数据依然会导致内存不够……主要是有两个1G内存的节点成为了短板。反正只是为了研究分布式计算的基本原理，所以我们用900K的数据跑了一些，结果如下

| num-executors | cores per executor | memory per executor(MB) | TIme(min)  |
|    ---        | ---                | ---                     | ---        |
| 4             | 4                  | 1024                    | 17         |
| 4             | 4                  | 800                     | 5.6        |
| 4             | 4                  | 600                     | 21         |
| 4             | 2                  | 600                     | 5.7        |
| 4             | 1                  | 600                     | 5.7        |
| 4             | 1                  | 800                     | 5.3        |
| 5             | 2                  | 800                     | 6          |
| 5             | 2                  | 600                     | 5.1        |
| 6             | 1                  | 600                     | 5.5        |
| 7             | 1                  | 800                     | 5          |

900K训练出来的MLP准确率大概在23%左右。总体而言，大多数都是因为内存问题导致速度极慢。有的是因为内存溢出导致executor崩溃，重新连接和计算浪费大量时间。也有的是即使分配了很多的内核，但是因为内存不足以支持这么多内核同时进行计算，所以效率反倒不如同样内存但是核数更少的例子。影响速度的另一个原因可能是网络传输速度太慢，我们并没有把所有板子单独假设一个网络连接起来，而是直接用的家里现有的无线网络和路由器，传输速度肯定下降了好多倍。没有精力再单独架设网路了，所以网络和内存限制相比不清楚哪个才是主要约束……

现在想买一大堆树莓派然后组装一个大玩具的想法已经完全没有了。这东西做各种嵌入式系统还可以，但对于我来说性能实在太差，起码要买上几十个才能赶上一台没有独显的电脑。那既然已经要买几十个树莓派了，我为什么不再想的大一点呢？买上几十块树莓派钱再稍微加yi点，就能搞一台多核多卡的超大号玩具了！又能自己组装，看着还倍儿有面子！不要和我说什么云服务更划算，舍得买几十块树莓派的人根本不在乎再多掏一点！
<a href="https://imgur.com/Fq4P6bB.jpg"><img src="https://i.imgur.com/Fq4P6bB.png" title="这都额勒金德的东西知道吗？" /></a>

当然了，现在我既没有几十个的树莓派，也没有超大玩具，5块板子的集群还有4块是白嫖天哥的╮(￣▽￣)╭不过没关系，《大腕》里的疯言疯语十几年后不也都实现了吗？二十一世纪最重要的是什么？人才！我还是挺有信心的(ง •̀-•́)ง

---

终于在新学期第一堂课开始之前把这件事也弄完了，假期真的干了好多的事情。上课之后大家都要开始忙了，所以不能在一起搞这些东西。天哥也要把他的那些板子都搬回自己家去了。只剩我一块板子也没法搞了，只能暂时和这个集群说再见了（其实我还有别的板子，但是被我搞坏了……）。不过没关系，我还给自己准备了很多很多好玩的事情，可以一件一件慢慢来~~~