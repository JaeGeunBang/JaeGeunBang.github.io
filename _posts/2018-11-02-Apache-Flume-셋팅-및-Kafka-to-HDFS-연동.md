---
layout: post
title: "Apache Flume 셋팅 및 Kafka to HDFS 연동"
categories:
  - Posts
tags:
  - hadoop
  - flume
last_modified_at: 2018-11-02T12:57:42+09:00
---



### Flume 홈페이지에서 다운로드 받음 
- https://flume.apache.org/
- 압축을 풀고, flume-env.sh, flume.conf, bash.rc를 각 설정함
  - bash.rc (Hadoop도 설치 후 경로를 잡아야 함 - ** 없으면 no schema hdfs 에러가 발생함.**)

```
  export JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk-amd64"
  export FLUME_HOME="/home/jaegeun/apache-flume-1.8.0-bin"
  export FLUME_CLASSPATH="/home/jaegeun/apache-flume-1.8.0-bin/lib"
  export HADOOP_HOME="/home/jaegeun/hadoop-2.9.1"
  export HADOOP_HDFS_HOME=$HADOOP_HOME
```
  - flume.conf (기존 파일명 flume-conf.properties.template)
    - agent이름은 agent1로 지정했으며, 각 sources, channles, sinks를 지정함
    - sources는 Kafka, channles는 Memory, sinks는 hdfs로 설정
    
```
  agent1.sources = source1
  agent1.channels = channel1
  agent1.sinks = sink1

  agent1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
  agent1.sources.source1.kafka.bootstrap.servers = 192.168.56.171:9092
  agent1.sources.source1.kafka.topics = tlog_kr
  agent1.sources.source1.kafka.consumer.group.id = flume
  agent1.sources.source1.channels = channel1
  agent1.sources.source1.interceptors = i1
  agent1.sources.source1.interceptors.i1.type = timestamp
  agent1.sources.source1.kafka.consumer.timeout.ms = 100.sinks.loggerSink.type = logger

  agent1.channels.channel1.type = memory
  agent1.channels.channel1.capacity = 1000
  agent1.channels.channel1.transactionCapacity = 1000

  agent1.sinks.sink1.type = hdfs
  agent1.sinks.sink1.hdfs.path = hdfs://192.168.56.191:9000/user/flume/%y-%m-%d
  agent1.sinks.sink1.hdfs.rollInterval = 5
  agent1.sinks.sink1.hdfs.rollSize = 0
  agent1.sinks.sink1.hdfs.rollCount = 0
  agent1.sinks.sink1.hdfs.fileType = DataStream
  agent1.sinks.sink1.channel = channel1
```

  - flume-env.sh (기존 파일명 flume-env.sh.template)
  
```
  export JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk-amd64"
```
  
- 실행
  - ./bin/flume-ng agent -n agent1 -c ./conf/ -f ./conf/flume.conf -Dflume.root.logger=INFO,console
  - flume-ng의 agent 커맨드로 실행하며, 
    - -n: 위에서 지정한 agent1로 설정함. 다른 root의 데이터 전송을 하고 싶다면, 새로운 이름의 agent 생성 후 실행하면 됨
    - -c, -f: config 파일 지정
    - Dflume.root.logger=INFO,console: 처음 실행할 때 flume이 정상 동작하는지 로그를 보여줌. 정상 확인 되었으면 지워도됨.
        


# 참고
https://www.tutorialspoint.com/apache_flume/apache_flume_quick_guide.htm
