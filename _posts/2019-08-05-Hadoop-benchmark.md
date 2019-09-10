---
layout: post
title: "Hadoop-benchmark"
categories:
  - Posts
tags:
  - hadoop
  - benchmark
last_modified_at: 2019-08-05T12:57:42+09:00
---


Hadoop을 위한 hardware가 구축되었고, software가 install되었을 때, 어플리케이션이 정상적으로 동작하는지 성능은 얼마나 나오는지 테스트를 해보자.



테스트는 bottom-up 방식으로 진행한다. 

1. disk, network 테스트를 수행해 합리적인 성능이 나오는지 진행함.
2. Hadoop의 구성 요소인 HDFS, YARN 테스트와 다른 hadoop 에코시스템인 impala, hbase, solr 등을 테스트함.
3. 실제 application을 테스트함.



### 테스트 주의사항

- 테스트는 최소 3번 진행하며, 그중 중간 값을 추출한다.

- 모든 테스트를 진행한 파라미터들과 그에 따른 결과를 기록한다.
- 같은 tool을 사용한다.



### 하드웨어 테스트

**DISK**

보통  DISK의  Sequential read, write의 성능은 약 100-150 MB/s 정도로 예측할 수 있다. DISK의 성능 측정은 `dd` 툴을 사용한다.

먼저 각 노드에서 df -h를 통해 마운트된 디스크에서 사용 가능한 디스크 공간과 마운트된 위치를 본다.

```
> df -h
Filesystem	Size	Used	Avail	Use%	Mounted on
/dev/sdk1	1.0T	...						/disk1
/dev/sdk2	1.0T	...						/disk2
/dev/sdk3	1.0T	...						/disk3
...
```



이후 dd툴을 통해 각 디스크의 성능을 측정한다. 

아래 명령은 각 마운트된 위치에 block이라는 파일을 쓴다. 128M 크기를 가진 파일을 10번 쓴다는 의미이다. 

명령을 수행 후 초당 몇 MB/s를 write했는지 알 수 있음.

```
dd if=/dev/zero bs=128M count=10 of=/disk1/ddtest/block
dd if=/dev/zero bs=128M count=10 of=/disk2/ddtest/block
dd if=/dev/zero bs=128M count=10 of=/disk3/ddtest/block
```



읽기 성능 테스트를 위해 위 input, output을 reverse함.

```
dd if=/disk1/ddtest/block bs=128M of=/dev/zero
dd if=/disk2/ddtest/block bs=128M of=/dev/zero
dd if=/disk3/ddtest/block bs=128M of=/dev/zero
```

결과를 보면 write 성능보다 read 성능이 더 빠른 것을 볼 수 있는데, 이는 읽을 때 캐시를 사용하기 때문이다. 

정확한 성능 측정을 위해 `dd`툴은 page cache 를 사용하지 않도록 관련 옵션을 제공한다. `iflag=direct`

```
dd if=/disk1/ddtest/block bs=128M of=/dev/zero iflag=direct
dd if=/disk2/ddtest/block bs=128M of=/dev/zero iflag=direct
dd if=/disk3/ddtest/block bs=128M of=/dev/zero iflag=direct
```

다시 측정해보면 결과가 아까보다 현저히 떨어지는 것을 볼 수 있다.



허나 실제 하둡은 여러 disk에 대해 Parallel 하게 read, write를 수행하기 때문에, parallel I/O을 측정해보는 것도 중요하다. 

parallel I/O 하게 측정하기 위해 위에서 사용한 dd 명령을 백그라운드 상에서 한번에 동작시켜본다. 

읽기를 위해 반대로도 진행해본다.

```
for n in $(seq 1 10); do
  dd if=/dev/zero bs=128M count=10 of=/disk${n}/ddtest/block 2>result.out
done
```

처음 테스트 주의사항에서도 말했듯이 여러번 테스트를 진행한 후 중간값을 기록하는 것이 좋다.



