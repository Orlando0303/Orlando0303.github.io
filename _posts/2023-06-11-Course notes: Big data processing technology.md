---
title: 'Course notes: Big data processing technology'
date: 2023-06-11
permalink: /posts/2023/06/Course notes_Big data processing technology/
tags:
  - Big data
  - Course notes
---

Related courses: Distributed computing (96), big data processing technology (94)

# HDFS
### HDFS基本架构
**一个HDFS文件系统包括一个主控节点NameNode和一组DataNode从节点**
- 用户通过应用程序发送文件名或数据块号到NameNode，NameNode发送块号和位置到HDFS客户端，进而在DataNode找到对应数据块

### 组件及功能
- **文件块**
  - 文件切分成块（默认大小64M），以块为单位，每个块有多个副本存储在不同的机器上，副本数可在文件生成时指定（默认3） 
  - **好处：**
    - **支持大规模文件存储：** 一个大规模文件可以被分拆成若干个文件块，不同的文件块可以被分发到不同的节点上，因此，一个文件的大小不会受到单个节点的存储容量的限制，可以远远大于网络中任意节点的存储容量
    - **简化系统设计：** 大大简化了存储管理，因为文件块大小是固定的，这样就可以很容易计算出一个节点可以存储多少文件块；同时方便元数据管理，元数据不需要和文件块一起存储，可以由其他系统负责管理元数据
    - **适合数据备份：** 每个文件块都可以冗余存储到多个节点上，大大提高了系统的容错性和可用性

- **NameNode**
  - Namenode是一个中心服务器，单一节点，负责管理文件系统的名字空间(namespace)以及**客户端**对文件的访问 
    - 命名空间保存了两个核心的数据结构，即FsImage和EditLog
      - **FsImage：** 用于维护文件系统树以及文件树中所有的文件和文件夹的**元数据**——文件名，文件目录结构，文件属性（生成时间、副本数、文件权限等），以及每个文件的块列表以及块所在的DataNode的位置信息
      - **EditLog：** 操作日志文件中记录了所有针对文件的创建、删除、重命名等操作。
      在名称节点运行期间，HDFS的所有更新操作都是直接写到EditLog中，久而久之， EditLog文件将会变得很大。
      这在运行时无显著影响，但重启时名称节点需要先将FsImage里面的所有内容映像到内存中，然后再一条一条地执行EditLog中的记录，当EditLog文件非常大的时候，会导致名称节点启动操作非常慢，而在这段时间内HDFS系统处于安全模式，一直无法对外提供写操作，影响了用户的使用
      因此，采用**Secondary NameNode第二名称节点**
    - **第二名称节点：** 保存名称节点中对HDFS 元数据信息的备份，并减少名称节点重启的时间，单独运行在一台机器上 ![Secondary Namenode](/images/图片/图片2.png)
  - **文件操作：** NameNode负责文件元数据的操作，DataNode负责处理文件内容的读写请求，数据流不经过NameNode，只会询问它跟那个DataNode联系
  - 副本存放在那些DataNode上由NameNode来控制，根据全局情况做出块放置决定，读取文件时NameNode尽量让用户先读取最近的副本 **（数据就近原则）**，降低块消耗和读取时延
  - Namenode全权管理数据块的复制，它周期性地从集群中的每个Datanode接收**心跳信号**和**块状态报告(Blockreport)**。接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表

- **DataNode**
  - 一个数据块在DataNode以文件存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据（数据块的长度，块数据的校验和，以及时间戳）
  - **心跳信号：** 每3秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode 的心跳，则认为该节点不可用。
  - **块状态报告：** DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息。 
  - **动态拓展：** 集群运行中可以安全加入和退出一些机器 

### HDFS存储原理
- **数据冗余存储：** 作为一个分布式文件系统，为了保证系统的容错性和可用性，HDFS采用了多副本方式对数据进行冗余存储，通常一个数据块的多个副本会被分布到不同的数据节点上
  - **优点**：
    - 加快数据传输速度
    - 容易检查数据错误
    - 保证数据可靠性
- **数据存取：** 
  - **副本放置：**
    - 第1个副本：放置在上传文件的DN；如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点 
    - 第2个副本：放置在与第一个副本不同的机架的节点上 
    - 第3个副本：放置在与第一个副本相同集群的节点 
    - 更多副本：随机节点 
  - **数据就近原则：** 当客户端读取数据时，从名称节点获得数据块的副本位置列表，列表中包含了副本所在的数据节点，可以调用API来确定客户端和这些数据节点所属的机架ID，当发现某个数据块副本对应的机架ID和客户端对应的机架ID相同时，就优先选择该副本读取数据，否则，随机选择一个副本读取数据
- **数据错误与恢复：**
  - **名称节点出错：** 
  根据备份服务器Secondary NameNode中的FsImage和Editlog数据进行恢复
  - **数据节点出错：** 
  心跳信号超时：NameNode认为该节点不可用，这些数据节点就会被标记为“宕机”，节点上面的所有数据都会被标记为“不可读”，名称节点不会再给它们发送任何I/O请求。同时将该节点上的所有数据块副本进行复制（数据节点的不可用，导致一些数据块的副本数量小于冗余因子）
  - **数据出错：**
    - 客户端在读取到数据后，会采用md5和sha1对数据块进行校验，以确定读取到正确的数据
    - 在文件被创建时，客户端就会对每一个文件块进行信息摘录，并把这些信息写入到同一个路径的隐藏文件里面
    - 当客户端读取文件时，会先读取隐藏文件，然后根据隐藏文件中的信息摘录，读取文件块，最后对读取到的数据进行校验，以确定读取到正确的数据，否则客户端就会请求到另外一个数据节点读取该文件块，并且向名称节点报告这个文件块有错误，名称节点定期检查并且重新复制这个块

### HDFS可靠性的设计实现
- **安全模式**
刚启动的时候，等待每一个DataNode报告情况，退出安全模式的时候才进行副本复制操作
- **SecondaryNameNode**
  - NameNode失效怎么办？
    用来备份NameNode的元数据，以便在NameNode失效时能从SecondaryNameNode恢复出NameNode上的元数据
