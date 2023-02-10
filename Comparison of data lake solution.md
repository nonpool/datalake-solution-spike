## Current Pain Point

1. It will be a serious flaw that if the system only support us copy and paste the code to Simba web page integrated editor, which means we cannot do DevOps or deployment automatically. Imaging if we have development, QA and production environments, then the data engineers would copy and paste the code manually from page every day. These large number of manual operations would make the entire development process low reliability and high complexity.
2. The user experience in web editor is extremely poor, and there is no engineering experience.
- Breakpoints are not supported. Some development resources can only be accessed online, which means we need to debug the whole module in the editor of the simba webpage. However, simba does not support this, and thus making the development process extremely painful.
- Files cross import is not supported. This will lead to the code smell. If many modules have the same function, then you have to write repeated code in each file which leads to code redundancy. And for more complex functions, the amount of code in a file may be extremely large, which is difficult to maintain.
- Passing parameters or environment variables is not supported. It would lead to security issues if we leave sensitive information, for example, database account and passwords as plain text on Simba webpage without encryption.
3. There are almost no relevant reference documents, all functions can only be explored by ourselves, and the cost of getting started is relatively high. This problem is especially obvious in multi-team collaboration.
4. It is hard to do trouble-shooting while the code execution failed since the error messages on webpage is quite limited.
5. The python code on the simba actually parses the '#' symbol in string into a comment, causing failure in code execution. Which also reflects the low reliability of the system.
6. At present, spark/flink on simba are all unavailable, which is also unacceptable for big data processing.
7. The processing efficiency is low. Usually, one simple SQL execution takes more than 10 seconds. When there is a large amount of data to be processed, things would become much worse. However, it is difficult to directly optimize it since it does not support Spark.
8. Currently, the data reuse rate in Simba is low, and there is no data development standard. Both data and code are messy, with a lot of redundant data and redundant processing.
9. There is no data quality monitoring. Without data quality auto checking, we have to manually post-check data correctness during the whole data process.
10. The data is directly stored on the ECS disk without any backup mechanism. It is also hard to migrate. 
11. The system is unstable.For example, 504 gateway timeout often occurs while executing, the background task is still executing while the task has already been killed in page, etc.



Based on the current pain points, we combined the current mainstream solutions and their actual situation on Alibaba Cloud to make a new technology selection:



## Current Mainstream Data Lake Storage Solutions

- Databricks Delta Lake

<img src="https://camo.githubusercontent.com/5535944a613e60c9be4d3a96e3d9bd34e5aba5cddc1aa6c6153123a958698289/68747470733a2f2f646f63732e64656c74612e696f2f6c61746573742f5f7374617469632f64656c74612d6c616b652d77686974652e706e67" alt="Delta Lake Logo" style="zoom: 25%;" />

Delta Lake is an open source project of Apache Spark on Databricks. It is an important part of Databricks's expansion of the Spark ecosystem. With deep integration of Spark, we can easily enjoy the convenience brought by the Spark ecosystem.

Official Introduction: Delta Lake is an open-source storage framework that enables building a Lakehouse architecture with compute engines including Spark, PrestoDB, Flink, Trino, and Hive and APIs for Scala, Java, Rust, Ruby, and Python.

