## 当前痛点

1. 无法做自动化运维/部署,只支持把代码copy到simba网页集成的编辑器中,这是一个非常大的硬伤,假如我们有dev qa prod环境,那dev环境每天都会是开发在网页上复制粘贴,大量的手动操作使得整个开发流程可靠性低,复杂度高
1. 网页编辑器体验极差,毫无工程化体验

- 不支持断点.对于某些开发资源可能只有线上能访问的,可能就需要在simba网页的编辑器里调试,然而它却连断点都不支持,开发过程痛苦无比
- 不支持文件相互引用. 这样会导致如果很多模块有相同的功能(函数),那就得每个文件里都写上重复的代码,而且对于比较复杂的功能,一个文件的代码量可能极多,后期维护困难
- 不支持传参或者环境变量. 对于某些敏感信息,例如数据库账号密码,目前在上simba上只能明文写在代码中

3. 文档几乎等于0,所有功能基本就只能靠摸索,上手成本较高,这个问题在多团队协作时候尤为明显
4. 代码执行出错之后,页面上看到的错误信息及其有限,很难判断真正的错误原因,导致排错困难
5. simba上的python代码居然会把字符串中的#解析成注释导致代码执行失败,也侧面反应了系统的可靠性不高
6. 目前simba上的spark/flink全处于不可用状态,这对于大数据处理来说也是不可接受的
7. 处理效率较低,通常一个简单的sql执行也需要花费10s以上才能看到结果,在有大量数据需要处理时候,由于不支持spark,所以很难直接进行优化
8. 当前simba中的数据复用度低,也没有数据开发的规范,数据和代码都比较凌乱,有大量的冗余数据和冗余处理
9. 没有数据质量监控.全链路的数据处理都是人工后置校验数据正确性,没有自动化数据质量校验
10. 数据直接存储在ECS磁盘上,没有备份机制,同时可迁移性也较差



基于当前痛点,我们结合当前的主流解决方案以及它们在阿里云上的实际情况做出新的技术选型:



## 目前主流的数据湖存储方案

- Databricks Delta Lake

<img src="https://camo.githubusercontent.com/5535944a613e60c9be4d3a96e3d9bd34e5aba5cddc1aa6c6153123a958698289/68747470733a2f2f646f63732e64656c74612e696f2f6c61746573742f5f7374617469632f64656c74612d6c616b652d77686974652e706e67" alt="Delta Lake Logo" style="zoom: 25%;" />

Delta Lake是Spark商业公司Databricks的开源项目,是Databricks公司壮大spark生态的重要一环,深度集成spark,可以非常方便的享受到spark生态带来的便利.

官网简介: Delta Lake is an open-source storage framework that enables building a
Lakehouse architecture with compute engines including Spark, PrestoDB, Flink, Trino, and Hive and APIs for Scala, Java, Rust, Ruby, and Python.

![image-20230112154210961](/Users/can.wang/Library/Application Support/typora-user-images/image-20230112154210961.png)

- Apache Hudi

<img src="https://camo.githubusercontent.com/167d7939912a5b6487ee0c6c38374215ebd740386d5e61b10356b065243b61a4/68747470733a2f2f687564692e6170616368652e6f72672f6173736574732f696d616765732f687564692d6c6f676f2d6d656469756d2e706e67" alt="Hudi logo" style="zoom:50%;" />

Apache Hudi(Hadoop Upsert And Incremental),是Uber内部孵化出的开源项目,设计之初的主要目的是解决传统数仓无法快速的upsert数据的痛点,开源之后朝着「下一代的流式数据湖平台」的方向发展.

官网简介:Apache Hudi is a transactional data lake platform that brings database and data warehouse capabilities to the data lake. Hudi reimagines slow old-school batch data processing with a powerful new incremental processing framework for low latency minute-level analytics.