- **心跳包和副本重新创建**
  - 一个DataNode崩溃了怎么办？
    Hearbeat和副本重建
- **数据一致性**
  - 网络传输中，数据改变了怎么办？
    数据校验和CheckSum机制
- **租约**
  - 多个用户同时写一个文件怎么办？
    NameNode发放租约给客户端
- **回滚**
  - 版本升级出错了怎么办？
    回滚到前一个版本
----
# MapReduce
### 分布式并行编程
- **摩尔定律**: CPU性能大约每隔18个月翻一番, 从2005年开始摩尔定律逐渐失效 ，需要处理的数据量快速增加，人们开始借助于分布式并行编程来提高程序性能 
- **分布式程序**: 运行在大规模计算机集群上，可并行执行大规模数据处理任务，从而获得海量的计算能力
- **MapReduce**：谷歌公司最先提出了分布式并行编程模型MapReduce，Hadoop MapReduce是它的开源实现，后者比前者使用门槛低很多

### MapReduce编程模型概念
- **高度抽象：** MapReduce将复杂的、运行于大规模集群上的并行计算过程高度地抽象到了两个函数**Map**和**Reduce**
- **编程容易：** 不需要掌握分布式并行编程细节，也可以很容易把自己的程序运行在分布式系统上，完成海量数据的计算
- **分而治之的策略：** 一个存储在分布式文件系统中的大规模数据集，会被切分成许多独立的分片（split），这些分片可以被多个Map任务并行处理
- **计算向数据靠拢：** 把计算分布在集群中，而不是把数据汇总再计算。这是MapReduce的设计理念，而不是“数据向计算靠拢”，因为，移动数据需要大量的网络传输开销
- **Master/Slave架构：** 包括一个Master和若干个Slave。Master上运行JobTracker，Slave上运行TaskTracker 

### MapReduce的体系结构
MapReduce体系结构主要由四部分组成：Client、JobTracker、TaskTracker以及Task
![体系结构](/images/图片/%E5%9B%BE%E7%89%871.png)
- **Client：** 
  - 用来提交MapReduce程序到JobTracker端，提交的时候需要指定输入数据所在的路径、输出数据所在的路径、Map任务的个数、Reduce任务的个数、Map任务的输入格式、Map任务的输出格式、Map任务的类、Reduce任务的类等信息
  - 用户可通过Client提供的一些接口查看作业运行状态
- **JobTracker：** 
  - 负责资源监控和作业调度
  - 监控所有TaskTracker与Job的健康状况，一旦发现失败，就将相应的任务转移到其他节点
  - 跟踪任务的执行进度、资源使用量等信息，并将这些信息告诉任务调度器（TaskScheduler），而调度器会在资源出现空闲时，选择合适的任务去使用这些资源
- **TaskTracker：**
  - 周期性地通过“心跳”将本节点上资源的使用情况和任务的运行进度汇报给JobTracker，同时接收JobTracker 发送过来的命令并执行相应的操作（如启动新任务、杀死任务等）
  - 使用“slot”等量划分本节点上的资源量（CPU、内存等）。一个Task 获取到一个slot 后才有机会运行，而Hadoop调度器的作用就是将各个TaskTracker上的空闲slot分配给Task使用。slot 分为Map slot 和Reduce slot 两种，分别供MapTask 和Reduce Task 使用
- **Task：**
Task 分为Map Task 和Reduce Task 两种，均由TaskTracker 启动
  - Map Task：负责处理输入数据的一部分，将输入数据转换成中间结果，中间结果的格式为<key, value>，并将中间结果输出到本地磁盘
  - Reduce Task：负责处理Map Task 的输出结果，将中间结果按照key 进行合并，然后将合并后的结果输出到HDFS 中

### MapReduce工作流程
- 不同的Map任务之间不会进行通信
- 不同的Reduce任务之间也不会发生任何信息交换
- 用户不能显式地从一台机器向另一台机器发送消息
- 所有的数据交换都是通过MapReduce框架自身去实现的

![工作流程](/images/图片/%E5%9B%BE%E7%89%873.png)
#### 各个阶段的工作流程
- **Split（分片）**
HDFS 以固定大小的block 为基本单位存储数据，而对于MapReduce 而言，其处理单位是split。split 是一个逻辑概念，它只包含一些元数据信息，比如数据起始位置、数据长度、数据所在节点等。它的划分方法完全由用户自己决定
- **Map任务的数量**
Hadoop为每个split创建一个Map任务，split 的多少决定了Map任务的数目。大多数情况下，理想的分片大小是一个HDFS块
- **Reduce任务的数量**
  - 最优Reduce任务个数取决于集群中可用的reduce任务槽(slot)的数目
  - 通常设置比reduce任务槽数目稍微小一些的Reduce任务个数，这样可以预留一些系统资源处理可能发生的错误

----
# 分布式数据库HBase
### HBase简介
- HBase是一个高可靠、高性能、面向列、可伸缩的分布式数据库，是谷歌BigTable的开源实现，主要用来存储非结构化和半结构化的松散数据
- HBase的目标是处理非常庞大的表，可以通过水平扩展的方式，利用廉价计算机集群处理由超过10亿行数据和数百万列元素组成的数据表 
- **关系数据库已经流行很多年，并且Hadoop已经有了HDFS和MapReduce，为什么需要HBase?**
  - Hadoop可以很好地解决大规模数据的离线批量处理问题，但是，受限于MapReduce编程框架的高延迟数据处理机制，使得Hadoop无法满足大规模数据实时处理应用的需求
  - HDFS面向批量访问模式，不是随机访问模式
  - 传统的通用关系型数据库无法应对在数据规模剧增时导致的系统扩展性和性能问题（分库分表也不能很好解决）
  - 传统关系数据库在数据结构变化时一般需要停机维护；空列浪费存储空间
  - 因此，业界出现了一类面向半结构化数据存储和处理的高可扩展、低写入/查询延迟的系统，例如，键值数据库、文档数据库和列族数据库（如BigTable和HBase等）
