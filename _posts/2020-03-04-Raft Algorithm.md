---
layout: post
title: "Raft Algorithm"
categories:
  - Posts
tags:
  - raft
  - consensus
last_modified_at: 2020-03-04T12:57:42+09:00
---



## Raft Algorithm



Raft Algorithm은 분산 시스템에서 노드 간의 상태를 공유하는 알고리즘을 뜻함. 대표적으로 Paxos, Zab (Zookeeper)가 있으나 어려워, 이를 이해하기 쉽게 구현하기 위해 설계되었음.



> Raft is a protocol for implementing distributed consensus.



**동작과정**

노드는 크게 3가지 형태를 가진다.

- Leader: 분산 시스템의 상태를 결정하는 노드
- Follower: Leader를 따르는 노드
- Candidate: Leader가 되기 전 후보자들. 후보자들 중 vote가 많은 노드가 Leader가 된다.



**Leader Election**

초기 노드는 모두 follower 상태에서 시작한다.

![1](https://user-images.githubusercontent.com/22383120/75855977-bcfa8100-5e36-11ea-8b76-17991f40f84c.PNG)



이중 leader로 부터 응답을 받지 못하면 Candidate 상태로 바뀐다.

![2](https://user-images.githubusercontent.com/22383120/75855978-bd931780-5e36-11ea-94ca-70b6ff12746c.PNG)



Candidate 상태로 변한 노드는 vote를 다른 노드에 요청하면, 이에 대한 reply을 받는다. 

대다수의 node로 부터 reply를 받은 note가 leader로 선출된다.

![3](https://user-images.githubusercontent.com/22383120/75855981-be2bae00-5e36-11ea-8b2f-c5911a5423d1.PNG)



**Log Replication**

이후 시스템의 모든 변화는 leader를 통해 이루어 진다.

Client가 5라는 값을 보내면 먼저 leader에게 전달되며, 변화가 발생하는데 이러한 변화는 node's log (`log entity`)에 기록된다.

log entity는 uncommit 한 상태이며, 이는 다른 node의 value가 update 되지 않았다.

![4](https://user-images.githubusercontent.com/22383120/75855983-be2bae00-5e36-11ea-9612-64515a493e97.PNG)



commit한 상태로 만들기 위해, 다른 node (follower)의 value를 전달해 update 하며, 이를 leader에게 reply 한다.



이후 leader는 follower들에게 log entity가 commit 됐다고 알려준다.



**Leader Election (DEEP)**

election을 control 하기 위해 두 종류의 timeout을 setting 할 수 있다.



*election timeout*

follower가 candidate가 될때 까지 기다리는 시간을 의미한다.

시간은 random으로 선정되며 150ms ~ 300ms사이로 결정된다. 즉, 노드마다 election timeout 값이 다르다.



아래 그림 처럼 맨 위 노드가 random 시간이 가장 짧아 가장 먼저 candidate가 되었다.

![6](https://user-images.githubusercontent.com/22383120/75855987-bec44480-5e36-11ea-9a3e-469164d437df.PNG)



가장 먼저 candidate가 된 노드가 다른 노드들에게 Request Vote를 보내며, 이후 vote를 가장 많이 받은 node가 leader로 선출된다. 

- candidate는 여러 노드가 될 수 있기 때문에 vote를 가장 많이 받아야 leader가 된다.
- 여러 candidate 노드가 같은 vote를 받았다면, 이는 다시 선출을 진행한다. leader가 뽑힐때까지..

![7](https://user-images.githubusercontent.com/22383120/75855988-bec44480-5e36-11ea-8a2d-102a4d71b501.PNG)



*heartbeat timeout*

leader가 된 노드는 follower들에게 heartbeat를 주기적으로 보내는 시간을 의미한다. 만약 시간 내 heartbeat를 받지 못했다면 해당 node들은 candidate 상태가 된다.



![8](https://user-images.githubusercontent.com/22383120/75855989-bf5cdb00-5e36-11ea-9b5e-ff10cdfa9335.PNG)

만약 leader가 dead가 되어 heartbeat를 더이상 보내지 못한다면, 나머지 follower 중 leader가 다시 선출된다.



**Log Replication (DEEP)**

Leader는 follower들에게 주기적으로 heartbeat를 보내는데, 만약 Client가 Leader에게 특정 값을 보내면 heartbeat 이후에 수정된 값을 follower들에게 보낸다.



![9](https://user-images.githubusercontent.com/22383120/75855992-bf5cdb00-5e36-11ea-8a63-80b67e0a7dbd.PNG)

이후 모든 follower들에게 ack 를 받고 난 후 commit이 되며, commit 된 이후에 client에게 response를 보낸다.



Raft는 network partition 의 상태에서도 consistent를 유지할 수 있다.

![10](https://user-images.githubusercontent.com/22383120/75855994-bff57180-5e36-11ea-9895-a8986bd56f93.PNG)

위 그림처럼 A, B와 C, D, E가 분리가 되었을때 Leader가 두 노드가 되는것을 볼 수 있다.



![11](https://user-images.githubusercontent.com/22383120/75855995-bff57180-5e36-11ea-924c-b1a829bb5c08.PNG)

이후 Clinet가 둘이 각 Network에 value를 보내는데, 아래 Client는 3을 보낸다.

허나 3은 commit이 되지 않는데 이러한 이유는 과반수를 넘지 못했기 때문이다. (빨간색 표시)



위 Client는 8을 보내는데, 8은 commit이 된다. 과반수를 넘었기 때문이다. (검은색 표시)

![12](https://user-images.githubusercontent.com/22383120/75855976-bc61ea80-5e36-11ea-873c-3bd883f46c64.PNG)

network partition이 정상적으로 돌아오면, 이미 commit 된 노드의 값을 가지는 `8`로 update가 되며, Leader는 위 Network의 노드가 그대로 Leader를 유지한다.



### 참고

http://thesecretlivesofdata.com/raft/

