---
layout: post
title: "Apache Hive: From MapReduce to Enterprise-grade Big Data Warehousing (SIGMOD'19)"
categories:
  - Posts
tags:
  - hive
  - paper
  - hadoop
last_modified_at: 2020-01-20T12:57:42+09:00
---



## Intro



Apache Hive는 analytic big-data workloads 를 위한 open-source relational database system이다. 

- Hive는 약 10년전 Hadoop에 SQL-like interface를 제공했으며, 주로 ETL이나 batch reporting workloads 용도로 사용되었다.
- 허나 최근 공통적으로 요구하는 니즈는 low-latency SQL engine이다.
- Hive는 이러한 요구를 만족시켜주기 위해 renovation, evolve가 필요하며, 이를 통해 common data warehousing technique로 채택될 것이다.



Hive를 renovation 하기 위한 연구들은 많이 있었으며, 그 중 크게 4가지로 분류할 수 있다.

- SQL and ACID support
  - insert, delete 뿐만 아니라, update, merge 지원도 가능하다.
  - ACID guarantee를 제공한다. (Snapshot Isolation)
- Optimization techniques
  - Apache Calcite를 통해 쿼리 최적화를 한다.
  - 여기서 공통으로 사용하는 기술로, query reoptimziation, query result caching, materialized view rewriting 등이 있다.
- Runtime Latency
  - 빠른 latency를 위해 columnar data store, vectorization of operator을 지원하며, MapReduce를 Tez, Spark Engine으로 바꿀수 있다.
  - 추가로 Yarn의 start-up overhead가 발생하기 때문에, 이는 LLAP (Long-running executor)을 사용한다.
- Fedaration Capability
  - 다양한 시스템들과 쉽게 결합할 수 있는 환경을 갖춘다.



## 아키텍쳐



![1](https://user-images.githubusercontent.com/22383120/72703463-fb92ef80-3b98-11ea-859b-21c6357f7bfc.PNG)

Data Storage (File format, File System)

- File System은 HDFS 뿐만 아니라 AWS S3, Azure Blob Storage, 이외 Druid, HBase를 통해서 데이터를 읽고 쓸 수 있다.
- File Format은 ORC, Parquet 등이 있다.



Data catalog

- Hive Metastore (HMS)에 모든 data source에 대한 information 정보를 저장한다. 
  - HMS는 data catalog이며, 모든 data를 Hive에 의해 쿼리를 할 수 있다.
- 기본 RDBMS를 사용해 information을 저장한다.
- HMS API는 여러 programming language를 지원하며, Thirft를 이용해 서비스가 실행된다.



Exchangeable data processing runtime

- 기본 MapReduce Engine을 사용하지만 이는 Tez, Spark Engine으로 쉽게 교체할 수 있다.



Query Server

- Hive Server2 (HS2)는 사용자가 Hive로 SQL query를 날릴 수 있게 한다.
  - 기본 JDBC, ODBC를 지원하며, JDBC thin client를 포함하는데 이는 `beeline`이라 한다.

![2](https://user-images.githubusercontent.com/22383120/72703874-35182a80-3b9a-11ea-855f-80771f38d6fe.PNG)



- 위 그림은 SQL query를 HS2가 받아 executable plan으로 바꾸는 과정을 보여준다.
- 단계
  - 1. 먼저 SQL query를 HS2에 제출하면, driver에 의해 query가 handle 되며, Parser를 통해 AST로 변환하고, 이를 통해 Calcite Logical Plan을 만든다.
    2. Logical plan을 Calcite가 optimization 한다. 참고로 HS2는 data source에 대한 information을 HMS를 통해 접근한다.
    3. optimized 된 logical plan을 이후 physical plan으로 바꾸며, 이는 data partitioning, sorting에 관한 plan 이다.
    4. physical plan은 추가적인 optimization을 하여 vectorized paln이 만들어 진다.
    5. 이후 task Compiler (ex. Tez Compiler)에 의해 실제 executable task들이 만들어지고, 이는 Yarn에 제출된다.



## SQL and ACID Support

SQL support

- Standard atomic SQL data type과 non-atomic type SQL type을 지원한다.
  - non-atomic type SQL type은 STURCT, ARRAY, MAP
- 또한, Hive는 correlated subquery를 위한 확장 지원한다.
  - advanced OLAP operator (grouping set, window function set 등), integrity contraint, outer query로 부터 column 참조 등등
- 기존 유용한 feature로 PARTITION BY를 그대로 사용한다.
  - table을 creation time 기준으로 나뉘어 directory를 구분하며, 이는 scanning 할 때 필요한 directory만 읽을 수 있다.
- 즉, 기존 feature들과 extended된 correlated subquery를 지원한다.



ACID implementation

- 기존 Hive는 insert, delete만 지원했지만, 현재 full DML (update, merge 추가 지원), ACID transaction을 지원한다.
  - ACID guarantee는 Snapshot Isolation을 통해 제공된다.
- 현재 Transaction은 single statement만 다룰 수 있다. (즉, Insert, delete, update, merge 중 한번에 한가지만 수행 가능)
  - 하지만, transaction을 통해 multi-insert statement는 가능하다.
- 주요 row level operation의 부족함을 극복하기 위해 두 가지가 필요하다.
  - 1. transaction manager의 부족
    2. file update 지원의 부족 (HDFS 기반 스토리지)
- Transaction and lock management
  - Transaction과 lock에 대한 정보는 HMS에 저장된다.
    - Transaction은 TxnId로 구분되며, 이는 1개 또는 그 이상의 multiple write transaction을 mapping 한다.
      - write id는 각 테이블과 매핑되며, 같은 테이블에 있는 record들은 같은 write id를 가진다.
      - file id는 같은 write id를 가지는 여러 file들을 구분하기 위함이다.
      - row id는 file 내에 있는 record를 구분하기 위함이다.
    - 즉, 최종 테이블에 write된 record는 `write id, file id, row id의 조합으로 unique 하게 구분`할 수 있다.
    - 이러한 정보를 바탕으로 delete 작업(?)을 수행할 수 있다.
- Data and file layout
  - Hive는 데이터를 각 테이블에 저장하며, partition을 서로 다른 directory로 운영한다. (동시의 read나 write를 위함)
  - 각 테이블, partition을 위해 두 종류의 directory를 사용한다. (base, delta)
    - base: 데이터를 실제 저장하는 directory
      - ex) base_100 dir에 write 100 까지의 record들이 저장된다.
    - delta: insert, delete record를 위한 directory로 transaction에 관한 정보를 저장한다.
      - ex) delta_101_101 dir에 101 write id를 통해 생성된 record transaction 정보(?)가 저장된다.
    - delete된 record는 base에 저장된 record와 delta에 저장된 delete transaction 정보를 anti-join을 통해 알 수 있다.
      - delta file은 대부분 작기 때문에, merging phase (compaction) 단계가 필요하다.
    - update된 record는 insert, delete transaction을 바탕으로 알 수 있다.
  - Compaction (merging)
    - minor compaction: 델타 파일이 많을 때, 하나로 합치는 방법이다.
    - major compaction: 델타 파일이 클 때, base file에 적용 후 델타 파일을 초기화 한다.



## 참고

https://arxiv.org/pdf/1903.10970.pdf