- **HBase与传统的关系数据库的区别**
  - **数据类型**：关系数据库采用关系模型，具有丰富的数据类型和存储方式，HBase则采用了更加简单的数据模型，它把数据存储为未经解释的字符串
  - **数据操作**：关系数据库包含了丰富的操作，涉及复杂的多表连接。HBase操作则不存在复杂的表与表之间的关系，只有简单的插入、查询、删除、清空等
  - **存储模式**：关系数据库是基于行模式存储的。HBase是基于列存储的，每个列族都由几个文件保存，不同列族的文件是分离的
  - **数据索引**：关系数据库通常可以针对不同列构建复杂的多个索引，以提高数据访问性能。HBase只有一个索引——行键
  - **数据维护**：在关系数据库中，更新操作会用最新的当前值去替换记录中原来的旧值。HBase更新操作并不会删除数据旧版本，而是生成一个新版本，旧版本仍然保留
  - **可伸缩性**：关系数据库很难实现横向扩展，纵向扩展的空间也比较有限。相反，HBase和BigTable这些分布式数据库就是为了实现灵活的水平扩展而开发的，能够轻易地通过在集群中增加或者减少硬件数量来实现性能的伸缩
### HBase数据模型
- **数据模型概述**
  - HBase是一个稀疏、多维度、排序的映射表，这张表的索引是行键、列族、列限定符和时间戳
  - 每个值是一个未经解释的字符串，没有数据类型
  - 用户在表中存储数据，每一行都有一个可排序的行键和任意多的列
  - 表在水平方向由一个或者多个列族组成，一个列族中可以包含任意多个列，同一个列族里面的数据存储在一起
  - 列族支持动态扩展，可以很轻松地添加一个列族或列，无需预先定义列的数量以及类型，所有列均以字符串形式存储
  - HBase中执行更新操作时，并不会删除数据旧的版本，而是生成一个新的版本，旧有的版本仍然保留（这是和HDFS只允许追加不允许修改的特性相关的）
- **数据模型概念**
  ![HBase数据模型概念](/images/图片/图片4.png)
  - **表**：HBase采用表来组织数据，表由行和列组成，列划分为若干个列族
  - **行**：每个HBase表都由若干行组成，每个行由行键（row key）来标识。
  - **列族**：一个HBase表被分组成许多“列族”（Column Family）的集合，它是基本的访问控制单元
  - **列限定符**：列族里的数据通过列限定符（或列）来定位
  - **单元格**：在HBase表中，通过行、列族和列限定符确定一个“单元格”（cell），单元格中存储的数据没有数据类型，总被视为字节数组byte[]
  - **时间戳**：每个单元格都保存着同一份数据的多个版本，这些版本采用时间戳进行索引
- **数据坐标**
  HBase中需要根据行键、列族、列限定符和时间戳来确定一个单元格，因此，可以视为一个“四维坐标”，即 **[行键, 列族, 列限定符, 时间戳]**
- **面向列的存储**
  ![对比图](/images/图片/图片5.png)
  ![示例](/images/图片/图片6.png)
  ![示例](/images/图片/图片7.png)
### HBase的实现原理
- **功能组件**
  - **库函数**：链接到每个客户端
  - **一个Master主服务器**
    - 负责管理和维护HBase表的分区信息，维护Region服务器列表，分配Region，负载均衡
    - 客户端并不是直接从Master主服务器上读取数据，而是在获得Region的存储位置信息后，直接从Region服务器上读取数据
    - 客户端并不依赖Master，而是通过Zookeeper获得Region位置信息，大多数客户端甚至从来不和Master通信，这种设计方式使得Master负载很小 
  - **许多个Region服务器**
    - 负责存储和维护分配给自己的Region，处理来自客户端的读写请求
- **表和Region**
  - 一个HBase表被划分成多个Region，一个Region会分裂成多个新的Region
    - 开始只有一个Region，后来不断分裂
    - Region拆分操作非常快，因为拆分之后的Region读取的仍然是原存储文件，直到“合并”过程把存储文件异步地写到独立的文件之后，才会读取新文件
  - 不同的Region可以分布在不同的Region服务器上
  ![示例](/images/图片/图片8.png)
    -  每个Region的最佳大小取决于单台服务器的有效处理能力
    -  同一个Region不会被分拆到多个Region服务器
    -  每个Region服务器存储10-1000个Region
- **Region的定位**
  HBase使用三层结构来保存region的位置信息
  ![示例](/images/图片/图片10.png)
  ![示例](/images/图片/图片9.png)
  - **元数据表**，又名.META.表，存储了Region和Region服务器的映射关系
    - 当HBase表很大时， .META.表也会被分裂成多个Region
  - **根数据表**，又名-ROOT-表，记录所有元数据的具体位置
    - -ROOT-表只有唯一一个Region，名字是在程序中被写死的
  - Zookeeper文件记录了-ROOT-表的位置
  - 为了加快访问速度，.META.表的全部Region都会被保存在内存中
  - 客户端访问数据时的“三级寻址”
  - 为了加速寻址，客户端会缓存位置信息，同时，需要解决缓存失效问题
  - 寻址过程客户端只需要询问Zookeeper服务器，不需要连接Master服务器
### HBase运行机制
- **系统架构**
![示例](/images/图片/图片11.png)
  - **客户端**：包含访问HBase的接口，同时在缓存中维护着已经访问过的Region位置信息，用来加快后续数据访问过程
  - **Zookeeper服务器**：集群管理工具。可以帮助选举出一个Master作为集群的总管，并保证在任何时刻总有唯一一个Master在运行，这就避免了Master的“单点失效”问题
  - **Master**：主服务器。
    - 管理用户对表的增加、删除、修改、查询等操作
    - 实现不同Region服务器之间的负载均衡
    - 在Region分裂或合并后，负责重新调整Region的分布
    - 对发生故障失效的Region服务器上的Region进行迁移
  - **Region服务器**：负责维护分配给自己的Region，并响应用户的读写请求
