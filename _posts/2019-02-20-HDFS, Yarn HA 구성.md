---
layout: post
title: "HDFS, Yarn HA 구성"
categories:
  - Posts
tags:
  - hadoop
  - yarn
  - zookeeper
last_modified_at: 2019-02-20T12:57:42+09:00
---



HDFS, Yarn의 네임노드와 리소스매니저는 단일 장애점(SPOF)이기 때문에 HA 구성을 해야한다.

HA 구성을 위해 주키퍼 설치가 필요하다.

앞에 Hadoop 셋팅을 이어서 진행해본다.



## 전체 구성

**Hadoop-1 (192.168.56.191)**

`NameNode, ResourceManager, Zookeeper, JournalNode, DataNode, DFSZKFailoverController, NodeManager`



**Hadoop-2 (192.168.56.192)**

`NameNode, ResourceManager, Zookeeper, JournalNode, DataNode, DFSZKFailoverController, NodeManager `



**Hadoop-3 (192.168.56.193)**

`JournalNode, Zookeeper, DataNode, NodeManager`



**Hadoop-4 (192.168.56.194)**

`DataNode, NodeManager`



**Hadoop-5 (192.168.56.195)**

`DataNode, NodeManager`




네임노드와 리소스매니저는 Hadoop 1,2 서버에서 서비스가 동작한다.

`Active, Standby`



저널노드와 주키퍼는 Hadoop 1,2,3 서버에서 서비스가 동작한다.

데이터노드, 노드매니저는 모든 노드에서 서비스가 동작한다.



### 주키퍼 설치

주키퍼 홈페이지에서 최신 버전의 주키퍼를 설치한다. 

설치 후 아래 conf/ 폴더에 zoo_sample.cfg 파일을 zoo.cfg로 변경하고 내용을 추가한다.

```conf
cd /home/zookeeper
cp conf/zoo_sample.cfg conf/zoo.cfg
vi conf/zoo.cfg
```


***zoo.cfg***

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/zookeeper/data
clientPort=2181
maxClientCnxns=0
maxSessionTimeout=180000

server.1=192.168.56.191:2888:3888
server.2=192.168.56.192:2888:3888
server.3=192.168.56.193:2888:3888
```
> dataDir (주키퍼 스냅샷을 저장하는 경로이다)

> maxClientCnxnx (최대 클라리언트 커넥션 수이다. 0은 무제한을 의미하며 기본값은 60이다)

> maxSessionTimeout (세션별 타임아웃 시간이며 밀리초 단위로 설정한다)

> server.#=hostname:2888:3888 (#에 서버 번호가 들어가며 서버 번호는 ${dataDir}/myid로 저장해야 한다. 첫번째 포트인 2888은 주키퍼 서버에 접속하기 위한 포트, 두번째 포트인 3888은 주키퍼 서버들 끼리 리더를 선출하기 위해 사용함.)



설정 후 /home/zookeeper/data/myid를 만들어 각 서버 id 값을 부여한다. 

서버 id는 주키퍼 서버 마다 위에서 할당한 번호를 입력한다.

```
vi /home/zookeeper/data/myid
1 or 2 or 3 입력
```


아래 명령을 통해 주키퍼를 실행할 수 있으며 서버 상태( 해당 주키퍼 서버가 leader인지 follower 인지)를 확인할 수 있다.

```
./bin/zkServer.sh start
./bin/zkServer.sh status
```



### 하둡 셋팅

***etc/hadoop/slave***
모든 노드가 데이터노드 역할을 하기 때문에, 모든 노드를 써준다.

```
192.168.56.191
192.168.56.192
192.168.56.193
192.168.56.194
192.168.56.195
```



***etc/hadoop/core-site.xml***

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop-cluster</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/tmp</value>
  </property>
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>192.168.56.191:2181,192.168.56.192:2181,192.168.56.193:2181</value>
  </property>
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/jaegeun/.ssh/id_rsa</value>
  </property>
</configuration>
```
> fs.defaultFS: hdfs 네임노드의 호스트를 쓴다. 여기서 네임노드는 두 개 이상이기 때문에 hadoop-cluster라는 이름으로 설정하고, 이는 hdfs-site.xml에서 자세히 정의한다.

> ha.zookeeper.quorum: 주키퍼 서버 호스트와 포트번호를 쓴다. (주키퍼 서버 호스트:포트번호)