이외에도 `fio`라는 툴을 제공해주며, fio(https://github.com/axboe/fio)는 dd 처럼 sequantial read,write 뿐만 아니라 random read,write에 대한 성능 테스트도 진행이 가능해 더 복잡한 workload에 대해 고려할 수 있다. 

또한, `SMART`(https://en.wikipedia.org/wiki/S.M.A.R.T.)라는 디스크 self-reporing 기술을 제공하며, 이는 몇 에러를 보고하며 앞으로 발생할 disk 실패에 대한 정보를 제공해준다.



**NETWORK**

Network 테스트는 크게 두가지로 나뉜다. **Latency, Throughput (Bandwidth) 테스트**이다. 

Hadoop은 분산 환경에서 운영되는 시스템이다보니 Network 테스트는 중요하다. 

특히 Hadoop 시스템의 master, slave 노드는 서로 노드가 죽었는지 heartbeat를 보내야하는데 이를위해 Latency가 제대로 평가가 되어야 한다. 

먼저 같은 Rack에 있는 노드들 끼리 Latency 측정, 이후 서로 다른 Rack에 있는 노드들끼리 Latency 를 측정해야한다. 또한 Throughput은 테스트에 참여하는 모든 노드들이 동시에 반복적으로 진행되어야 한다.



**Latency 측정**

Latency는 `ping`을 통해 쉽게 측정할 수 있다. ping은 ICMP 프로토콜을 이용해 remote server에 echo request 패킷을 순차적으로 보내며, Latency를 측정해 결과를 제공한다.

보통 같은 Rack에 있는 머신끼리는 한자리 수의 ms 소요, 다른 Rack은 두자리수의 ms가 소요되며, 그 이후 latency는 점검이 필요하다.

```
> ping -c 10 slave02.node
> ping -c 10 slave03.node
```

각 slave노드로 10번의 packet을 전송하며, 평균 latency를 계산해준다.



**Throughput 측정**

Throughput 측정을 위해 `iperf3`(https://iperf.fr/) 이라는 툴을 사용한다. 

이 툴을 통해 throughput 테스트를 하기 위해 두 개 노드가 각 client, server 역할을 해야한다.



**서버노드**

```
> iperf3 -s -p 13000
```

13000 포트로 리스닝 한다.



**클라이언트노드**

```
> iperf3 -c slave01.node -p 13000 -t 10 -n 104857600
```

서버노드에 100MB에 데이터를 10번 보낸다는 의미이다. 이를 통해 Throughput을 측정할 수 있다.



**Intra, Inter Throughput**

Throughput은 Rack에 따라 Intra, Inter Throughput 측정 테스트가 필요하다.

Intra Throughput은 Rack 내 있는 모든 노드들 사이의 Network Throughput, Inter Throughput은 Rack 사이 노드들 끼리의 Network Throughput을 의미한다.



**Intra Throughput**

Rack 내 모든 노드들 끼리의 Throughput을 측정한다. 

트래픽을 점차 늘려 Top-of-rack switches가 Rack 내 모든 노드들의 데이터를 최대 bandwidth로 send, receive 할 수 있는지 측정한다.



**Inter Throughput**

Rack 간 노드들 끼리의 Throughput을 측정한다. 

Rack이 R개라고 한다면, Rack 간 커넥션은 처음에 R 이후에 2R, 3R ... 트래픽을 점차 늘리면서 Throughput을 측정한다. 

노드 간 연결이 점점 늘어날수록 측정된 Throughput들이 최대 bandwidth 가 되는 지점이 있을 것이다. 해당 지점을 influection point 라 한다.



예를들어, 네트워크 토폴로지의 oversubscription이 4:1, Rack은 5개, Rack 당 노드는 20개 일 경우, 동시에 50개 cross-rack connection을 할 수 있을 것으로 예측할 수 있다. 

그 후, 실제 Inter Throughput 측정을 통해 랙 간 노드들 사이 50개 동시 연결을 수행했을 때, 예측한 대로 성능 감소가 발생하는지 체크해보아야 한다. 

만약 그렇지 않다면 조사를 해야한다.



> 클라우데아에서 말하길 oversubscription은 1:1 비율이 가장 선호된다고 하며, 최대 4:1을 넘어서는 안된다고 한다. 
>
> 특히 Hadoop 3.0에서 사용하는 Erasure Coding 방식은 Rack 기반의 데이터 정책을 가지며 블록을 서로 다른 Rack에 고르게 분포하려한다. 
>
> 즉, oversubscription이 없거나 매우 작은 oversubscription 비율을 가진 환경에서 적용해야 해야 최상의 성능을 발휘할 수 있다.



## Hadoop (HDFS) 테스트

일단 하드웨어에 대한 테스트가 끝이났다면, 실제 하둡을 이루는 HDFS, Yarn/MapReduce에 대한 테스트가 필요하다. 

그 중 HDFS 테스트 방법을 진행한다. HDFS는 Hadoop 시스템 중 공통적으로 사용되기 컴포넌트이기 때문에 필수적으로 성능 테스트를 해봐야 한다.



**싱글 테스트**

단순하게 하나의 client로 HDFS에 data 를 write, read 테스트를 한다. 

이 테스트를 통해 HDFS에 잘못된 configuration을 찾을 수 있다. 테스트는 위에서 설명한 `dd` 툴을 통해 진행한다. 

아래는 hdfs에 데이터를 write했을 때의 성능을 구한다.

```
> dd if=/dev/urandom bs=128M count=10 | hdfs dfs -put - /tmp/hdfstest.out
```



아래는 read의 성능을 구한다.

```
> time hdfs dfs -cat /tmp/hdfstest.out > /dev/null
```

최소 Write는 35MB/s, 읽기는 70 MB/s에 성능이 측정되는 것을 확인해야 한다.



**분산 테스트**

HDFS는 거대한 데이터 덩어리를 분산 환경에서 동시에 read, write 수행하기 때문에 분산 테스트는 필수적으로 진행해야 한다. 

이를 위해 ` TestDFSIO` 툴을 사용한다. testDFSIO 는 MapReduce job이며, 동시에 I/O tasks들 진행해 read, write 성능을 측정할 수 있다. 

필요한 사이즈를 설정할 수 있으며, 압축도 제공한다.



```
> yarn jar [hadoop-mapreduce path]/hadoop-mapreduce-client-jboclient*-tests.jar TestDFSIO [genericOptions] -read [-random | -backward | -skip [-skipSize Size]] | -write | -append | -truncate | -clean [-compression codecClassName] [-nrFiles N] [-size Size[B|KB|MB|GB|TB]] [-resFile resultFileName] [-bufferSize Bytes] [-rootDir]
```

- 필요한 옵션은 아래와 같다.
  - read, write, append, truncat, clean 중 하나의 수행 옵션을 선택한다.
  - nrFiles N 설정을 통해 생상할 파일의 개수를 설정한다. 파일 당 control file이 하나씩 생성되고, 각 mapper로 수행됨.
  - size는 각 파일의 크기를 설정한다.
  - resFile은 테스트 결과를 저장할 경로를 설정한다.

```
> yarn jar ${hadoop_lib}/hadoop-mapreduce/hadoop-mapreduce-client-jobclient.jar \
TESTDFSIO \
-D test.build.data=/user/jgb710 \
-write \
-resFile w18_128 \
-nrFiles 6
-size 512MB
```

수행 결과 최종적으로 처리된 파일 수, Throughput, 총 처리된 파일 크기, 표준분산 등에 대한 결과를 제공한다. 

throughput은 `모든 매퍼에 의해 쓰여진 데이터 / 모든 매퍼에 의해 수행된 시간`을 바탕으로 계산된다. 

average는 `매퍼에 의해 보고된 모든 rate / 전체 매퍼 수`를 바탕으로 계산된다. 

여기서 봐야할 건 평균 I/O rate이 50MB/s가 초과하는지, 표준편차 값이 낮은지 확인한다.



## Genaral 테스트

Hadoop에서 일반적인 테스트를 할 때 많이 사용하는 툴은 TeraSort이다. 

TeraSort는 분산 환경에서 대규모 데이터를 sort 하는 application이다. TeraSort는 아래 3단계로 나뉜다.

- TeraGen
  - 명시된 수의 10-byte key, random value를 가지는 100-bytes의 record들을 생성한다.
  - 오로지 map 으로만 동작하며, 각 mapper는 records의 chunk를 hdfs에 쓴다.
- TeraSort
  - TeraGen에 의해 생성된 데이터를 읽어 Sorting 후 hdfs에 쓴다.
- TeraValidation
  - Sorting 된 결과를 읽어, Sorting의 정확도를 판단한다.

TeraSort는 실제 현실 세계에서 사용하는 workload가 아니기 때문에 성능을 최대화 하는 목적을 가지고 테스트할 필요가 없으며, 외부 결과와 비교할 필요도 없다. 

단지 TeraSort를 통해 Hadoop의 HDFS, Yarn, MapReduce 기능이 정상적으로 동작하는지, 이후 성능 비교 (I/O Hardware의 제한을 했을 때 얼마나 성능 감소가 되는지?, encryption 특징을 추가 했을 때 얼마나 성능 감소가 되는지?) 를 위해 Baseline 성능을 얻는 것을 목표로 한다.



**TeraGen**

TeraGen은 크게 Disk, Disk + Network 테스트를 할 수 있다. 

1. Disk 측정은 HDFS Replication을 1로 두어 Generate하는데 network 비용이 들지 않도록 막을 수 있다. 

2. HDFS Replication을 3으로 다시 두어 network 비용이 추가로 들었을 때의 성능을 측정할 수 있다. 3. 마지막으로 생성되는 데이터의 크기를 전체 클러스터 노드의 메모리보다 3배 이상으로 설정한다. 이를 통해 page cache에 영향으로 더 빨리 job이 수행되는 현상을 막을 수 있다.

mapper의 수는 전체 클러스터 노드의 disk 수와 동일하게 한다. 

만약 50개 노드가 있고 각 노드당 12개 DISK로 구성되어 있다면, 총 600 Mapper를 생성할 수 있도록 한다.



**TeraSort**

TeraSort는 HDFS Replication을 1, 3으로 각각 두어 테스트를 진행한다. 

이를 조정하기 위해 `mapreduce.terasort.output.replication` 파라미터를 이용한다.



## 참고

https://www.cloudera.com/documentation/other/reference-architecture/PDF/cloudera_ref_arch_metal.pdf