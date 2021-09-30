# Hadoop 体系概览

## 1. 背景

 大数据处理核心模型的三驾马车分别为：

 1. 分布式文件系统 ： 解决海量数据存储问题
 2. MapReduce：解决数据编程模型和计算框架问题
 3. BigTable：以Table形式对外提供数据应用的能力。

Hadoop与之对应的：

1. 分布式文件系统 —— HDFS
2. MapReduce—— Hadoop MapReduce，TEZ，Spark，Storm
3. BigTable —— HBase

本文主要针对核心的HDFS，HIVE，HBase，YARN进行概览描述

## 2. HDFS

### 2.1 HDFS概览

> Hadoop 分布式文件系统 (HDFS) 是一种分布式文件系统。 HDFS 最初是作为 Apache Nutch 网络搜索引擎项目的基础设施而构建的。 HDFS 是 Apache Hadoop Core 项目的一部分。项目 URL 是 http://hadoop.apache.org/。
>
> HDFS 具有高度容错性，旨在部署在低成本硬件上。 
>
> - 提供对应用程序数据的高吞吐量访问，适用于具有大型数据集的应用程序。 
> -  放宽了一些 POSIX 要求，以启用对文件系统数据的流式访问。 

**设计目标及假设**：

- 硬件容错

  > 硬件故障是常态而不是例外。因此，故障检测和快速、自动恢复是 HDFS 的核心架构目标。

- 流式数据访问

  > 设计着眼于批处理而不是用户交互使用。
  >
  > 重点是数据访问的高吞吐量而不是数据访问的低延迟。 
  >
  > 一些关键领域的 POSIX 语义已被妥协以提高数据吞吐率。

- 超大数据集

  > 典型文件大小为GB到TB

- 简单的数据一致性模型

  > - 一次写入多次读取的访问模型。
  >
  > - 文件一旦创建、写入和关闭，除了追加和截断外，不需要更改。不能在任意点更新。
  >
  > - 这种假设简化了数据一致性问题并实现了高吞吐量数据访问。 

- 计算移动成本高于数据移动成本

- 跨异构硬件和软件平台的可移植性

**NameNode and DataNodes**

HDFS是一个主从架构。集群由一个NameNode 多个DataNode组成，NameNode 执行文件系统命名空间操作，如打开、关闭和重命名文件和目录。它还确定块到 DataNode 的映射。同时响应客户端请求。

