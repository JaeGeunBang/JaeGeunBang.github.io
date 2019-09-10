---
layout: post
title: "Rack-Awareness 설정"
categories:
  - Posts
tags:
  - hadoop
  - rack-awareness
last_modified_at: 2019-02-21T12:57:42+09:00
---


하둡 Rack 설정을 하기위해 아래 참고 사이트를 참고한다.
[Hadoop 공식 홈페이지 - rack awareness](https://hadoop.apache.org/docs/r2.7.2/hadoop-project-dist/hadoop-common/RackAwareness.html#python_Example)

Rack 설정을 하기위해 core-site.xml 내 `net.topology.script.file.name` 속성에 아래 스크립트 파일을 값으로 입력해준다.

***etc/hadoop/core-site.xml***
```
<property>  
  <name>net.topology.script.file.name</name>  
  <value>$HADOOP_HOME/etc/hadoop/topology.py</value>  
</property>
```

***etc/hadoop/topology.py***
```
#!/usr/bin/python
# this script makes assumptions about the physical environment.
#  1) each rack is its own layer 3 network with a /24 subnet, which
# could be typical where each rack has its own
#     switch with uplinks to a central core router.
#
#             +-----------+
#             |core router|
#             +-----------+
#            /             \
#   +-----------+        +-----------+
#   |rack switch|        |rack switch|
#   +-----------+        +-----------+
#   | data node |        | data node |
#   +-----------+        +-----------+
#   | data node |        | data node |
#   +-----------+        +-----------+
#
# 2) topology script gets list of IP's as input, calculates network address, and prints '/network_address/ip'.

import netaddr
import sys
sys.argv.pop(0)                                                  # discard name of topology script from argv list as we just want IP addresses

netmask = '255.255.255.0'                                        # set netmask to what's being used in your environment.  The example uses a /24

for ip in sys.argv:                                              # loop over list of datanode IP's
address = '{0}/{1}'.format(ip, netmask)                      # format address string so it looks like 'ip/netmask' to make netaddr work
try:
   network_address = netaddr.IPNetwork(address).network     # calculate and print network address
   print "/{0}".format(network_address)
except:
   print "/rack-unknown"                                    # print catch-all value if unable to calculate network address
```
공식 홈페이지에서 제공해주는 파이썬 스크립트를 사용하면, 아래와 같이 출력된다.
```
/192.168.56.0
/192.168.56.0
/192.168.56.0
/192.168.56.0
/192.168.56.0
```
서브넷 ID 값이 공통으로 출력되며 모든 노드는 "/192.168.56.0" 이라는 랙으로 인식된다.
즉, 위 스크립트를 그대로 사용하면 모든 노드는 같은 랙에 있다고 설정된다.
각 노드의 랙 정보를 변경하기 위해 wiki에서 제공해주는 sh를 적용해보자.

[topology_rack_awareness_scripts](https://wiki.apache.org/hadoop/topology_rack_awareness_scripts)

***etc/hadoop/topology.sh***
```bash
HADOOP_CONF=/home/hadoop/etc/hadoop

while [ $# -gt 0 ] ; do
  nodeArg=$1
  exec< ${HADOOP_CONF}/topology.data 
  result="" 
  while read line ; do
    ar=( $line ) 
    if [ "${ar[0]}" = "$nodeArg" ] ; then
      result="${ar[1]}"
    fi
  done 
  shift 
  if [ -z "$result" ] ; then
    echo -n "/default/rack "
  else
    echo -n "$result "
  fi
done
```
쉘 스크립트에서 topology_tmp.data에서 입력받은 IP와 실제 노드의 IP를 대조하여 매칭 되는 rack을 출력한다.  <br>만약 노드 IP가 .data에 없다면 /default/rack을 출력한다.

***etc/hadoop/topology.data***
```
192.168.56.191   /dc_1/rack_1
192.168.56.192   /dc_1/rack_2
192.168.56.193   /dc_1/rack_2
192.168.56.194   /dc_1/rack_1
192.168.56.195   /dc_1/rack_1
```
모든 노드는 같은 데이터 센터 (dc_1)에 있다고 가정했고, rack은 1,2로 나누었다.
앞서 네임노드, 리소스 매니저의 HA 구성을 191, 192 노드로 했기 때문에 둘은 각 노드를 rack 1, 2로 나뉘었다.

rack에 대한 정보는 logs/에 네임노드 로그에서 확인할 수 있다.

```
2019-02-21 11:12:30,861 INFO org.apache.hadoop.net.NetworkTopology: Adding a new node: /dc_1/rack_1/192.168.56.191:50010
...
2019-02-21 11:12:30,917 INFO org.apache.hadoop.net.NetworkTopology: Adding a new node: /dc_1/rack_1/192.168.56.194:50010
...
2019-02-21 11:12:31,015 INFO org.apache.hadoop.net.NetworkTopology: Adding a new node: /dc_1/rack_2/192.168.56.193:50010
...
2019-02-21 11:12:31,056 INFO org.apache.hadoop.net.NetworkTopology: Adding a new node: /dc_1/rack_2/192.168.56.192:50010
...
2019-02-21 11:12:31,130 INFO org.apache.hadoop.net.NetworkTopology: Adding a new node: /dc_1/rack_1/192.168.56.195:50010
```

Yarn의 리소스 매니저가 동작중이면 Yarn UI에서도 확인이 가능하다.
> 192.168.56.191:8088 -> Cluster -> Nodes

## HDFS 확인
rack-awareness 설정 후 실제 HDFS에 데이터를 put하면 block이 다른 rack에 배치되는 것을 확인할 수 있다.
> bin/hdfs dfs -put LICENSE.txt /user/hadoop/

보통 리플리케이션이 3이라면 아래 순서로 복사본 배치를 진행한다.
1. 같은 노드
2. 다른 랙(무작위) 노드
3. 두번 째 랙과 같은 랙의 다른 노드

위 기본 전략은 신뢰성 (블록을 여러 랙에 저장), 쓰기 대역폭 (하나의 네트워크 스위치만 통과), 읽기 대역폭 (두 랙에서 가까운 렉을 선택), 클러스터 전반에 걸친 블록의 분산(클라이언트는 로컬 랙에 하나의 블록만 저장) 사이의 균형을 전체적으로 잘 맞추고 있다.
