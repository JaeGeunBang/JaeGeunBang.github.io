---
layout: post
title: "kafka-fluentd 연동"
categories:
  - Posts
tags:
  - fluentd
  - kafka
last_modified_at: 2018-07-09T12:57:42+09:00
---

이번에 kafka의 다른 데이터를 저장소로 옮길 때, fluentd를 이용했습니다.(elasticsearch, s3) <br>
fluentd에 fluent-kafka-plugin을 제공을 해주어 쉽게 데이터를 이동시킬 수 있었습니다. <br>
fluentd-kafka-plugin 버전은 0.7.3, ruby-kafka는 0.6.5를 이용했습니다.

**td-agent.conf**
```
<source>
  @type kafka_group
  brokers broker_host:9092
  topics "log_#{(DateTime.now).strftime('%y%W')}"
  consumer_group fluentd_server
  add_prefix kafka

  #start_from_beginning false
</source>

```
kafka_group을 이용했으며, topics은 주마다 새로 생성하는 관계로 위와 같이 년도,주 정보로 topic을 리스닝합니다. <br>
또한, start_from_beginning은 true (default)로 설정되어, 새로운 topic이 생성시 첫 message부터 읽을수 있도록 설정했습니다.<br>
다른 옵션 값들은 주지 않았습니다. <br>

kafka에서는 아래 명령을 통해 현재 consumer group이 읽고 있는 topic의 offset 정보를 가져와 확인할 수 있습니다.

```
./bin/kafka-consumer-groups.sh --bootstrap-server broker_host:9092 --group fluentd_server --describe
```