- **Region服务器工作原理**
  ![示例](/images/图片/图片12.png)
  Region服务器内部管理了一系列Region对象和一个HLog文件，其中HLog是磁盘上面的记录文件它记录着所有的更新操作。每个Region对象又是由多个Store组成的，**每个Store对应了表中的一个列族的存储**。每个Store又包含了一个MemStore和若干个StoreFile，其中MemStore是在内存中的缓存，保存最近更新的数据，StoreFile是磁盘中的文件。
  - **用户读写数据过程** 
    - 用户写入数据时，被分配到相应Region服务器去执行
    - 用户数据首先被写入到MemStore和Hlog中
    - 只有当操作写入Hlog之后，commit()调用才会将其返回给客户端
    - 当用户读取数据时，Region服务器会首先访问MemStore缓存，如果找不到，再去磁盘上面的StoreFile中寻找
  - **缓存的刷新**
    - 系统会周期性地把MemStore缓存里的内容刷写到磁盘的StoreFile文件中，清空缓存，并在Hlog里面写入一个标记
    - 每次刷写都生成一个新的StoreFile文件，因此，每个Store包含多个StoreFile文件
    - 每个Region服务器都有一个自己的HLog 文件，每次启动都检查该文件，确认最近一次执行缓存刷新操作之后是否发生新的写入操作；如果发现更新，则先写入MemStore，再刷写到StoreFile，最后删除旧的Hlog文件，开始为用户提供服务
  - **StoreFile的合并**
    - 每次刷写都生成一个新的StoreFile，数量太多，影响查找速度
    - 调用Store.compact()把多个合并成一个
    - 合并操作比较耗费资源，只有数量达到一个阈值才启动合并
- **Store工作原理**
  - Store是Region服务器的核心
  - 多个StoreFile合并成一个StoreFile
  - 单个StoreFile过大时，又触发分裂操作，1个父Region被分裂成两个子Region
- **HLog工作原理**
  - **HLog保证系统恢复**
    - HBase系统为每个Region服务器配置了一个HLog文件，它是一种预写式日志（Write Ahead Log）
    - 用户更新数据必须首先写入日志后，才能写入MemStore缓存，并且，直到MemStore缓存内容对应的日志已经写入磁盘，该缓存内容才能被刷写到磁盘
  - **恢复过程**
    - Zookeeper会实时监测每个Region服务器的状态，当某个Region服务器发生故障时，Zookeeper会通知Master
    - Master首先会处理该故障Region服务器上面遗留的HLog文件，这个遗留的HLog文件中包含了来自多个Region对象的日志记录
    - 系统会根据每条日志记录所属的Region对象对HLog数据进行拆分，分别放到相应Region对象的目录下，然后，再将失效的Region重新分配到可用的Region服务器中，并把与该Region对象相关的HLog日志记录也发送给相应的Region服务器
    - Region服务器领取到分配给自己的Region对象以及与之相关的HLog日志记录以后，会重新做一遍日志记录中的各种操作，把日志记录中的数据写入到MemStore缓存中，然后，刷新到磁盘的StoreFile文件中，完成数据恢复
    - 共用日志优点：提高对表的写操作性能；缺点：恢复时需要分拆日志
### Hbase性能优化
- **行键（Row Key）**
  - 访问hbase table中的行，只有三种方式：
    - 通过单个row key访问
    - 通过row key的range
    - 全表扫描
  - Row key可以是任意字符串(最大长度是 64KB，实际应用中长度一般为 10-100bytes)，在hbase内部，row key保存为字节数组
  - 行键是按照字典序存储，因此，设计key时，要充分利用这个排序特点，将经常一起读取的数据存储到一块，将最近可能会被访问的数据放在一块
    - 字典序的排序结果是：1,10,100,11,12,13,14,15,16,17,18,19,2,20,21, …, 9, 91,92,93,94,95,96,97,98,99
  - 例如：如果最近写入HBase表中的数据是最可能被访问的，可以考虑将时间戳作为行键的一部分 
- **InMemory**
  创建表的时候，可将表放到Region服务器的缓存中，保证在读取的时候被cache命中
- **Max Version**
  创建表的时候，可设置表中数据的最大版本，如果只需要保存最新版本的数据，那么可以设置setMaxVersions(1)
- **Time To Live**
  创建表的时候，可设置表中数据的存储生命期，过期数据将自动被删除
### HBase编程实践
- **启动HBase**
  ![启动](/images/图片/图片13.png)
- **验证**
  ![验证](/images/图片/图片14.png)
- **查看数据库状态-status **
  ![状态](/images/图片/图片15.png)
- **表操作**
  ![表操作](/images/图片/图片16.png)
  ![表操作](/images/图片/图片17.png)
- **插入记录：put；获取记录：get；更新也是put（HBase中修改是直接增加记录）**
  ![表操作](/images/图片/图片18.png)
- **全表扫描：scan**
  ![表操作](/images/图片/图片19.png)
- **删除记录：delete**
  ![表操作](/images/图片/图片20.png)
- **删除整行：deleteall**
  ![表操作](/images/图片/图片21.png)
- **查询表中有多少行：count** 
  ![表操作](/images/图片/图片22.png)
- **清空表：truncate** 
  ![表操作](/images/图片/图片23.png)
- **删除表：disable、drop**
  ![表操作](/images/图片/图片24.png)
----
# 数据仓库Hive
### Hive概述
- Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供类SQL查询功能，除了不支持更新、索引和事务，几乎SQL的其它特征都能支持 
- 本质是将HQL转换为MapReduce程序
### Hive的架构简介
- **在Hadoop生态圈的位置**
![Hive架构](/images/图片/图片25.png)
- **接口**
  ![Hive架构](/images/图片/图片26.png)
### Hive工作原理
- **查询转换成MapReduce作业的过程**
  ![Hive原理](/images/图片/图片27.png)
  - 当用户向Hive输入一段命令或查询时，Hive需要与Hadoop交互工作来完成该操作。首先，驱动模块接收该命令或查询编译器。接着，对该命令或查询进行解析编译。然后，由优化器对该命令或查询进行优化计算。最后该命令或查询通过执行器进行执行。
  - 执行器通常的任务是启动一个或多个MapReduce任务，有时也不需要启动MapReduce任务，像执行包含*的操作（如select * from 表）时
