---
layout: post
title: "Capacity, Fair Scheduler 비교"
categories:
  - hadoop
  - yarn
  - scheduler
tags:
  - hive
last_modified_at: 2019-02-271T12:57:42+09:00
---



Yarn에서 제공하는 스케줄러인 Capacity, Fair 스케줄러에 대해 알아보았다.



## Capacity Scheduler

현재 Hortonworks에서 사용하는 스케줄러

클러스터의 지정된 가용량을 미리 할당 받는 방법으로, 최소한의 큐 가용성을 보장해 주는 컨셉으로 등장했다.

큐 내 스케줄링은 FIFO 방식만 제공한다.

![hadoop_1](https://user-images.githubusercontent.com/22383120/53460946-4805a600-3a82-11e9-8818-6f66c131a9ca.PNG)

각 큐에 Value값을 설정해 미리 가용량을 할당 받을 수 있다. 

물론 미리 가용량을 할당 받더라도 큐 탄력성을 통해 다른 큐의 자원을 가져다 쓸 수도 있고, 큐에 할당 받은 가용 자원이 없을 때 다른 큐의 컨테이너를 선점 (preemption) 할 수 있다.



## Fair Scheduler

현재 MapR, Cloudera에서 사용함

사용 중인 모든 잡에게 자원을 동적으로 배분하는 방법이다.

큐 내 스케줄링 방식은 균등 방식 (모든 잡에게 동등하게 자원을 분배) 뿐만 아니라 FIFO, DRF (우성 자원 방식) 등을 지원한다.

![hadoop_2](https://user-images.githubusercontent.com/22383120/53460984-5eabfd00-3a82-11e9-8b9a-139e2183603d.PNG)

각 큐에 Weight을 설정해 큐가 자원을 할당받을 가중치를 결정한다.

즉, Weight이 높은 큐는 다른 큐에 비해 더 많은 자원을 분배 받을 수 있다.

Capacity와 마찬가지로 평소에 다른 큐의 자원을 가져다 쓸 수 있고, 큐에 할당 받은 Weight 기준으로 다른 큐의 컨테이너를 선점 할 수 있다.



## 비교
처음 Capacity Scheduler는 선점 기능을 제공하지 못했던 걸로 알았는데, 최근 문서를 보니 선점 기능을 제공해주는 것으로 보인다. 

"하둡 완벽 가이드"에 보면 Capacity Scheduler 컨테이너를 선점하기 위해 강제로 죽이는 방법을 사용하지 않는다고 나온다.

허나 [공식문서 2.7.3](https://hadoop.apache.org/docs/r2.7.3/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html) 에 Capacity Scheduler에 들어가보면 선점 기능이 추가가 되어있다. ([공식문서 2.7.2](https://hadoop.apache.org/docs/r2.7.2/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html)에는 없음.)



즉, Fair Scheduler에서 큐 내 스케줄링 방식을 FIFO로 바꾸면 Capacity Scheduler와 크게 다른점은 없어보인다.

두 스케줄러 모두 `계층 큐`, `선점`, `딜레이 스케줄링`, `ACL 큐` 등을 모두 지원해 주기 때문이다. 

해당 문서([https://www.quora.com/On-what-basis-do-I-decide-between-Fair-and-Capacity-Scheduler-in-YARN](https://www.quora.com/On-what-basis-do-I-decide-between-Fair-and-Capacity-Scheduler-in-YARN)) 에 두 스케줄러를 비교한 글이 있었고, 이를 통해 Fair Scheduler가 좀 더 선호되는 것을 알게 되었다.



## 참고
[https://mapr.com/community/s/question/0D50L00006BItkWSAT/fair-scheduler-vs-capacity-scheduler](https://mapr.com/community/s/question/0D50L00006BItkWSAT/fair-scheduler-vs-capacity-scheduler)