> dfs.ha.fencing.methods: active 네임 노드가 두개 동작하여 발생하는 스플릿브레인 현상을 막기 위해, 하나의 active 네임 노드만 동작할 수 있도록 fencing메서드를 쓴다.



***etc/hadoop/hdfs-site.xml***

```
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/home/hadoop/dfs/name</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/home/hadoop/dfs/data</value>
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/home/hadoop/dfs/journalnode</value>
  </property>
  <property>
    <name>dfs.nameservices</name>
    <value>hadoop-cluster</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.hadoop-cluster</name>
    <value>nn1,nn2</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.hadoop-cluster.nn1</name>
    <value>192.168.56.191:8020</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.hadoop-cluster.nn2</name>
    <value>192.168.56.192:8020</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.hadoop-cluster.nn1</name>
    <value>192.168.56.191:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.hadoop-cluster.nn2</name>
    <value>192.168.56.192:50070</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://192.168.56.191:8485;192.168.56.192:8485;192.168.56.193:8485/hadoop-cluster</value>
  </property>
  <property>
    <name>dfs.client.failover.proxy.provider.hadoop-cluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
</configuration>
```
> dfs.nameservices: hdfs 네임스페이스 이름을 쓴다. 앞서 core-site.xml의 fs.defaultFS에 적었던 이름과 동일해야한다. (hadoop-cluster)

> dfs.ha.namenodes.`hadoop-cluster`: 각 네임노드의 명칭을 쓴다. 뒤에 `hadoop-cluster`는 dfs.nameservices에 적은 hdfs 네임스페이스를 써야한다. (nn1, nn2)

> dfs.namenode.rpc-address.`hadoop-cluster`.`nn1`: 네임 노드의 명칭을 추가로 쓰며, RPC를 위해 네임노드 호스트와 포트번호를 쓴다.

> dfs.namenode.http-address.`hadoop-cluster`.`nn1`: 네임 노드의 명칭을 추가로 쓰며, http를 위해 네임노드 호스트와 포트번호를 쓴다.