- **join转化为MapReduce任务的过程**
  ![Hive原理](/images/图片/图片28.png)
  - 在Map阶段，表user中记录(uid,name)映射为键值对(uid,<1,name>)，表order中记录(uid, orderid)映射为键值对(uid,<2, orderid >)
    - 1,2是表user和order的标记位
    - (1,Lily)->(1,<1,Lily>)， (1,101)->(1,<2,101>)
  - 在Shuffle、Sort阶段， (uid,<1,name>)和(uid,<2, orderid >)按键uid的值进行哈希，然后传送给对应的Reduce机器执行，并在该机器上按表的标记位对这些键值对进行排序
    - (1,<1,Lily>)、(1,<2,101>)和 (1,<2,102>)传送到同一台Reduce机器上，并按该顺序排序
    - (2,<1,Tom>)和(2,<2,103>)传送到同一台Reduce机器上，并按该顺序排序
  - 在Reduce阶段，对同一台Reduce机器上的键值对，根据表标记位对来自不同表的数据进行笛卡尔积连接操作，以生成最终的连接结果。
    - (1,<1,Lily>)∞(1,<2,101>)->(Lily, 101) 
    - (1,<1,Lily>)∞ (1,<2,102>)->(Lily, 102)
    - (2,<1,Tom>) ∞(2,<2,103>)->(Tom, 103)
- group by转化为MapReduce任务的过程
  ![Hive原理](/images/图片/图片29.png)
  - 在Map阶段，表score中记录(rank,level)映射为键值对(<rank,level> , count(rank,level))
    - score表的第一片段中有两条记录(A,1)，(A,1)->(<A,1> ,2) 
    - score表的第二片段中有一条记录(A,1)，(A,1)->(<A,1> ,1) 
  - 在Shuffle、Sort阶段， (<rank,level> ,count(rank,level))按键<rank,level>的值进行哈希，然后传送给对应的Reduce机器执行，并在该机器上按<rank,level>的值对这些键值对进行排序
    - (<A,1>,2)和 (<A,1>,1)传送到同一台Reduce机器上，按到达顺序排序
    - (<B,2>,1>)传送到另一台Reduce机器上
  - 在Reduce阶段，对Reduce机器上的这些键值对，把具有相同<rank,level>键的所有count(rank,level)值进行累加，生成最终结果。
    - (<A,1>,2>)+ (<A,1>,1>)->(A,1,3)
    - (<B,2>,1)->(B,2,1)
### Hive组成
- **用户接口**：Hive shell, thrift客户端, web等 
- **Thrift服务器**
- **元数据库**：Derby, Mysql 
- **解析器**：包括解释器、编译器、优化器和执行器。查询计划由MapReduce调用执行 
- **Hadoop**：数据仓库和查询计划存储在HDFS上，计算过程由MapReduce执行 
### 调优
- **数据存储模型**
  ![调优](/images/图片/图片30.png)
  - **内部表（Table）**
    - 与数据库中的 Table 在概念上是类似
    - 每一个 Table 在 Hive 中都有一个相应的目录存储数据
      - 例如，一个表test，它在 HDFS 中的路径为：/ warehouse/test。warehouse是在hive-site.xml 中由 ${hive.metastore.warehouse.dir} 指定的数据仓库的目录
      - 所有Table 数据（不包括 External Table）都保存在这个目录中
    - 删除表时，元数据与数据都会被删除
  - **外部表**
    - 指向已经在 HDFS 中存在的数据，可以创建 Partition
    - 它和内部表 在元数据的组织上是相同的，而实际数据的存储则有较大的差异
  - 内部表的创建过程和数据加载过程，在加载数据的过程中，实际数据会被移动到数据仓库目录中；之后对数据的访问将会直接在数据仓库目录中完成。**删除表时，表中的数据和元数据将会被同时删除**
  - 外部表只有一个过程，加载数据和创建表同时完成，并不会移动到数据仓库目录中,只是与外部数据建立一个链接。**当删除一个外部表时，仅删除链接**
  - **分区表（按字段分）**
    - 分区表是指在表中按照某个字段进行分区，分区字段的值相同的数据会被放到同一个分区中
    - Partition 对应于数据库的 Partition 列的密集索引
    - 在 Hive 中，表中的一个 Partition 对应于表下的一个目录，所有的 Partition 的数据都存储在对应的目录中
      - 例如：test表中包含 date 和 city 两个 Partition，‍则对应于date=20130201, city = bj 的HDFS 子目录‍：/warehouse/test/date=20130201/city=bj
      - 对应于date=20130202, city=sh 的HDFS 子目录为：/warehouse/test/date=20130202/city=sh
  - **桶表（按值分）**
    - 桶表是对数据进行哈希取值，然后放到不同文件中存储
    - 创建表
      - create table bucket_table(id string) clustered by(id) into 4 buckets;
    - 加载数据
      - set hive.enforce.bucketing = true;
      - insert into table bucket_table select name from stu; 
      - insert overwrite table bucket_table select name from stu;
    - 数据加载到桶表时，会对字段取hash值，然后与桶的数量取模。把数据放到对应的文件中。
    - 抽样查询
      - select * from bucket_table tablesample(bucket 1 out of 4 on id);
  - **分区表与桶表的优化目标和适用场景**
    - **分区表**：减少数据扫描量，提高查询性能。适用于某个列具有较低基数（即唯一值的数量较少）且经常用于查询条件的场景。通过分区表，可以在执行查询时只扫描与分区条件匹配的分区，而不是整个表，从而提高查询性能。
    - **桶表**：提高数据存储的均匀性。适用于某个列具有较高基数（即唯一值的数量较多）且经常用于 join 操作或查询条件的场景。通过桶表，可以在进行 join 操作时实现优化的 bucketed-map-join，从而提高 join 性能。
- **列裁剪（Column Pruning）**
  - 读取数据时，只读取需要用到的列，而忽略其他列。例如： SELECT a,b FROM t WHERE e<10; 
- **分区裁剪（Partition Pruning）** 
  - 在查询过程中减少不必要的分区。
  - 例如： SELECT * FROM (SELECT c1, COUNT(1) FROM T GROUP BY c1) subq WHERE subq.prtn=100; 
  - 要实现分区裁剪，须设置hive.optimize.pruner=true 
- **Map Join操作** 
  - Map Join操作无须reduce操作就可以在map阶段全部完成，前提是在map过程中可以访问到全部需要的数据
