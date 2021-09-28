# Flink简介

## 1. 背景

Apache Flink 是一个框架和分布式**处理引擎**，用于在*无边界和有边界*数据流上进行**有状态**的计算。以**内存**速度和任意规模进行计算

### 1.1  文档

- 官方文档：[Apache Flink: Apache Flink 是什么？](https://flink.apache.org/zh/flink-architecture.html)
- 中文论坛：[Flink 中文社区 | 学习资料 (flink-learning.org.cn)](https://flink-learning.org.cn/article)

## 2. 特性 & 场景

### 2.1 特性

- **擅长处理无界和有界数据集**
  - **支持高吞吐、低延迟、高性能的流处理**
  - **支持程序自动优化**：避免特定情况下 Shuffle、排序等昂贵操作，中间结果进行缓存
  - **支持具有 Backpressure 功能的持续流模型**：防止生成数据的速度比下游算子消费的的速度快引发的消息拥堵问题
- **精确的状态控制**：
  - **支持多种状态类型**：支持基础类型、list、map等。
  - **插件化的State Backend**：支持多种状态存储的后端，可存储到内存或数据库中。
  - **精确一次语义(Exactly-once)**： checkpoint 和故障恢复算法保证了故障发生后应用状态的一致性，发生故障时对程序透明。发生故障时，对应用程序透明，不造成正确性的影响
  - **超大数据量状态**： 能够利用其异步以及增量式的 checkpoint 算法，存储数 TB 级别的应用状态
  - **可弹性伸缩的应用**：支持有状态应用的分布式的横向伸缩
- **提供丰富的时间语义支持**
  - **事件时间模式**：根据事件本身自带的时间戳进行结果的计算，事件时间模式的处理总能保证结果的准确性和一致性
  - **Watermark 支持**： 引入了 watermark 的概念，用以衡量事件时间进展。Watermark 也是一种平衡处理延时和完整性的灵活机制。
  - **迟到数据处理**：当以带有 watermark 的事件时间模式处理数据流时，在计算完成之后仍会有相关数据到达。这样的事件被称为迟到事件。
  - **处理时间模式**：根据处理引擎的机器时钟触发计算，一般适用于有着严格的低延迟需求，并且能够容忍近似结果的流处理应用
- **部署应用到任意地方**：
  - **集成多种集群管理服务**：支持 yarn、k8s、Mesos等
  - **便利升级应用服务版本**：在新版本更新升级时，可以根据上一个版本程序记录的 Savepoint 内的服务状态信息来重启服务
  - **方便的集群迁移**：通过使用 Savepoint，流服务应用可以自由的在不同集群中迁移部署
- **良好的监控和控制服务**：
  - **Web UI方式支撑运维**：提供了一个web UI来观察、监视和调试正在运行的应用服务
  - **指标服务**：提供了一个复杂的度量系统来收集和报告系统和用户定义的度量指标信息。度量信息可以导出到多个报表组件服务
  - **标准的WEB REST API接口服务**：提供多种REST API接口，有提交新应用程序、获取正在运行的应用程序的Savepoint服务信息、取消应用服务等接口

### 2.2 应用场景

主要聚焦七大应用场景：

1. 底层平台建设

2. 实时数仓

3. 实时推荐 
4. 实时分析 
5. 实时大屏
6. 风控 
7. 数据湖

部分互联网公司Flink应用: 

- 阿里：
  - 双十一运营数据实时计算，40 亿条 / 秒数据。

- 字节跳动: 

  - 基于 Flink 的 MQ-Hive 实时数据集成 (Exactly Once，无中间环节实时性高，无中间存储)
- 腾讯：

  - 王者荣耀背后的实时大数据平台（基于事件时间水印的监控，减少计算量，提高准确性）
  - 腾讯基于flink实时数仓及多维数据分析
- 快手：

  - 统计监控
  - 数据ETL
  - 实时业务处理。
- 滴滴：
- 实时监控，业务指标、导航等
  - 实时同步，轨迹数据同步
  - 实时特征，乘客/上下车特征
- 网易：
  - 网易游戏基于Flink的流行ETL建设
  - 网易云音乐基于Flink+Kafka实时数仓
  - 网易基于Flink+Iceberg构建数据湖

## 3. 架构 & 核心概念

![flink架构图](https://nightlies.apache.org/flink/flink-docs-release-1.13/fig/processes.svg)

Client不是运行时的一部分，而是发送任务指令的。命令行中为./bin/flink run ...

### 3.1 核心组件

#### 3.1.1 JobManager

>1.  主要工作：

- 调度下一个task.
- 对完成或失败的task进行反应。
- 协调checkpoint。
- 协调从失败中恢复。

>2.  主要组成部分：

- **ResourceManager**：
  - 资源提供、回收、分配，管理 task slots。
  - 有YARN、Mesos、Kubernetes 和 standalone 部署多种实现。
  - standalone 只能分配可用的，不能启动新的。

- **Dispatcher**： 提供REST接口。提供JobMaster，提供WebUI。

- **JobMaster**：管理单个[JobGraph](https://nightlies.apache.org/flink/flink-docs-release-1.13/zh/docs/concepts/glossary/#logical-graph)的执行。

#### 3.1.2 TaskManager

> 1. 主要工作：

- 执行作业流
- 缓存和交换数据流

> 调度的最小单位是task slot，表示并发处理的task数量。对应的是线程数。

### 3.2 集群类型

|                  | Session 集群                                                 | Job 集群                           | Application 集群                    |
| ---------------- | ------------------------------------------------------------ | ---------------------------------- | ----------------------------------- |
| **生命周期**     | 预设的、长期运行的、不受作业影响                             | 提交作业时启动，完成即销毁         | 仅从 Flink 应用程序main方法执行作业 |
| **资源隔离**     | 不隔离，存在资源竞争<br>TaskManager奔溃任务全失败            | 资源隔离，仅影响单作业             | 作用于单应用程序资源隔离            |
| **其他注意事项** | 1.可以节省资源申请和启动时间<br> 2. 适合执行时间短的交互式分析。 | 适合长时间运行且启动时间不敏感任务 | 可看做是客户端运行方案              |

### 3.3 有状态流处理

> 状态定义：有些操作会记住多个事件的信息。这些操作称为有状态的。常见例子如下：

- 搜索某些事件模式时，状态将存储到目前为止遇到的事件序列。
- 当聚合每分钟/小时/天的事件时，状态持有待处理的聚合。
- 在数据点流上训练机器学习模型时，状态保存模型参数的当前版本。
- 当需要管理历史数据时，状态允许有效访问过去发生的事件。

> Keyed State:  按照key组织记录key的状态



## 4. 开发使用

![api](https://nightlies.apache.org/flink/flink-docs-release-1.13/fig/levels_of_abstraction.svg)

### 4.1 API分层

flink提供不同级别的抽象接口，详情如下：

1. 最底层为**有状态实时流处理**，实现**Process Function** ，被继承到DataStream API ，用户可以在此层抽象中**注册事件时间**和**处理时间**，可实现复杂计算。
2. 第二层是**Core APIs**，包含 **DataStream API**和 **DataSet AP**I，提供通用模块组件。如用户自定义转换（transformations）、联接（joins）、聚合（aggregations）、窗口（windows）和状态（state）操作。
3. 第三层抽象**Table API**。以表为中心的声明式编程（DSL），提供select、project、join、group-by 和 aggregate 等。
4. 最顶层抽象是SQL。似于 Table API，基于SQL表达式。

### 4.2 基于DataStream/DateSet API

开发核心步骤：

> 1. 获取执行环境。
> 2. 加载或创建初始数据（创建source）
> 3. 指定数据转换 (编写map)
> 4. 指定计算结果存储位置 (创建sink)
> 5. 触发程序运行。

#### 4.2.1 maven依赖

``` xml

<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java_2.11</artifactId>
  <version>1.13.2</version>
  <scope>provided</scope>
</dependency>

<!--   执行环境依赖     -->
<dependency>
   <groupId>org.apache.flink</groupId>
   <artifactId>flink-clients_2.11</artifactId>
   <version>${flink.version}</version>
 </dependency>
```

#### 4.2.2 应用开发

##### 4.2.2.1 获取执行环境

  ``` java
getExecutionEnvironment()
createLocalEnvironment()
createRemoteEnvironment(String host, int port, String... jarFiles)
  ```

##### 4.2.2.2 创建source

- 第一种：ENV提供了如fromCollection，fromElements，generateSequence，createInput，readFile，socketTextStream等

  ```java
DataStream<String> text = env.readTextFile("file:///path/to/file");
socketTextStream(String hostname, int port)
  ```

- 第二种：Connectors，需要maven额外依赖

  ```java
  DataStream<String> stream = env.addSource(new FlinkKafkaConsumer<>("topic", new SimpleStringSchema(), properties));
  ```

- 第三种：用户自定义，继承**RichSourceFunction** ，实现以下方法：

  ``` java
public void open(Configuration parameters) throws Exception; 
public void close() throws Exception;
public void run(SourceContext<Student> ctx);
  ```

##### 4.2.2.3 编写转换运算（核心）

- 第一种：使用内置的Map ,FlatMap ,Filter ,KeyBy ,Reduce ,Window ,Union ,Window Join,Connect ,Iterate 等。

- 第二中：继承**RichFlatMapFunction**，实现方法

  ``` java
  public void open(Configuration parameters)
  public void flatMap(IN value, Collector<OUT> out) throws Exception
  ```

##### 4.2.2.4 编写sink

- 第一种：内置的如print，writeAsText，writeAsCsv，writeToSocket。

- 第二种：Connectors，需要maven额外依赖

  ``` java
  FlinkKafkaProducer<String> myProducer = new FlinkKafkaProducer<>(
          "my-topic",                  // target topic
          new SimpleStringSchema(),    // serialization schema
          properties,                  // producer config
          FlinkKafkaProducer.Semantic.EXACTLY_ONCE); // fault-tolerance
  
  stream.addSink(myProducer);
  ```

- 第三种：继承**RichSinkFunction**，实现以下方法

  ```java
  public void open(Configuration parameters) throws Exception；
  void invoke(IN value, Context context)
  ```

##### 4.2.2.5 触发执行

```java
env.execute();
```

### 4.3 基于Table API/SQL

> able API 和 SQL 集成在同一套 API 中, 核心概念是**Table**。

#### 4.3.1 Maven依赖

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-streaming-java_2.11</artifactId>
  <version>1.13.2</version>
  <scope>provided</scope>
</dependency>
<!--   table依赖     -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-java-bridge_2.11</artifactId>
    <version>${flink.version}</version>
</dependency>
<!--   table代码实现是采用了scala     -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-streaming-scala_2.11</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
</dependency>

<!-- (old Planner) 二选一 -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table-planner_2.11</artifactId>
  <version>1.13.2</version>
  <scope>provided</scope>
</dependency>
<!-- (Blink planner) 二选一 -->
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner-blink_2.11</artifactId>
    <version>${flink.version}</version>
    <scope>provided</scope>
</dependency>

<!--   执行环境依赖     -->
<dependency>
   <groupId>org.apache.flink</groupId>
   <artifactId>flink-clients_2.11</artifactId>
   <version>${flink.version}</version>
 </dependency>
```

依赖参考 [概览 | Apache Flink](https://nightlies.apache.org/flink/flink-docs-release-1.13/zh/docs/dev/table/overview/) :

> - flink-table-common: 公共模块，比如自定义函数、格式等需要依赖的。
> - flink-table-api-java: Table 和 SQL API，使用 Java 语言编写的，给纯 table 程序使用（还在早期开发阶段，不建议使用）
> - flink-table-api-scala: Table 和 SQL API，使用 Scala 语言编写的，给纯 table 程序使用（还在早期开发阶段，不建议使用）
> - flink-table-api-java-bridge: Table 和 SQL API 结合 DataStream/DataSet API 一起使用，给 Java 语言使用。
> - flink-table-api-scala-bridge: Table 和 SQL API 结合 DataStream/DataSet API 一起使用，给 Scala 语言使用。
> - flink-table-planner: table Planner 和运行时。这是在1.9之前 Flink 的唯一的 Planner，但是从1.11版本开始我们不推荐继续使用。
> - flink-table-planner-blink: 新的 Blink Planner，从1.11版本开始成为默认的 Planner。
> - flink-table-runtime-blink: 新的 Blink 运行时。
> - flink-table-uber: 把上述模块以及 Old Planner 打包到一起，可以在大部分 Table & SQL API 场景下使用。打包到一起的 jar 文件 flink-table-\*.jar 默认会直接放到 Flink 发行版的 /lib 目录下。
> - flink-table-uber-blink: 把上述模块以及 Blink Planner 打包到一起，可以在大部分 Table & SQL API 场景下使用。打包到一起的 jar 文件 flink-table-blink-*.jar 默认会放到 Flink 发行版的 /lib 目录下。

#### 4.3.2 应用开发

##### 4.3.2.1 计划器

| 差异                        | Old planner           | Blink                        |
| --------------------------- | --------------------- | ---------------------------- |
| 作业转换                    | 支持                  | 不支持，统一做DataStream处理 |
| BatchTableSource            | 支持BatchTableSource  | 以StreamTableSource替代      |
| FilterableTableSource不兼容 | PlannerExpression下推 | Expression下推               |
| 字符串的键值配置            | 不支持                | 支持                         |
| CalciteConfig实现不同       | -                     | -                            |
| DAG图生成                   | 每个sink都生成新的DAG | 多sink优化成一张DAG          |
| Catalog统计                 | 不支持                | 支持数据统计                 |

##### 4.3.2.2 环境创建

> **TableEnvironment** 是核心概念，主要负责

- 注册外部的 catalog
- 在内部的 catalog 中注册 Table
- 加载可插拔模块
- 执行 SQL
- 注册自定义函数
- 将 DataStream 或 DataSet 转换成 Table
- 持有对 ExecutionEnvironment 或 StreamExecutionEnvironment 的引用

``` java
// **********************
// FLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

EnvironmentSettings fsSettings = EnvironmentSettings.newInstance().useOldPlanner().inStreamingMode().build();
StreamExecutionEnvironment fsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment fsTableEnv = StreamTableEnvironment.create(fsEnv, fsSettings);
// or TableEnvironment fsTableEnv = TableEnvironment.create(fsSettings);

// ******************
// FLINK BATCH QUERY
// ******************
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.table.api.bridge.java.BatchTableEnvironment;

ExecutionEnvironment fbEnv = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment fbTableEnv = BatchTableEnvironment.create(fbEnv);

// **********************
// BLINK STREAMING QUERY
// **********************
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

StreamExecutionEnvironment bsEnv = StreamExecutionEnvironment.getExecutionEnvironment();
EnvironmentSettings bsSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
StreamTableEnvironment bsTableEnv = StreamTableEnvironment.create(bsEnv, bsSettings);
// or TableEnvironment bsTableEnv = TableEnvironment.create(bsSettings);

// ******************
// BLINK BATCH QUERY
// ******************
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.TableEnvironment;

EnvironmentSettings bbSettings = EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build();
TableEnvironment bbTableEnv = TableEnvironment.create(bbSettings);
```

##### 4.3.2.3 表创建

> 表同时包含了（Table和View），并且有区分**临时表**（Temporary Table）和**永久表**（Permanent Table）
>
> - 永久表需要 catalog，如Hive Metastore
> - 临时表仅会话有效
> - 标识相同，临时表会覆盖永久表

```java
Table projTable = tableEnv.executeSql("CREATE TEMPORARY TABLE table1 ... WITH ( 'connector' = ... )");
//创建视图
tableEnv.createTemporaryView("projectedTable", projTable);
```

##### 4.3.2.4 表查询

Table API 方式

```java
Table orders = tableEnv.from("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter($("cCountry").isEqual("FRANCE"))
  .groupBy($("cID"), $("cName"))
  .select($("cID"), $("cName"), $("revenue").sum().as("revSum"));
```

SQL方式

```java
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );
```

##### 4.3.2.5 输出表

```
final Schema schema = new Schema()
    .field("a", DataTypes.INT())
    .field("b", DataTypes.STRING())
    .field("c", DataTypes.BIGINT());

tableEnv.connect(new FileSystem().path("/path/to/file"))
    .withFormat(new Csv().fieldDelimiter('|').deriveSchema())
    .withSchema(schema)
    .createTemporaryTable("CsvSinkTable");
    result.executeInsert("CsvSinkTable");
```

## 5. 集群部署

解压软件包

```shell
tar xzf flink-*.tgz
cd flink-*
```

Standalone模式：

``` shell
bin/start-cluster.sh
```

YARN  Session 模式：

``` shell
export HADOOP_CLASSPATH=`hadoop classpath`
./bin/yarn-session.sh --detached
./bin/flink run ./examples/streaming/TopSpeedWindowing.jar
```

YARN Pre-Job模式：

``` shell
export HADOOP_CLASSPATH=`hadoop classpath`
./bin/flink run -t yarn-per-job --detached ./examples/streaming/TopSpeedWindowing.jar
```

启动shell:

```
bin/sql-client.sh embedded shell
```

## 6. 竞品比较

待补充

## 7. 常见问题

待补充

## 8.附录

### 8.1 核心词汇

- **Event** ：流或批处理应用程序的 input 和 output
- **ExecutionGraph/Physical Graph** ： 把逻辑图转换为可执行的结果。

  - 节点：task
  - 边：数据流的输入/输出关系或 partition
- **JobGraph/Logical Graph**：逻辑有向图

  - 节点：算子
  - 边：数据流的输入/输出关系
- **Instance** ：运行时的特定类型(通常是 Operator 或者 Function)
- **Record** ：数据集或数据流的组成元素
- **Partition**：分区是数据流的独立子集。

## REF

1. https://developer.aliyun.com/article/783020