> dfs.namenode.shared.edits.dir: 에디트로그를 공유하기 위한 저널로그 서버를 쓴다. 저널로그는 hadoop1,2,3 서버에서 동작시킨다. (qjournal://hostname1:8485;hostname2:8485:hostname3:8485/{dfs.nameservices})

> dfs.ha.automatic-failover.enabled: failover를 자동화 하기 위해 true로 해준다. active 네임노드가 죽으면, standby 네임노드가 failover controller에 의해 자동으로 active 노드가 된다.



***etc/hadoop/yarn-site.xml***

```
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
    <name>yarn.nodemanager.local-dirs</name>
    <value>/home/yarn/nm-local-dir</value>
  </property>
  <property>
    <name>yarn.resourcemanager.fs.state-store.uri</name>
    <value>/home/yarn/system/rmstore</value>
  </property>
  <property>
    <name>yarn.resourcemanager.recovery.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
  </property>
  <property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>192.168.56.191:2181,192.168.56.192:2181,192.168.56.193:2181</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>cluster1</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>192.168.56.191</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>192.168.56.192</value>
  </property>
```
> yarn.resourcemanager.recovery.enabled: 리소스 매니저 시작히 state 복구 여부이며, true로 한다.

> yarn.resourcemanager.store.class: 리소스 매니저의 상태 정보저장을 위해 주키퍼 저장소를 사용한다.

> yarn.resourcemanager.zk-address: 주키퍼 서버 정보를 쓴다.

> yarn.resourcemanager.ha.enabled: 리소스매니저의 HA 사용 여부이며, true로 한다.

> yarn.resourcemanager.cluster-id: 리소스 매니저의 클러스터 이름을 지정한다.

> yarn.resourcemanager.ha.rm-ids: 리소스 매니저 이름을 지정한다. (rm1, rm2)

> yarn.resourcemanager.hostname.`rm1`: 각 리소스 매니저의 호스트를 쓴다.



모든 하둡 서버로 위 설정 파일을 복사한다.



### 하둡 실행

하둡은 아래 순서로 실행 시킨다.
1. Zookeeper Failover Controller 실행을 위해 초기화를 진행한다.
> hadoop-1: bin/hdfs zkfc -formatZK




2. 저널 노드를 실행한다.
> hadoop-1: sbin/hadoop-daemon.sh start journalnode
>
> 
>
> hadoop-2: sbin/hadoop-daemon.sh start journalnode
>
> 
>
> hadoop-3: sbin/hadoop-daemon.sh start journalnode




3. 저널 노드가 준비 되었다면, 네임 노드를 포맷 후 실행한다. (active 네임노드)
> hadoop-1: bin/hdfs namenode -format
>
> 
>
> hadoop-1: sbin/hadoop-daemon.sh start namenode




4. standby 네임노드를 실행한다.
> hadoop-2: bin/hdfs namenode -bootstrapStandby
>
> 
>
> hadoop-2: sbin/hadoop-daemon.sh start namenode




5. 나머지 데이터 노드, ZKFC를 실행한다.
> hadoop-1: sbin/start-dfs.sh



이미 네임노드와 저널노드는 실행중이기 때문에 running.. 중이라고 뜬다. 나머지 데이터 노드와 ZKFC가 정상적으로 시작됐음을 본다.

각 서버에서 필요한 서비스가 떠있는지 확인한다. (jps)



**Hadoop-1 (192.168.56.191)**

`NameNode, Zookeeper, JournalNode, DataNode, DFSZKFailoverController`



**Hadoop-2 (192.168.56.192)**

`NameNode, Zookeeper, JournalNode, DataNode, DFSZKFailoverController`



**Hadoop-3 (192.168.56.193)**

`JournalNode, Zookeeper, DataNode`



**Hadoop-4 (192.168.56.194)**

`DataNode`



**Hadoop-5 (192.168.56.195)**

`DataNode`



Hadoop-1, 2에 네임노드가 구동중이다. 

여기서 active 네임노드가 죽을 경우, standby 네임노드가 active로 전환되는데 이는 DFSZKFailoverController가 수행한다. 

각 네임노드의 active, standby를 확인하기 위해 `192.168.56.191:50070`, `192.168.56.192:50070` 에 접속해보면 확인해볼 수 있다. 

또한 아래 명령을 통해 확인하 수 있다.

> hadoop1: bin/hdfs haadmin -getServiceState rm1 (=active)
>
> 
>
> hadoop1: bin/hdfs haadmin -getServiceState rm2 (=standby)



Hadoop-1,2,3에 저널노드와 주키퍼가 구동중이다. 

저널노드는 네임 스페이스 이미지, 에디트 로그를 공유하고 있어 active 네임노드가 죽어도 standby 네임노드가 바로 active 네임노드로 구동이 가능하다.



### Yarn 실행

1. 아래 명령을 통해 Yarn을 실행한다. (active 리소스 매니저)
> hadoop-1: sbin/start-yarn.sh




2. standby 리소스 매니저 실행한다.
>  hadoop-2: sbin/yarn-daemon.sh start resourcemanager




3. 상태 확인을 통해 리소스 매니저의 active, standby 상태를 확인할 수 있다.
> hadoop1: bin/yarn rmadmin -getServiceState rm1 (=active)
>
> 
>
> hadoop1: bin/yarn rmadmin -getServiceState rm2 (=standby)



Yarn의 리소스 매니저는 DFSZKFailoverController가 내장되어 있어서, HDFS 처럼 따로 서비스를 띄울 필요가 없다. Yarn의 상태 정보는 주키퍼에서 저장/관리 한다.



#### 최종 확인

**Hadoop-1 (192.168.56.191)**

`NameNode, ResourceManager, Zookeeper, JournalNode, DataNode, DFSZKFailoverController, NodeManager`



**Hadoop-2 (192.168.56.192)**

`NameNode, ResourceManager, Zookeeper, JournalNode, DataNode, DFSZKFailoverController, NodeManager `



**Hadoop-3 (192.168.56.193)**

`JournalNode, Zookeeper, DataNode, NodeManager`



**Hadoop-4 (192.168.56.194)**

`DataNode, NodeManager`



**Hadoop-5 (192.168.56.195)**

`DataNode, NodeManager`




failover 테스트를 위해 kill 명령을 통해 네임노드, 리소스매니저를 죽여보자.
```
kill -9 pid
```



## 참고
[Yarn HA 구성](https://bloodguy.tistory.com/entry/Hadoop-YARN-ResourceManager-HA-HighAvailability) <br>
[HDFS HA 구성](https://excelsior-cjh.tistory.com/73) <br>
[Hadoop 공식 홈페이지](https://hadoop.apache.org/)