- **Group By操作** 
  - Map端部分聚合。很多聚合操作都可以先在map端进行部分聚合，然后在reduce端得出最终结果。相关参数：hive.map.aggr=true，hive.groupby.mapaggr.checkinterval=100000 
  - 有数据倾斜（数据分布不均匀）时进行负载均衡。相关参数： hive.groupby.skewindata=true(默认为false) 
### 适用环境
- Hive不能提供排序和查询cache功能，也不提供在线事务处理，不提供实时查询和记录级的更新 
- Hive能很好地处理不变的大规模数据集上批量任务 
- Hive具有很好的**可扩展性**（基于Hadoop平台）和**延展性**（结合MapReduce和用户自定义的函数库） 
- Hive拥有良好的容错性和低约束的数据输入格式 
- Hive vs SQL 
  ![Hive vs SQL](/images/图片/图片31.png)
----
# Spark
### Spark简介
- **Hadoop缺点**
  - 表达能力有限：Map、Reduce
  - 磁盘I/O开销大
  - 延迟高
    - 任务之间的衔接涉及IO开销
    - 在前一个任务执行完成之前，其他任务就无法开始，难以胜任复杂、多阶段的计算任务
- **Spark与Hadoop的对比**
  - Spark的计算模式也属于MapReduce，但不局限于Map和Reduce操作，还提供了多种数据集操作类型，编程模型比Hadoop MapReduce更灵活
  - Spark提供了**内存计算**，可将中间结果放到内存中，对于迭代运算效率更高
  - Spark基于**DAG的任务调度**执行机制，要优于Hadoop MapReduce的迭代执行机制 
- **Spark特点**
  - **运行速度快**：使用DAG执行引擎以支持循环数据流与内存计算
  - **容易使用**：支持使用Scala、Java、Python和R语言进行编程，可以通过Spark Shell进行交互式编程 
  - **通用性**：Spark提供了完整而强大的技术栈，包括SQL查询、流式计算、机器学习和图算法组件
  - **运行模式多样**：可运行于独立的集群模式中，可运行于Hadoop中，也可运行于Amazon EC2等云环境中，并且可以访问HDFS、Cassandra、HBase、Hive等多种数据源 
- **Scala简介**
  - Scala是一种多范式编程语言，设计初衷是要集成面向对象编程和函数式编程的各种特性。它运行于Java虚拟机之上，可以编译成Java字节码，也可以编译成JavaScript，方便Scala程序员利用Web浏览器或Node.js来运行Scala程序
  - Scala具备强大的并发性，支持函数式编程，可以更好地支持分布式系统
  - Scala语法简洁，能提供优雅的API
  - Scala是Spark的主要编程语言，但Spark还支持Java、Python、R作为编程语言
  - Scala的优势是提供了REPL（Read-Eval-Print Loop，交互式解释器），提高程序开发效率
### Spark生态系统
**目的**：提供一套完整的大数据处理解决方案，同时支持批处理、交互式查询和流数据处理
![Spark生态系统](/images/图片/图片32.png)
  - **Spark Core**：Spark的核心组件，提供了Spark的基本功能，包括任务调度、内存管理、错误恢复、与存储系统交互等
  - **Spark SQL**：Spark SQL是Spark的数据处理组件，提供了SQL查询和结构化数据处理功能
  - **Spark Streaming**：Spark Streaming是Spark的流式数据处理组件，提供了流式数据处理功能
  - **MLlib**：MLlib是Spark的机器学习组件，提供了常用的机器学习功能
  - **GraphX**：GraphX是Spark的图计算组件，提供了图计算功能
  - SparkR：SparkR是Spark的R语言接口，提供了R语言的编程接口
  - Spark集群管理器：Spark可以运行于独立的集群模式中，也可以运行于Hadoop中，还可以运行于Amazon EC2等云环境中
  - Spark Shell：Spark Shell是Spark提供的交互式编程工具，支持Scala、Python和R语言
- **应用场景**
![Spark应用场景](/images/图片/图片33.png)
### Spark运行架构
Spark运行架构包括集群资源管理器（Cluster Manager）、运行作业任务的工作节点（Worker Node）、每个应用的任务控制节点（Driver）和每个工作节点上负责具体任务的执行进程（Executor）
- **Spark运行架构特点**
  - 每个Application都有自己专属的Executor进程，并且该进程在Application运行期间一直驻留。Executor进程以多线程的方式运行Task
  - Spark运行过程与资源管理器无关，只要能够获取Executor进程并保持通信即可
  - Task采用了数据本地性和推测执行等优化机制
- **Executor的优点**
  - 利用多线程来执行具体的任务，减少任务的启动开销
  - 每个Executor中有一个BlockManager存储模块，会将内存和磁盘共同作为存储设备，有效减少I/O开销
    - **数据块管理**：BlockManager 是 Spark 的核心组件，负责管理数据块（Block）。这些数据块可以是 RDD 分区，广播变量或 Shuffle 文件。BlockManager 提供了一个抽象层，使 Spark 可以透明地从内存或硬盘中获取数据块，无论数据块是存储在本地还是在网络的其他节点上。
    - **容错与数据复制**：BlockManager 负责在节点间复制数据块，以提供容错能力。此外，它也可以在内存压力下将数据块溢写到磁盘，从而支持处理比物理内存更大的数据集。
    - **进程间通信**：在 Spark 应用程序中，每个 Executor 进程都有一个 BlockManager 实例。这些 BlockManager 实例通过 BlockManagerMaster 进行通信，以跟踪整个应用中的数据块存储情况。
    - **内存与硬盘间调度**：BlockManager 负责管理数据块在内存和硬盘之间的调度。当内存资源压力较大时，数据块会被溢写到硬盘。相反，当内存资源充足时，数据块会从硬盘加载到内存，以提高数据处理速度。
