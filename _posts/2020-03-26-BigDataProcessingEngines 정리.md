---
layout: post
title: "BigDataProcessingEngines 정리"
categories:
  - Posts
tags:
  - druid
  - hadoop
  - hbase
  - hive
  - phoenix
last_modified_at: 2020-03-26T12:57:42+09:00P
---









여기서 Apache Hive, Phoenix, HBase Druid를 소개한다.



Apache Hive

- Apache ORC라는 columnar storage format을 사용하여, OLAP와 SQL query processing을 수행한다.



Apache Phoenix / HBase

- OLTP database로써 수많은 record에 실시간 쿼리를 가능하데 하며, random key 기반의 lookup과 update를 가능하게 한다.



Apache Druid

- 실시간 time-series 분석과 OLAP 분석을 low latency 내로 가능하게 하는 데이터 스토어이다.



**공통점**

1. Caching 기능을 지원한다.
   - HBase는 BlockCache, Hive는 LLAP IO layer, Druid는 in-memory caching options을 제공한다.
   - 기존 수행된 쿼리 데이터를 메모리에 가지고 있다. 만약, 기존 쿼리에 Filter만 추가한다면 비싼 disk read를 하지 없이 query를 수행할 수 있다.
2. 특정 데이터를 바로 접근할 수 있다.
   - HBase는 HashMap 기반 O(1) random access를 지원하며, Druid는 inverted bitmap index를 사용해 어떤 Column 값이 어떤 Row에 있는지 알수 있다. Hive table은 statistics, index, partitioing 정보를 가진다.
3. HDFS를 공통적으로 사용한다.
   - 각자 기술만의 fault tolerance를 위해 HBase는 region replication을 제공하며, Druid는master, worker의 duplication, Hive는 Yarn, HDFS에서 제공하는 fault-tolerant logic을 그대로 사용한다.



**차이점**

Apache Hive

- OLAP engine으로 다양한 폭으로 사용된다. 기본 HDFS을 사용하며, 다양한 data format을 지원한다.
- 이중 가장 최적화된 format은 ORC이며, 데이터를 ORC를 변환하면 `압축` 되며, column 기반으로 `disk에 sequentially 하게 저장`할 수 있다.
  - LLAP을 통해 disk에서 data를 읽어 memory에 multiple time 동안 저장할 수 있다.
  - 즉, `LLAP과 Hive를 통해 ad-hoc 분석, 대규모 aggregation, low-latency reporting이 가능하다.`
  - 뿐만 아니라, concurrent query, ACID transantion도 제공된다. 
  - druid, hbase는 특화된 성능을 제공한다면, Hive는 모든 trade(?) 고려한다.

- 요약
  - ACID, realtime database, EDW (Enterprise Data Warehouse)
  - Unified SQL interface, JDBC 제공
  - Reporting, Batch 용도
  - Join, large, aggregate, ad-hoc 등



Apache Phoenix / HBase

- 분산 key-value store로, random read, write, update, delete가 가능하며, OLTP 엔진이다.
- aggregation, joining 하기 위한 용도로 사용되지 않는다.
- Phoenix를 통해 SQL을 기반으로 Hbase를 조회할 수 있으며 Secondary Index도 지원함. 
  - Secondary Indax를 통해 Primary Index가 아닌 Column에 대해서도 빠른 조회가 가능하다.
  - 대신, Phoenix는 대규모 데이터에는 적합하지 않다.
- 요약
  - 빠른 low latency random access (key 기반의 lookup)
  - large-volumn OLTP
  - update, delete 가능



Apache Druid

- 빠른 OLAP query를 수행하며, streaming data에 대해 실시간 indexing이 가능하다.
  - cube-speed OLAP query를 제공한다.
  - ex) 특정 2주 기간 내 이탈리아 여행 최저가 항공권을 알고 싶을 때,
- 몇 초 내 수억 또는 수십억 개의 row를 정확하게 찾아내는데 탁월하며, 뿐만 아니라 빠른 aggregate value를 구할 수 있다.
  - 허나 join을 통한 분석은 불가능 하다. (join이 가능한 OLAP engine인 `Kylin` 도 있다.)
  - 만약 druid를 통해 join과 같은 분석이 필요하다면, pre-join된 데이터를 준비해야 함.

- 요약
  - 빠른 low latancy OLAP 제공
  - Aggregations, drilldown
  - Time-series, Real-time Ingestion 제공



참고

https://blog.cloudera.com/big-data-processing-engines-which-one-do-i-use-part-1/