![image-20230112154210961](https://raw.githubusercontent.com/nonpool/storyImg/master/img/image-20230112154210961.png)

- Apache Hudi

<img src="https://camo.githubusercontent.com/167d7939912a5b6487ee0c6c38374215ebd740386d5e61b10356b065243b61a4/68747470733a2f2f687564692e6170616368652e6f72672f6173736574732f696d616765732f687564692d6c6f676f2d6d656469756d2e706e67" alt="Hudi logo" style="zoom:50%;" />

Apache Hudi (Hadoop Upsert And Incremental) is an open source project incubated within Uber. The main purpose of the design at the beginning was to solve the pain point that traditional data warehouses cannot quickly upsert data. After open source, it is developing towards the direction of the next generation of streaming data lake platform.

Official Introduction:Apache Hudi is a transactional data lake platform that brings database and data warehouse capabilities to the data lake. Hudi reimagines slow old-school batch data processing with a powerful new incremental processing framework for low latency minute-level analytics.

Architecture Diagram:
![Hudi Data Lake](https://hudi.apache.org/assets/images/hudi-lake-overview-e39f80337517a0a1999d8eb5cd0ac965.png)



- Apache Iceberg

<img src="https://camo.githubusercontent.com/d120d4367f4fcaea0086ec2533ecad35c4ce2fadc313071ee2c26ff319833168/68747470733a2f2f696365626572672e6170616368652e6f72672f646f63732f6c61746573742f696d672f496365626572672d6c6f676f2e706e67" alt="Iceberg" style="zoom:50%;" />

Apache Iceberg is an open source project incubated within Netflix. It was originally self-developed to solve defects of Hive, and later developed into a general data lake project with a highly abstract design.

Official Introduction:Iceberg is a high-performance format for huge analytic tables. Iceberg brings the reliability and simplicity of SQL tables to big data, while making it possible for engines like Spark, Trino, Flink, Presto, Hive and Impala to safely work with the same tables, at the same time.

Architecture Diagram:
![Comp-1](https://raw.githubusercontent.com/nonpool/storyImg/master/img/Comp-1.gif)



## Detailed Comparison

### 1. ACID and Isolation Level

|         | ACID | Isolation Level                                     | Concurrent Write | Time-Travel |
| ------- | ---- | --------------------------------------------------- | ---------------- | ----------- |
| Iceberg | yes  | Serializable/Snapshot Isolation                     | yes              | yes         |
| Hudi    | yes  | Snapshot Isolation                                  | yes              | yes         |
| Delta   | yes  | Serializable/Snapshot Isolation/Write Serialization | yes              | yes         |

Serializable: All readers and writers must execute serially.

Snapshot Isolation: If the data written by multiple writers has no intersection, they can be executed concurrently, otherwise they can only be executed serially. Reader and writer can execute at the same time.

Write Serialization: Multiple writers must be strictly serialized, while readers and writers can be executed at the same time.

Above three frameworks are all excellent, all support snapshot isolation with the best concurrency as well as support ACID/concurrent writing and Time-Travel. 

### 2. Schema Changes and Design

|         | add column | delete column    | rename column    | update column | column reordering| abstract schema   | change partition |
| ------- | ---------- | ---------------- | ---------------- | ------------- | ---------------- | ----------------- | ---------------- |
| Iceberg | yes        | yes              | yes              | yes           | yes              | yes               | yes              |
| Hudi    | yes        | yes (Spark only) | yes (Spark only) | yes           | yes (Spark only) | no (Spark-schema) | no               |
| Delta   | yes        | yes              | yes              | yes           | yes              | no (Spark-schema) | no               |

Currently, hudi only supports Spark in deleting/updating columns and column reordering operations while other solutions have no restrictions.


Iceberg does a better job here, abstracting its own schema and not binding any schema at the computing engine level. It also supports direct partition changes.

### 3. Read and Write Ecology

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

It can be seen that all above three data lake frameworks can read and write mainstream computing engines such as Spark and Flink.  In additon, also support most other mainstream technologies. With comparison, Iceberg supports smaller scope of technologies while Delta (open source edition) temporarily does not support Kafka Connect.

In addition, it should be noted that although they support many reading and writing frameworks, some functions may only be used with specific frameworks to have more complete functions and higher performance. For example, the reading and writing of Delta actually has many advanced features that only support Spark.

### 4. Community Current Situation

Github Repo Comparison

|                       | Iceberg    | Delta      | Hudi       |
| --------------------- | ---------- | ---------- | ---------- |
| open source start time| 2018-11-06 | 2019-04-12 | 2019-01-17 |
| Github Star           | 3.7k       | 5.6k       | 3.8k       |
| Github Fork           | 1.4k       | 1.3k       | 1.7k       |
| Open Issue            | 790        | 204        | 155        |
| Close Issue           | 1205       | 705        | 1896       |
| Open PR               | 405        | 61         | 338        |
| Close PR              | 4168       | 590        | 5262       |
| Releases              | 8          | 19         | 20         |

Main Contributors
![未命名绘图](https://raw.githubusercontent.com/nonpool/storyImg/master/img/%E6%9C%AA%E5%91%BD%E5%90%8D%E7%BB%98%E5%9B%BE.jpg)

As the current mainstream open source data lake solutions, their recent community activity is relatively high. However, Delta and Hudi have risen rapidly as rising stars, and they are obviously stronger than Iiceberg in the construction and promotion of open source communities.

The open source version and commercial version of Delta provide detailed internal design documents, making it easy for users to understand the internal design and core functions of this solution. In addition, Databricks also share a large number of technical videos and speeches to public. Uber's engineers also shared a lot of Hudi's technical details and internal solutions. At the same time, they are also actively promoting community building in China, providing official technical public accounts and weekly mailing lists.

### 5. Performance Comparison

![img](https://assets.website-files.com/61f38d7a4e41d6a673cd65c7/63beedba0ce0ca850f65eb69_MKya00GHAVf1JdQe1aQJw1-0F3rqNpP2or02x7sz_qQxhlcEcW5UCIQKj-88GRHaojWb-q5hlzJYJC0nFOAFXUac5uAI8nR0s-d28V-zJ-pRvr9nk4XgtGlPOwZQOtDqfXNMVOQ5OPkVskm2cBywSbk9kFQs7BxgTjC8janNHkpbKv2QFYx5mcOxhqud7w.png)

https://www.onehouse.ai/blog/apache-hudi-vs-delta-lake-transparent-tpc-ds-lakehouse-performance-benchmarks 

![img](https://assets.website-files.com/61f38d7a4e41d6a673cd65c7/63beedba88a2b73a9206681f_coCqF-RLFP8unfs38MQT8phpTPz8mIhgHq_nzrMPNZQc9Uh4VSPm0tHX50z8PleW5jLCbRimDirOJRN0dSyAzu7a8d0FBrdD77B25QOKPzeSpQUFOfXuZ9GxMWP1seuORzl8j0kQlLCwErJASjiFHSKJmarwSUqdkPDCRRVVQdv-3OoKkU42sLisq-CHgw.png)

https://brooklyndata.co/blog/benchmarking-open-table-formats 

It can be seen that the performance of delta and hudi are comparable, while iceberg is relatively inferior.

### Conclusion

As a new generation of data lake solutions, they all have many of the same advantages

- Support cheap storage
- Support stream batch read and write
- Full ACID transaction support
- Data multi-version storage, support time travel
- Changeable table schema
- Open source data storage format
- High tool compatibility

The above are only obvious differences in some technical details. For example, Hudi supports Merge On Read tables and incremental queries. Iceberg is highly abstract and can define a self-describing schema. And delta is more integrated with spark etc.

From a macro perspective, both Iceberg/Delta and Hudi can meet our current and foreseeable future needs. But from the perspective of comprehensive performance, both Dleta and Hudi are better choices than Iceberg. In addition, Delta has the support of a mature commercial cloud product (Databricks), which will have an advantage over Hudi. Therefore, we will give priority to using Delta as a data lake storage solution.


## Cloud Product Comparison

After research, there are the following three cloud product solutions to choose

- Alibaba Cloud EMR on ECS
- Alibaba Cloud EMR on ACK (Spark version)
- Azure Databricks
- AWS Glue

### Alibaba Cloud EMR on ECS

![架构图](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/7365272361/p332638.png)

Alibaba Cloud EMR on ECS supports Delta, Hudi, Iceberg as a data lake storage solution, and can also integrate Alibaba Cloud product ecology such as DataWorks and Maxcompute to realize the task scheduling, authority management, monitoring, operation and maintenance requirements required by the entire data lake project. Almost all products are out of the box, easy to operate and maintain, and low follow-up costs.

Since almost all products of this solution are Ali Cloud products, the disadvantages are obvious

- Deep binding with Alibaba Cloud, poor migration
- Dataworks/Maxcompute as data integration/data processing can only use UI to write code and task arrangement, and cannot integrate CI/CD
- It is difficult to integrate other open source components (it has high risk and low convenience to integrate since it is directly installed by ECS)
- high price

### Alibaba Cloud EMR on ACK (Spark Version)

Compared with Alibaba Cloud EMR on ECS  which uses various out-of-the-box products of Alibaba Cloud to meet our current needs, the EMR on ACK Spark version is a fully customized solution based on spark on kubernetes. The EMR on ACK Spark version only installs spark-related components on ACK, and supports the use of DLF to manage metadata. Except these, other components need to be built by themselves.

![image-20230202150914419](https://raw.githubusercontent.com/nonpool/storyImg/master/img/image-20230202150914419.png)

The pros and cons of these solutions are obvious

pros:

- No Alibaba Cloud strong binding products
- Open source components can be introduced on demand
- Cheapest price

cons:

- Higher construction cost
- The post-operation and maintenance cost is relatively high, mainly reflected in the maintenance of security
- Expensive performance tuning

### Azure Databricks

Databricks is a product of Spark Databricks with a rich ecology. In addition to spark and delta, it also integrates functions such as security, notebook, permissions, visual scheduling, data quality and CICD support. Nearly all tools of above are out of the box. However, Alibaba Cloud does not have databricks products. There is only Azure as a reliable cloud service provider for databricks products in China.

![image-20230203100409481](https://raw.githubusercontent.com/nonpool/storyImg/master/img/image-20230203100409481.png)

The pros and cons of these solutions are also obvious

pros: 

- Out of the box, low operation and maintenance costs
- Rich in functions, one product meets all needs
- Powerful performance, Databricks has optimized both spark and delta, the performance can reach 2~3 times of the open source version

cons:

- Need to work across clouds. Since a lot of business data is in Alibaba Cloud, it is necessary to connect to Alibaba Cloud to access data through their specific way
- High costs



### Detailed Comparison
|                    | Alibaba Cloud EMR on ECS                                                                                        | Alibaba Cloud EMR on ECS (Spark version)                        | Azure Databricks                                                                                          | AWS Glue                                                        |
| ------------------ | --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| CI/CD              | Unsupported                                                                                                     | Supported                                                       | Supported                                                                                                 | Supported                                                       |
| Notebook           | Self-contained                                                                                                  | Self-built                                                      | Self-contained                                                                                            | Self-contained                                                  |
| Document           | Less                                                                                                            | Depends on open source components                               | Detailed                                                                                                  | Detailed                                                        |
| Ease of Use        | Need to be used with other cloud components                                                                     | Most components need to be self-built                           | Out of box                                                                                                | Need to be used with other cloud components                     |
| Migration          | Weak                                                                                                            | Strong                                                          | Middle                                                                                                    | Middle                                                          |
| Elastic Expansion  | Scale by time or load                                                                                           | Scale by time/load/task                                         | Scale by load/task                                                                                        | Serverless                                                      |
| Task Scheduling    | Depends on DataWorks<br />Support drag and drop                                                                 | Self-built Airflow<br />Dragging requires additional components | Self-contained<br />Support drag and drop and code control                                                | Self-contained <br />Support drag and code control              |
| Data Security      | Depends on DataWorks<br />Permission/ hierarchical classification/ privacy data protection/risk warning/ etc on | Self-built, Functions added on demand                           | Self-contained<br />Access control/encryption/log audit, etc                                              | Dependent on other components                                   |
| Component Safety   | Vulnerability repair/ security authentication/ intranet Access                                                  | Intranet access                                                 | Penetration test/vulnerability repair/security certification/intranet access/contract commitment          | Serverless, the security of all components is guaranteed by AWS |
| Data Governance    | Self-contained quality rule template/ data map/ data operation review                                           | Self-built data consanguinity/custom rules                      | Best solution, with data quality control/ data lineage/ data operation review                             | Rely on Glue data catalog or AWS Lake Formation                 |
| Permission Control | Self-contained cluster/ workspace/ job/ member/ resource/ data source access control                            | Rely on Ranger and other open source components                 | Self-contained workspace object/ cluster/ pool/ job/ table/ Access control of confidential data           |Self-contained job access control                                |
| Price              | Middle<br />Software service fee + resource usage fee<br />Dataworks Enterprise Edition: 20000 RMB/month<br />  | Low<br />ACK fee only (resource usage fee)                      | High<br />DBU + resource usage fee<br />DBU will charge according to the size of hardware resources<br /> | Middle<br />Calculate resource + other component usage fee      |
| Maintenance Costs  | Low                                                                                                             | High                                                            | Low                                                                                                       | Low                                                             |