- **RDD（弹性分布式数据集）**
  - **弹性（Resilient）**： “有恢复能力的”或“有弹回原状的能力”。如果集群中的某个节点发生故障，Spark 可以通过 RDD 重新计算丢失的数据，而不需要从硬盘或其他地方恢复，这大大提高了处理速度
  - **分布式（Distributed）**：这意味着数据集可以被分割成很多小部分，这些小部分可以在许多不同的计算机或计算机集群节点上进行处理。这样可以让大数据处理任务并行化，从而提高处理速度。
  - **数据集（Dataset）**：想要处理的数据。这些数据可以是任何类型，例如，这些数据可能是用户的点击流日志，或者是一大堆科学实验的结果。
  - **why RDD？**
    - 为了解决大规模数据处理中的两个关键问题：**效率和容错性**
    - RDD 通过提供一个在内存中存储数据的抽象模型，使得 Spark 可以高效地进行迭代计算，并在出现错误时快速恢复数据。避免反复进行磁盘IO
  - **RDD的解决策略——只读性**
  **只读性是指不能直接修改RDD的内容，但可以通过转换操作生成新的RDD。**
    - **并行数据处理**：RDD将数据集分割成多个分区，每个分区在集群中的一个节点上独立并行处理。这种分布式的并行处理方式加快了大规模数据的处理速度，从而提高计算效率。
    - **惰性计算**：RDD上的操作并不会立即执行，而是在实际需要结果时才开始计算，这种机制称为惰性计算。这种设计使Spark可以进行操作优化，如合并多个操作，避免不必要的数据加载和计算，从而进一步提高计算效率。
    - **容错性**：RDD具有强大的容错性。当某个节点出现故障，Spark可以使用其他节点的数据或者重新计算丢失的数据进行恢复。这种容错机制保证了大规模数据处理的稳定性，使得Spark能够在面临节点故障的情况下依然能够持续运行。
- **DAG（有向无环图），反映RDD之间的转换关系**
  - 在 Spark 中对 RDD 进行一系列的转换操作（如 map，filter，reduce 等）时，Spark 会使用 DAG 来记录这些操作。这些操作不会立即执行，而是在遇到行动操作（action）时，根据 DAG 中的信息，Spark 会生成一个执行计划，然后按照这个计划进行计算。这种基于 DAG 的执行模型使得 Spark 可以进行更多的优化，比如合并操作，避免不必要的数据加载等，从而提高计算效率。**（惰性）**
  - DAG 记录 RDD 之间的转换关系，使得 Spark 可以在节点出错时，根据 DAG 重新计算丢失的数据，这提高了 Spark 的容错能力。**（容错性）**
  - DAG 可以清晰地展示出 RDD 之间的依赖关系，对于理解和调试 Spark 程序非常有帮助
- **Application**：在 Spark 中，一个 Application 是用户提交的一个 Spark 程序或作业，它由一个驱动程序（Driver Program）和多个 Executor 组成。驱动程序包含应用程序的 main() 函数，并且定义了数据集（RDDs）以及对它们的转换和操作。Executor 运行这些操作并返回结果给驱动程序。
- **Executor**：Executor 是 Spark 应用程序在集群中的一个运行实例，它是 Spark 应用程序运行的基本单位。每个 Executor 都有固定数量的内核（cores）和内存，它可以运行多个并行任务。Executor 在 Spark 应用程序启动时被分配，并在整个应用程序运行期间保持不变。Executor 可以在本地或在集群的节点上运行。
- **Job**：Job 是 Spark 中的一项操作，例如，一个 action 会触发一个 Job。Job 是由多个 Stage 组成的，Stage 之间可能会有依赖关系。例如，如果你在一个 RDD 上执行了一个 action，这会触发一个 Job。Job 会被划分成多个 Stage，每个 Stage 包含多个 Task。
- **Stage**：Stage 是 Job 的一个组成部分，每个 Job 都由多个 Stage 组成。一个 Stage 包含对 RDD 的一系列转换操作。如果在 RDD 的转换过程中遇到 Shuffle 操作（例如，groupByKey 或 reduceByKey），那么就会产生一个新的 Stage，因为 Shuffle 操作需要把数据在不同的 Executor 之间进行交换，这需要大量的网络 I/O，所以 Spark 会尽量减少 Shuffle 操作的次数。
- **Task**：Task 是 Spark 应用程序中要执行的最小工作单元，每个 Task 都在一个 Executor 的一个内核上运行。每个 Task 都是在数据的一个分区上运行的一组转换（Transformations）操作。
- **分区**：分区是数据的基本单位，被用来在集群中并行处理。一个RDD或者DataFrame在Spark中是由多个分区组成的，每个分区都会在集群中的一个节点上处理。
  - 分区的主要优点是它们允许并行处理，这使得Spark能够在多个节点上并行处理数据。每个分区都是RDD中的一个数据子集，可以在集群的一个节点上独立地处理。分区的数量和如何分区可以在创建RDD或DataFrame时指定，或者由Spark自动决定。
- 一个Application由一个Driver和若干个Job构成，一个Job由多个Stage构成，一个Stage由多个没有Shuffle关系的Task组成
- 当执行一个Application时，Driver会向集群管理器申请资源，启动Executor，并向Executor发送应用程序代码和文件，然后在Executor上执行Task，运行结束后，执行结果会返回给Driver，或者写到HDFS或者其他数据库中
### 运行流程
![运行流程](/images/图片/%E5%9B%BE%E7%89%8734.png)
- 首先为Application构建起基本的运行环境，即由Driver创建一个SparkContext，进行资源的申请、任务的分配和监控
- 资源管理器为Executor分配资源，并启动Executor进程
- SparkContext根据RDD的依赖关系构建DAG图，DAG图提交给DAGScheduler解析成Stage，然后把一个个TaskSet提交给底层调度器TaskScheduler处理；Executor向SparkContext申请Task，Task Scheduler将Task发放给Executor运行，并提供应用程序代码
- Task在Executor上运行，把执行结果反馈给TaskScheduler，然后反馈给DAGScheduler，运行完毕后写入数据并释放所有资源 
##### 从编程角度看
- **创建SparkContext**：SparkContext是Spark的主入口点，它连接到Spark集群并协调和监控Spark应用程序的运行。通常，在启动一个Spark应用程序时，第一步就是创建一个SparkContext
- **读取数据**：Spark可以从各种数据源读取数据，包括HDFS、本地文件系统、S3、Cassandra、HBase、JDBC源等。数据被读取后，会被转换成RDD或DataFrame
- **转换操作**：在数据被读入后，我们可以对其执行各种转换操作，例如map、filter、reduceByKey等。这些操作都是惰性的，只有当触发行动操作时才会实际执行
- **行动操作**：行动操作会触发实际的计算，例如count、collect、saveAsTextFile等。执行行动操作时，Spark会提交一个Job，然后将Job分解成一组Stages，每个Stage包含一组可以并行执行的Tasks
- **任务调度和执行**：Spark将任务分发给集群中的各个节点上的Executor进行执行。每个任务在一个数据分区上执行，并且尽可能在数据所在的节点上执行，以利用数据局部性
- **结果收集**：当所有的任务都执行完毕后，结果会被收集到驱动程序中，或者保存到外部存储系统
- **结束SparkContext**：在所有的计算都完成后，我们需要停止SparkContext，释放资源
### Spark运行原理
- **背景**
  - 许多迭代式算法（比如机器学习、图算法等）和交互式数据挖掘工具，共同之处是，不同计算阶段之间会重用中间结果
  - 目前的MapReduce框架都是把中间结果写入到HDFS中，带来了大量的数据复制、磁盘IO和序列化开销
  - RDD提供了一个抽象的数据架构，我们不必担心底层数据的分布式特性，只需将具体的应用逻辑表达为一系列转换处理，不同RDD之间的转换操作形成依赖关系，可以实现管道化，避免中间数据存储
