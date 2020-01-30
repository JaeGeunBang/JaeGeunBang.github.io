---
layout: post
title: "HBase - Hadoop Database"
categories:
  - Posts
tags:
  - hbase
  - paper
  - hadoop
last_modified_at: 2020-01-13T12:57:42+09:00
---



해당 논문은 링크가 따로 없다.. HBase를 요약정리한 논문 정도로 보면 된다.



### Intro

<hr>

폭발적인 데이터의 성장으로 인해 기존 RDB 한계가 드러나며, NoSQL들이 등장했다.

- Apache Cassandra, Voldemort, HBase, Amazon Dynamo, Mongo DB, Redis 등
- 그중 HBase는 HDFS와 강하게 결합되며, structured petabyte 이상의 데이터를 다룰 수 있다.



### HBase

<hr>



HBase는 open-source column-oriented NoSQL DB라 정의할 수 있다.

- Java로 씌여졌으며, Hadoop의 HDFS 위에서 동작한다.
- 특징
  - distributed fault-tolerance
  - horizontal scalability
  - automatic sharding
  - automatic failover support
  - random read/write access
  - Java API 제공
  - MapReduce Job
- 이러한 특징들은 HDFS에 저장된 key-value 데이터에 real time Query를 가능한다. 또한 MapReduce를 통해 batch 처리 또한 가능하다.(?)
- 크게 3가지 모드가 있다.
  - Standalone - 하나의 Java Process가 local 머신에서 동작한다.
  - Pseudo-distributed - 여러 Java Process가 local 머신에서 동작한다.
  - Fully-distributed - 여러 Java Process가 여러 머신에서 동작한다. (네트워크를 통해 연결된 머신들)



**Logical Architecture**

- Master-Slave 구조를 가지며, HMaster, HRegions 로 구성된다.
  - HMaster는 Master 노드이며, 클러스터 관리, region 서버에 데이터 할당, failover 컨트롤, 로드 밸런싱과 같은 역할을 한다.
  - HRegions는 실제 데이터가 저장되며, read/write request 처리, splitting regions, client와의 interaction 과 같은 역할을 수행한다.

![2](https://user-images.githubusercontent.com/22383120/72341365-62358a80-370d-11ea-8738-f9f78dff4124.PNG)



**Data Model**

- ***Tables***: row들의 logical collection을 뜻하며, 여러 HRegion에 저장되어 있다.
- ***Rows***: 데이터 접근을 위한 unique한 row key를 가지며, 더불어 CF, Column Qualifier도 가진다.
- ***Column Families***: row 내 organized된 데이터이다. CF는 하나 이상의 column들로 이루어져 있으며 사전에 정의되어야 한다.
- ***Column Qualifier***: CF내 정의된 column이다. 빈값이여도 상관 없다. Column을 좀더 세분화 하고 싶을때 사용하는 것 같음.
- ***Cell***: row, CF, Column qualifier를 결합하며, unique 하게 구분되는 단위이다.
- ***Version/Timestamp***: cell 내 모든 value들은 versioned 되어지며, 데이터 접근이 발생할 때 가장 최신 Version 데이터를 읽게된다. (최근 시간)

![1](https://user-images.githubusercontent.com/22383120/72341137-dcb1da80-370c-11ea-9da3-48addf870cea.PNG)

**특징**

- Sorted rowkeys
  - Hbase 내 데이터는 rowkey 기반으로 정렬되어 있다. 그렇기 때문에 `range 검색`이 가능하며, `빠른 read 성능`을 보인다.
  - 보통 최근 record의 검색을 초점을 맞추는 application은 `rowkey에 시간`을 넣는다. 그럼 시간순대로 정렬될 것이며, `시간 range 검색`이 가능해진다.
- Control on data sharding
  - 데이터를 어떻게 클러스터 내 distributed 할지 결정할 수 있다. 
  - rowkey를 어떻게 하냐에 따라 특정 노드에 데이터가 hot(?) 할 수 있기 때문에, `rowkey는 고르게 분포될 수 있도록 선정해야한다.`
- Strong Consistency
  - CAP 이론에 따르면, HBase는 CP를 기반으로 동작한다. 즉. `항상 read에 대한 consistency를 보장`한다. (어느 노드에서 데이터를 읽더라도 같은 데이터를 반환해준다.)
  - 단, write 성능은 떨어진다. (tradeoff). application이 read 성능보다 write가 더 중요하다면 HBase는 올바른 선택은 아닐것이다.
- Scalability
  - HBase에 노드를 추가할 수록 성능도 linear하게 상승할 것이며 쉽게 확장할 수 있다.
- Automatic failover support
  - 높은 availability를 제공하는데, 이는 기본 데이터를 replica 하기 때문이다. (HDFS의 특징을 가져감)
- API Support
  - Java API를 제공한다.



### HBase 선택 기준

<hr>

HBase를 선택하는데 여러 기준이 있다.

- Data Volumns
  - `patebyte 이상의 unstructured data를 저장, 분석하기 위해 사용하기 적합`하다.
  - 허나, 소규모 데이터에 사용하기에는 부적절하다.
- Appliation type
  - `key 기반의 데이터를 접근하거나, 스키마의 변동이 심할때 사용하기 적합`하다.
- Hardware Environment
  - HBase는 HDFS위에서 동작하며, Hardware resource가 성능에 영향을 끼친다.
  - 최소 5대 이상의 node를 구축하는 것이 좋다.
- No Rqeuirement of relational feature
  - `RDB가 가지는 Trigger, Complex join 이나 query의 요구가 없을 때 사용하기 적합`하다.
- Quuck Access to data
  - `실시간 random read/write data access의 요구가 많을때 사용하기 적합`하다.

이외 다양한 Use Case들이 있다.

- web-search, incremental data, Content Serving, Information Exchange 등



### 카산드라와의 비교

<hr>

 ![3](https://user-images.githubusercontent.com/22383120/72342571-17694200-3710-11ea-86a2-9b50a1e88d9b.PNG)

둘의 주요 차이점은 카산드라도 HBase와 마찬가지로 NoSQL DB지만, `CAP 이론의 AP를 기반으로 동작`한다.

write 성능에 좀더 초점을 맞추고 싶다면 카산드라를 사용하는 것이 좋다.