![HDFS架构图](https://hadoop.apache.org/docs/r3.3.1/hadoop-project-dist/hadoop-hdfs/images/hdfsarchitecture.png)

**文件系统命名空间**

HDFS支持传统的分层文件组织，但不支持软连接等。 HDFS 遵循文件系统的命名约定，但保留了一些路径（例如 /.reserved 和 .snapshot ）

**数据复制**

- HDFS 旨在可靠地跨大型集群中的机器存储非常大的文件。它将每个文件存储为一个块序列。复制文件的块以实现容错。每个文件的块大小和复制因子是可配置的。

- 文件中除最后一个块外的所有块大小都相同。

- 应用程序可以指定文件的副本数。复制因子可以在文件创建时指定，以后可以更改。
-  HDFS 中的文件是一次性写入的（除了追加和截断），并且在任何时候都严格单写。
- NameNode 控制块复制。它会定期从集群中的每个 DataNode 接收 Heartbeat 和 Blockreport。收到心跳意味着 DataNode 运行正常。 Blockreport 包含 DataNode 上所有块的列表。

![块复制](https://hadoop.apache.org/docs/r3.3.1/hadoop-project-dist/hadoop-hdfs/images/hdfsdatanodes.png)

数据复制步骤：

- 副本放置

  > - 副本的放置对于 HDFS 的可靠性和性能至关重要。机架感知副本放置策略的目的是提高数据可靠性、可用性和网络带宽利用率。
  > - NameNode 通过 Hadoop 机架感知中概述的过程确定每个 DataNode 所属的机架 ID.
  > - 由于 NameNode 不允许 DataNode 拥有同一块的多个副本，因此创建的最大副本数是当时 DataNode 的总数。

- 副本选择

  > 为了减小带宽消耗和读取延迟，HDFS 尝试满足来自最接近读取器的副本的读取请求。如果在与读取器节点相同的机架上存在副本，则首选该副本来满足读取请求。如果 HDFS 集群跨越多个数据中心，那么驻留在本地数据中心的副本优先于任何远程副本

- 块放置策略

  > - 常见情况，复制因子为3时，HDFS的放置策略是如果写入者在数据节点上，则将一个副本放在本地机器上，否则在与写入者相同机架的随机数据节点上，另一个副本放在不同（远程）机架中的一个节点，以及同一远程机架中不同节点上的最后一个节点。
  >
  > - HDFS 还支持 4 种不同的可插入块放置策略。用户可以根据他们的基础设施和用例选择策略。默认情况下，HDFS 支持 BlockPlacementPolicyDefault。

- 安全模式

  > 启动时，NameNode 进入称为安全模式的特殊状态。 NameNode 处于安全模式状态时，不会发生数据块的复制

**文件系统元数据的持久化**

> - EditLog 的事务日志来持久记录文件系统元数据发生的每个更改。
> - 整个文件系统命名空间，包括块到文件和文件系统属性的映射，都存储在一个名为 FsImage 的文件中。 FsImage 也作为文件存储在 NameNode 的本地文件系统中。
> - NameNode 将整个文件系统命名空间和文件 Blockmap 的映像保存在内存中。NameNode 启动时，或由可配置的阈值触发检查点时，它从磁盘读取 FsImage 和 EditLog，将 EditLog 中的所有事务应用于 FsImage 的内存表示。
> - DataNode 数据块存储在其本地文件系统中的一个单独文件中。 启动时，它会扫描其本地文件系统，生成Blockreport给NameNode；

**通讯协议**

> 通信基于tcp/ip。
>
> - 客户端使用ClientProtocol 与 NameNode 进行对话
> - DataNode 使用 DataNode 协议与 NameNode 通信
> - NameNode不发起RPC，只响应请求。

**健壮性**

- 数据磁盘故障、心跳和重新复制

  > - 每个 DataNode 都会定期向 NameNode 发送 Heartbeat 消息确认存活情况。
  > - DataNode 死亡可能会导致某些块的复制因子低于其指定值。必要时启动复制。
  >
  > - 标记为死的超时时间保守地很长（默认超过 10 分钟），避免复制风暴。

- 集群重平衡

  > 如果 DataNode 上的可用空间低于某个阈值，则方案可能会自动将数据从一个 DataNode 移动到另一个 DataNode。

- 数据完整性

  > 客户端创建 HDFS 文件时，它会计算文件的每个块的校验和，并将这些校验和存储在同一 HDFS 命名空间中的单独隐藏文件中。检索时比对，不匹配选择其他副本的DataNode。

- 元数据磁盘故障

  > 可以将 NameNode 配置为支持维护 FsImage 和 EditLog 的多个副本，每次更新都导致副本同步更新。秒级事务速率。

- 快照：

  > 特定时间创建数据副本。允许损坏的HDFS回滚到快照点。

**数据组织**

- 数据块：HDFS 使用的典型块大小为 128 MB。
- 复制流水线：客户端写入第一个 DataNode。第一个 DataNode 开始分部分接收数据，将每个部分写入其本地存储库并将该部分传输到列表中的第二个 DataNode。第二个部分写入后传输到第三个。

**使用方式**

- FS Shell
- DFSAdmin
- Browser Interface

**空间开辟**

- 文件删除与取消删除

  > 如果启用垃圾配置，删除文件被移入/user/<username>/.Trash 。超过检查点周期后会被删除。

- 降低复制因子

  > 复制因子减少，NameNode选择删除副本，下一个 Heartbeat 将此信息传输到 DataNode。然后DataNode删除相应的块。

### 2.2  文档资料

**官网资料**：https://hadoop.apache.org/docs/r3.3.1/



## 3.YARN

### 3.1 概览

> YARN是Hadoop下的一个**资源管理器**。

![Yarn架构](https://hadoop.apache.org/docs/r3.3.1/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif)

1. 基本思想是将**资源管理**和**作业调度/监控**的功能拆分为单独的守护进程。

- 一个全局的 ResourceManager (*RM*) 
- 每个应用的per-application ApplicationMaster (*AM*)。
- 应用程序是单个作业或作业的 DAG

2. 数据计算框架由ResourceManager 和 NodeManager 构成

> **ResourceManager** ：在系统中的所有应用程序之间仲裁资源
>
> **NodeManager** ：每台机器的框架代理。负责容器、监控其资源使用情况（cpu、内存、磁盘、网络）并将其报告给 ResourceManager/Schedule

3. 

## 4. Hive

### 4.1 概览

> Apache Hive ™是使用 SQL 读取、写入和管理驻留在分布式存储中的大型数据集的**数据仓库软件**。提供了命令行工具和 JDBC 驱动程序来将用户连接到 Hive。

**提供特性**：

- 通过 SQL 轻松访问数据的工具

  > 支持数据仓库任务，例如提取/转换/加载 (ETL)、报告和数据分析
  >
  > - 支持标准SQL。
  > - SQL 还可以通过用户定义函数 (UDF)、用户定义聚合 (UDAF) 和用户定义表函数 (UDTF) 使用用户代码进行扩展

- 一种对各种数据格式强加结构的机制

  > 不是必须存储数据的单一“Hive 格式”。 Hive还支持：
  >
  > - 带有用于逗号和制表符分隔值 (CSV/TSV) 文本文件
  > - Apache Parquet™
  > - Apache ORC™ 和其他格式的内置连接器。
  > - 用户可以使用其他格式的连接器扩展 Hive

- 访问直接存储在HDFS 或其他数据存储系统（例如  HBase）中的文件

- 通过 Apache Tez、Apache Spark 或 MapReduce 执行查询

- 带有 HPL-SQL 的过程语言

- 通过 Hive LLAP、Apache YARN 和 Apache Slider 进行亚秒级查询检索

- 不是为在线事务处理 (OLTP) 设计，适合传统数据仓库。

### 4.2 HCatalog

#### 4.2.1 概述

HCatalog 是 Hadoop 的表和存储管理层，让用户能够通过不同数据处理工具的松地在网格上读写数据。为用户提供HDFS数据的关系视图，用户无需担心数据存储格式。

HCatalog 支持读取和写入可以写入 SerDe（串行器-解串器）的任何格式的文件。

- 默认支持RCFile、CSV、JSON 和 SequenceFile 以及 ORC 文件格式。
- 自定义格式，则实现 InputFormat、OutputFormat 和 SerDe。

![hive示意图](https://cwiki.apache.org/confluence/download/attachments/34013260/hcat-product.jpg?version=1&modificationDate=1375694292000&api=v2)

#### 4.2.2 架构

HCatalog 构建在 Hive **Metastore** 之上，并结合了 Hive 的 DDL。 HCatalog 为Pig或Hive命令行工具提供data definition and metadata exploration接口。

**接口**：

- Pig 的 HCatalog 接口由 HCatLoader 和 HCatStorer 组成，它们分别实现了 Pig 加载和存储接口。

  > **HCatStorer**： 接受要写入的表和可选的分区键规范以创建新分区，是HCatOutputFormat的实现。
  >
  > **HCatLoader**：是HCatInputFormat 的实现。

- MapReduce 的 HCatalog 接口由HCatInputFormat 和 HCatOutputFormat组成

  >  是 Hadoop InputFormat 和 OutputFormat 的实现。

- 数据使用命令行（CLI）进行定义。

  > 允许用户创建、更改、删除表等。 CLI 还支持如SHOW TABLES, DESCRIBE TABLE等。
  >
  > 不支持DDL的导入导出。

**数据模型**：

- HCatalog 提供数据的关系视图

  > 表可以放在数据库中。表也可以在一个或多个键上进行散列分区

- 分区包含记录。一旦创建了分区，就无法向其中添加、删除或更新记录。

### 4.3 WebHCat

  提供可用于运行MapReduce（或 YARN）、Hive 作业的服务。您还可以使用 HTTP（RestFul）接口执行 Hive 元数据操作。

![hive请求](https://cwiki.apache.org/confluence/download/attachments/34015492/WebHCatArch.jpg?version=1&modificationDate=1376805339000&api=v2)

### 4.2 文档资料

**官网资料**：https://cwiki.apache.org/confluence/display/HIVE

**命令行工具**：[LanguageManual Cli - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Cli)

## 5. HBASE

### 5.1 概览

> Apache HBase™ 是 一种分布式、可扩展的大数据存储的Hadoop 数据库。提供对大数据进行随机、实时读/写访问时

**特性**：

- 线性和模块化可扩展性。
- 严格一致的读取和写入。
- 自动和可配置的表分片
- 支持RegionServers 之间的自动故障转移。
- 使用 Apache HBase 表支持 Hadoop MapReduce 作业的便捷基类。
- 易于使用 Java API 进行客户端访问。
- 采用块缓存和布隆过滤器进行实时查询。
- 通过服务器端过滤器向下推查询谓词
- Thrift 网关以及支持 XML、Protobuf 和二进制数据编码选项的 REST-ful Web 服务
- 可扩展的基于 jruby (JIRB) shell
- 支持通过 Hadoop 指标子系统将指标导出到文件或 Ganglia；或通过 JMX



### 5.2 文档资料

**官网资料**：https://hbase.apache.org/book.html

**中文参考**：http://abloz.com/hbase/book.html

**Quickstart**: https://hbase.apache.org/book.html#quickstart