- **RDD操作**
  - **转换操作**：是在数据上应用的某种函数，它会返回一个新的RDD。常见的转换操作有map（将函数应用于RDD的每个元素），filter（选择满足条件的元素），union（合并两个RDD）等。转换操作的一个重要特点是惰性计算，也就是说，当你对RDD应用转换操作时，操作并不会立即执行，而是在遇到行动操作时才会执行。
  - **行动操作**：行动操作会触发计算，并返回结果到驱动程序或者将结果写入外部存储系统。常见的行动操作有count（返回RDD的元素数量），collect（返回RDD的所有元素），first（返回RDD的第一个元素），take（返回RDD的前n个元素）等。
  - **执行优化**：由于转换操作是惰性的，Spark有机会在实际计算开始之前对操作进行优化。例如，Spark可以将多个转换操作合并成一个操作，这被称为管道化。又例如，如果一个大的RDD经过转换后只剩下很少的数据，Spark可以在计算时只计算这些数据，而不是整个RDD，这被称为**谓词下推**。
  - 同时，由于行动操作的调用会触发实际的计算，所以Spark可以准确地知道数据的使用情况，比如哪些数据需要被缓存，哪些数据不再需要等，这有助于Spark更好地管理内存，提高计算效率。	
  - **优点：惰性调用、管道化、避免同步等待、不需要保存中间结果、每次操作变得简单**
- **Spark采用RDD实现高效计算的原因**
  - **高效的容错性**
    - 现有容错机制：数据复制或者记录日志
    - RDD：血缘关系（最后一个RDD经过“动作”操作进行转换，并输出到外部数据源这一系列处理）、重新计算丢失分区、无需回滚系统、重算过程在不同节点之间并行、只记录粗粒度的操作
  - 中间结果持久化到内存，数据在内存中的多个RDD操作之间进行传递，避免了不必要的读写磁盘开销
  - 存放的数据可以是Java对象，避免了不必要的对象序列化和反序列化
- **RDD之间的依赖关系（窄依赖和宽依赖）**
![示例](/images/图片/%E5%9B%BE%E7%89%8735.png)
![解释](/images/图片/%E5%9B%BE%E7%89%8736.png)
  - 根据DAG 图中的RDD 依赖关系，把一个作业分成多个阶段
    - 阶段划分的依据是窄依赖和宽依赖。
    - 对于宽依赖和窄依赖而言，窄依赖可以实现“流水线”优化，宽依赖无法实现“流水线”优化
  - 逻辑上，每个RDD 操作都是一个**fork/join**（一种用于并行执行任务的框架），把计算fork 到每个RDD 分区，完成计算后对各个分区得到的结果进行join 操作
![示例](/images/图片/%E5%9B%BE%E7%89%8737.png)
  - 被分成三个Stage，在Stage2中，从map到union都是窄依赖，这两步操作可以形成一个流水线操作
  - 分区7通过map操作生成的分区9，可以不用等待分区8到分区10这个map操作的计算结束，而是继续进行union操作，得到分区13，这样流水线执行大大提高了计算的效率
- **RDD运行过程**
  - 创建RDD对象
  - SparkContext负责计算RDD之间的依赖关系，构建DAG
  - DAGScheduler负责把DAG图分解成多个Stage，每个Stage中包含了多个Task，每个Task会被TaskScheduler分发给各个WorkerNode上的Executor去执行
### Spark SQL
- 前身是Shark（Hive on Spark），将物理执行计划从MapReduce作业替换成了Spark作业，通过Hive的HiveQL解析，把HiveQL翻译成Spark上的RDD操作。
![hive](/images/图片/%E5%9B%BE%E7%89%8727.png)
  - 导致问题：
    - 一是执行计划优化完全依赖于Hive，不方便添加新的优化策略；
    - 二是因为Spark是线程级并行，而MapReduce是进程级并行，因此，Spark在兼容Hive的实现上存在线程安全问题，导致Shark不得不使用另外一套独立维护的打了补丁的Hive源码分支
- Spark SQL在Hive兼容层面仅依赖HiveQL解析、Hive元数据。
- 从HQL被解析成抽象语法树（AST）起，全部由Spark SQL接管。Spark SQL执行计划生成和优化都由**Catalyst**（函数式关系查询优化框架）负责
- Spark SQL增加了SchemaRDD（即带有Schema信息的RDD），使用户可以在Spark SQL中执行SQL语句，数据既可以来自RDD，也可以是Hive、HDFS、Cassandra等外部数据源，还可以是JSON格式的数据
### 编程实践
- **读数据**
  Scala > val textFile = sc.textFile("file:///usr/local/spark/README.md") 
                    // 通过file:前缀指定读取本地文件
- **转换API**
  ![示例](/images/图片/%E5%9B%BE%E7%89%8738.png)
- **行动API**
  ![示例](/images/图片/%E5%9B%BE%E7%89%8739.png)







































  