其架构图为:![Hudi Data Lake](https://hudi.apache.org/assets/images/hudi-lake-overview-e39f80337517a0a1999d8eb5cd0ac965.png)



- Apache Iceberg

<img src="https://camo.githubusercontent.com/d120d4367f4fcaea0086ec2533ecad35c4ce2fadc313071ee2c26ff319833168/68747470733a2f2f696365626572672e6170616368652e6f72672f646f63732f6c61746573742f696d672f496365626572672d6c6f676f2e706e67" alt="Iceberg" style="zoom:50%;" />

Apache Iceberg由Netflix内部孵化的开源项目,最初为了解决Hive的诸多缺陷而自研,后发展成了一个通用的数据湖项目,有着高度抽象的设计.

官网简介为:Iceberg is a high-performance format for huge analytic tables. Iceberg brings the reliability and simplicity of SQL tables to big data, while making it possible for engines like Spark, Trino, Flink, Presto, Hive and Impala to safely work with the same tables, at the same time.

其架构图为:

![Comp-1](/Users/can.wang/Downloads/Comp-1.gif)



## 详细对比

### 1. ACID和隔离级别

|         | ACID | 隔离级别                                            | 并发写入 | 时间旅行 |
| ------- | ---- | --------------------------------------------------- | -------- | -------- |
| Iceberg | 是   | serializable/snapshot isolation                     | 是       | 是       |
| Hudi    | 是   | snapshot isolation                                  | 是       | 是       |
| Delta   | 是   | serializable/snapshot isolation/write serialization | 是       | 是       |

serializable: 所有的 reader 和 writer 都必须串行执行

snapshot isolation: 如果多个 writer 写的数据无交集,则可以并发执行,否则只能串行.reader 和 writer 可以同时执行

write serialization: 多个 writer 必须严格串行,reader 和 writer 之间则可以同时执行

三个框架都很优秀,全支持并发性最好的snapshot isolation,也都支持ACID/并发写入和时间旅行

### 2. schema变更和设计

|         | 增加列 | 删除列         | 重命名列       | 更新列 | 列重排序       | 抽象schema       | 变更分区 |
| ------- | ------ | -------------- | -------------- | ------ | -------------- | ---------------- | -------- |
| Iceberg | 是     | 是             | 是             | 是     | 是             | 是               | 是       |
| Hudi    | 是     | 是(only spark) | 是(only spark) | 是     | 是(only spark) | 否(spark-schema) | 否       |
| Delta   | 是     | 是             | 是             | 是     | 是             | 否(spark-schema) | 否       |

hudi在删除列/更新列和列重排序目前仅支持用spark,其他方案没有限制.

这里 Iceberg 是做的比较好的,抽象了自己的 schema,不绑定任何计算引擎层面的 schema.同时还支持直接变更分区.

### 3. 读写生态

|               | Iceberg                                                      | Hudi                                                         | Delta                                                        |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Apache Spark  | [Read + Write](https://iceberg.apache.org/docs/latest/getting-started/) | [Read +   Write](https://hudi.apache.org/docs/quick-start-guide) | [Read + Write](https://docs.delta.io/latest/quick-start.html#set-up-apache-spark-with-delta-lake) |
| Apache Flink  | [Read +   Write](https://iceberg.apache.org/docs/latest/flink/) | [Read +   Write](https://hudi.apache.org/docs/flink-quick-start-guide) | [Read + Write](https://github.com/delta-io/connectors/tree/master/flink) |
| Presto        | [Read + Write](https://prestodb.io/docs/current/connector/iceberg.html) | [Read](https://prestodb.io/docs/current/connector/hudi.html) | [Read](https://prestodb.io/docs/current/connector/deltalake.html) |
| Trino         | [Read +   Write](https://trino.io/docs/current/connector/iceberg.html) | [Read](https://trino.io/docs/current/connector/hudi.html)    | [Read + Write](https://trino.io/docs/current/connector/delta-lake.html) |
| Hive          | [Read +   Write](https://iceberg.apache.org/docs/latest/hive/) | [Read](https://hudi.apache.org/docs/next/query_engine_setup/#hive) | [Read](https://github.com/delta-io/connectors/tree/master/hive) |
| DBT           |                                                              | [Read + Write](https://hudi.apache.org/blog/2022/07/11/build-open-lakehouse-using-apache-hudi-and-dbt) | [Read + Write](https://docs.databricks.com/integrations/prep/dbt.html) |
| Kafka Connect |                                                              | [Write](https://github.com/apache/hudi/tree/master/hudi-kafka-connect) | [Proprietary   only](https://docs.confluent.io/cloud/current/connectors/cc-databricks-delta-lake-sink/cc-databricks-delta-lake-sink.html) |
| Pulsar        | [Write](https://hub.streamnative.io/connectors/lakehouse-sink/2.9.2/) | [Write](https://hub.streamnative.io/connectors/lakehouse-sink/2.9.2/) | [Write](https://hub.streamnative.io/connectors/lakehouse-sink/2.9.2/) |
| Debezium      | [Write](https://debezium.io/blog/2021/10/20/using-debezium-create-data-lake-with-apache-iceberg/) | [Write](https://hudi.apache.org/cn/blog/2022/01/14/change-data-capture-with-debezium-and-apache-hudi/) | [Write](https://medium.com/everything-full-stack/streaming-data-changes-to-a-data-lake-with-debezium-and-delta-lake-pipeline-299821053dc3) |
| Kyuubi        | [Read + Write](https://kyuubi.readthedocs.io/en/v1.6.0-incubating-rc0/connector/flink/iceberg.html) | [Read + Write](https://kyuubi.readthedocs.io/en/v1.6.0-incubating-rc0/connector/flink/hudi.html) |                                                              |
| ClickHouse    |                                                              | [Read](https://clickhouse.com/docs/en/whats-new/changelog/#-clickhouse-release-2211-2022-11-17) | [Read](https://clickhouse.com/docs/en/whats-new/changelog/#-clickhouse-release-2211-2022-11-17) |
| Apache Impala | [Read + Write](https://impala.apache.org/docs/build/html/topics/impala_iceberg.html) | [Read + Write](https://hudi.apache.org/docs/querying_data/#impala-34-or-later) |                                                              |
| Databricks    | [Read + Write](https://docs.microsoft.com/en-us/azure/databricks/delta/delta-utility#--convert-an-iceberg-table-to-a-delta-table) | [Read +   Write](https://hudi.apache.org/docs/azure_hoodie/) | [Read +   Write](https://docs.databricks.com/delta/index.html) |
| Snowflake     | Read + Write                                                 |                                                              | [Read](https://docs.snowflake.com/en/sql-reference/sql/create-external-table.html#external-table-that-references-files-in-a-delta-lake) |
| Vertica       |                                                              | [Read](https://www.vertica.com/kb/Apache_Hudi_TE/Content/Partner/Apache_Hudi_TE.htm) | [Read](https://www.vertica.com/kb/Vertica_DeltaLake_Technical_Exploration/Content/Partner/Vertica_DeltaLake_Technical_Exploration.htm) |
| Apache Doris  | [Read](https://doris.apache.org/docs/ecosystem/external-table/iceberg-of-doris?_highlight=iceberg) | [Read](https://doris.apache.org/docs/ecosystem/external-table/hudi-external-table/) |                                                              |
| Starrocks     | [Read](https://docs.starrocks.com/en-us/main/using_starrocks/External_table#apache-iceberg-external-table) | [Read](https://docs.starrocks.com/en-us/main/using_starrocks/External_table#hudi-external-table) | [Preview](https://docs.starrocks.io/en-us/main/data_source/catalog/deltalake_catalog) |

可见三个数据湖框架都对Spark/Flink等主流的计算引擎的读写,同时也都支持大部分其他的主流技术,Iceberg支持的范围稍差一点,开源版本delta的暂时不支持直接连接kafka.

另外需要注意的是,虽然它们支持的读写框架很多,但是在某些功能可能只与特定的框架一起使用才有更完整的功能和更高的性能,例如Delta的读写其实有不少高级特性都只支持spark.

### 4. 社区现状

github repo对比:

|             | Iceberg    | Delta      | Hudi       |
| ----------- | ---------- | ---------- | ---------- |
| 开源时间    | 2018-11-06 | 2019-04-12 | 2019-01-17 |
| Github Star | 3.7k       | 5.6k       | 3.8k       |
| Github Fork | 1.4k       | 1.3k       | 1.7k       |
| Open Issue  | 790        | 204        | 155        |
| Close Issue | 1205       | 705        | 1896       |
| Open PR     | 405        | 61         | 338        |
| Close PR    | 4168       | 590        | 5262       |
| Releases    | 8          | 19         | 20         |

主要贡献方:

![未命名绘图](/Users/can.wang/Downloads/未命名绘图.jpg)

作为目前主流的开源数据湖方案,它们近期的社区活跃度都比较高.但是Delta和Hudi以后起之秀的姿态快速崛起,在开源社区的建设和推动方面明显强Iiceberg.

Delta的开源版和商业版本,提供了详细的内部设计文档,用户非常容易理解这个方案的内部设计和核心功能,同时Databricks还提供了大量对外分享的技术视频和演讲.Uber的工程师也分享了大量Hudi的技术细节和内部方案落地,同时在国内也在积极地推动社区建设,提供了官方的技术公众号和邮件列表周报.

### 4. 性能对比

![img](https://assets.website-files.com/61f38d7a4e41d6a673cd65c7/63beedba0ce0ca850f65eb69_MKya00GHAVf1JdQe1aQJw1-0F3rqNpP2or02x7sz_qQxhlcEcW5UCIQKj-88GRHaojWb-q5hlzJYJC0nFOAFXUac5uAI8nR0s-d28V-zJ-pRvr9nk4XgtGlPOwZQOtDqfXNMVOQ5OPkVskm2cBywSbk9kFQs7BxgTjC8janNHkpbKv2QFYx5mcOxhqud7w.png)

https://www.onehouse.ai/blog/apache-hudi-vs-delta-lake-transparent-tpc-ds-lakehouse-performance-benchmarks 

![img](https://assets.website-files.com/61f38d7a4e41d6a673cd65c7/63beedba88a2b73a9206681f_coCqF-RLFP8unfs38MQT8phpTPz8mIhgHq_nzrMPNZQc9Uh4VSPm0tHX50z8PleW5jLCbRimDirOJRN0dSyAzu7a8d0FBrdD77B25QOKPzeSpQUFOfXuZ9GxMWP1seuORzl8j0kQlLCwErJASjiFHSKJmarwSUqdkPDCRRVVQdv-3OoKkU42sLisq-CHgw.png)

https://brooklyndata.co/blog/benchmarking-open-table-formats 

可以看出delta和hudi性能不相上下,iceberg则相对逊色

### 对比总结

作为新一代数据湖方案,它们都有非常多相同的优点:

- 支持廉价储存
- 支持流批读写
- 完整ACID事务支持
- 数据多版本存储,支持时间旅行
- 可改变的table schema
- 开源的数据存储格式
- 周边工具集丰富

仅在某些技术细节上有比较明显区别,比如说Hudi支持Merge On Read表,支持增量查询,Iceberg高度抽象,能定义自描述的schema,delta与spark集成度更高等等.

从宏观层面看,无论是Iceberg/Delta还是Hudi都能满足我们当下以及可预见的未来需求,考虑到综合性能和社区活跃度,更倾向于使用Delta作为存储方案



## 云上产品对比

经过了解,有以下三种云产品方案可选:

- 阿里云EMR on ECS
- 阿里云EMR on ACK spark版
- Azure Databricks
- AWS Glue

### 阿里云EMR on ECS

![架构图](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/7365272361/p332638.png)

阿里云EMR on ECS支持Delta/Hudi/Iceberg作为数据湖存储方案,同时也可以集成DataWorks和Maxcompute等阿里云产品生态实现整个数据湖项目所需的任务调度/权限管理/监控/运维等需求,所有产品几乎都是开箱即用,运维方便且后续成本较低.

本方案由于几乎全线都是阿里云的产品,所以缺点比较明显:

- 跟阿里云深度绑定,迁移性很差
- Dataworks/Maxcompute做为数据集成/数据处理只能使用UI写代码和任务编排,无法集成CI/CD
- 集成其他开源组件较困难(因为是ECS直接安装,风险较高,便利性很低)
- 价格不低

### 阿里云EMR on ACK spark版

如果说阿里云EMR on ECS是使用阿里云的各类开箱即用的产品来尽量满足我们当前的需求,那EMR on ACK spark版就是基于spark on kubernetes的完全定制方案,EMR on ACK spark版是仅仅是在ACK上安装了spark相关的组件,以及支持使用DLF来管理元数据,其他的都需要自行搭建

![image-20230202150914419](/Users/can.wang/Library/Application Support/typora-user-images/image-20230202150914419.png)

这个方案的优缺点都很突出

优点:

- 无任何阿里云强绑定产品
- 可以按需引入开源组件
- 价格最便宜

缺点:

- 搭建成本较高
- 后期运维成本较高,主要体现在安全方面的维护
- 性能调优成本高

### Azure Databricks

Databricks是Spark商业公司Databricks的产品,生态及其丰富,除了spark和delta之外还集成了安全/notebook/权限/可视化调度/数据质量/CICD支持等等功能,且所有工具几乎都是开箱即用的,只不过阿里云没有databricks产品,中国区有databricks产品的可靠云服务商,也就只有Azure了.

![image-20230202161334611](/Users/can.wang/Library/Application Support/typora-user-images/image-20230202161334611.png)

![image-20230203100409481](/Users/can.wang/Library/Application Support/typora-user-images/image-20230203100409481.png)



本方案同样有缺点明显

优点: 

- 开箱即用,运维成本低
- 功能丰富,一个产品满足了所有的需求
- 性能强大,Databricks对spark和delta都有优化,性能可以达到开源版的2~3倍

缺点:

- 需要跨云作业,由于很多业务数据都在阿里云,所以需要通过专线连接阿里云接入数据
- 价格较高



### 详细对比

|          | Aliyun EMR on ECS                                      | Aliyun EMR on ECS spark版             | Azure Databricks             |
| -------- | ------------------------------------------------------ | ------------------------------------- | ---------------------------- |
| CI/CD    | 不支持                                                 | 支持                                  | 支持                         |
| Notebook | 自带                                                   | 自建                                  | 自带                         |
| Document | 较少                                                   | 依赖于开源组件                        | 较为详细                     |
| 易用性   | 开箱即用                                               | 大部分组件需要自建                    | 开箱即用                     |
| 弹性伸缩 | 按时间或负载伸缩                                       | 按负载/任务伸缩                       | 支持                         |
| 任务调度 | 依赖DataWorksB<br />支持拖拽                           | 自建Airflow<br />拖拽需要额外引入组件 | 开箱即用<br />支持拖拽和code |
| 数据安全 | 依赖DataWorks<br />权限/分级分类/隐私数据保护/风险预警 | 自建,功能按需加入                     | 开箱即用<br />               |
|          |                                                        |                                       |                              |
|          |                                                        |                                       |                              |
|          |                                                        |                                       |                              |
|          |                                                        |                                       |                              |
