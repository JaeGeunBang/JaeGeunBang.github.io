---
layout: post
title: "Fair Scheduler 소개"
categories:
  - Posts
tags:
  - hadoop
  - yarn
  - scheduler
last_modified_at: 2019-03-01T12:57:42+09:00
---



Yarn에서 스케줄러란 노드매니저에게 제출된 Job들을 처리하기 위해, Job에게 Yarn이 관리하는 자원 (vCores, Memory)를 할당해 주는 기술이다.

가용할 수 있는 자원들을 최대한 활용하여 최대한 많은 Job들을 빠르게으로 처리하는데 목적을 둔다.

그 중 가장 많이 사용되는 Fair Scheduler에 대해 알아보자.



## Fair Scheduler

Fair Scheduler는 앞서 설명한것과 같이 사용중인 모든 잡에게 자원을 동적으로 배분하는 방법이다. 즉, 공평하게 모든 Job들에게 자원을 할당한다.

Fair Scheduler는 큐를 통해 자원을 공유한다. 큐에 할당된 Weight 값을 기준으로 Weight 값이 높은 큐가 상대적으로 많은 자원을 할당 받는다.

![hadoop_2](https://user-images.githubusercontent.com/22383120/53460984-5eabfd00-3a82-11e9-8b9a-139e2183603d.PNG)

위 Weight을 기준으로 계산해보면 A큐는 전체 자원의 30%, B 큐는 전체 자원의 50%, C 큐는 전체 자원의 20% 비율로 사용하게 된다. (클러스터 자원이 가득 찼을 경우) 

### 계층 큐
![hadoop_3](https://user-images.githubusercontent.com/22383120/53470827-23232a00-3aa6-11e9-81f5-d825ca630fa8.PNG)

위 그림 처럼 큐를 계층으로 관리할 수 있다. 

큐를 우선순위 순서대로 관리할 수 있으며, other 큐는 weight을 0을 줌으로써, priority 1,2 큐가 자원을 사용하지 않을 때만 각 other 큐에 할당되도록 설정할 수 있다.



### 큐 할당 정책
Job 제출했을 때, 해당 Job을 어느 큐에 할당할 것인지 설정할 수 있다. 
```
<queuePlacementPolicy>
  <Rule  name=“specified” create=“false“>
  <Rule  name=“primaryGroup” create=“false”>
  <Rule  name=“default” queue=“dev.eng”>
</queuePlacementPolicy>
```
위 예제를 통해 큐 할당 정책을 살펴보자. 

큐 할당 정책은 맨 위 Rule 부터 차례대로 적용한다. 즉, 맨 위 Rule이 적합하지 않다면, 그 다음 Rule로, 또 그 다음 Rule이 적합하지 않다면, 맨 마지막 Rule이 적용될 것이다.



여러 정책 중 몇가지를 살펴보자.

***specified***

> 어플리케이션 제출 시 명확하게 특정 큐를 적어준다.
> 
> hadoop jar mymr.jar app1 `-Dmapreduce.job.queuename="queue"`

***user***
> 어플리케이션 제출 시 제출한 계정의 이름을 가진 큐에 할당한다.
> 
> 만약 계정의 이름이 master라면, master라는 큐에 할당한다.

***primaryGroup***
> 어플리케이션 제출 시 제출한 계정의 그룹 이름을 가진 큐에 할당한다.
>
> 만약 계정의 그룹이 root라면, root라는 큐에 할당한다.

***nestedUserQueue***
> 어플리케이션 제출 시 제출한 계정의 큐를 생성하는데, root 큐 아래 생성하는 것이 아닌, 동봉된 rule에서 결정된 큐아래 생성한다.

```
<rule name="nestedUserQueue" create=”true”>
  <rule name="primaryGroup" create="false" />
</rule>
```

> 만약 계정의 이름이 test, 그룹이 master 라면, master 큐 아래 test 큐를 생성한다.

***default***
> 가장 기본적인 정책이며, 위 rule들이 모두 적용되지 않았을 경우 디폴트 큐로 할당해 주기 위해 사용한다.

***reject***
> 어플리케이션 제출이 거절된다. 전형적으로 맨 마지막 rule에 위치하며, default를 쓰지 않을 때 reject을 쓰면 된다.



### 선점 (Preemption)
다른 큐의 자원을 뺏는 기능이다. 즉, 스케줄러가 자원의 균등 공유에 위배되는 큐에 실행되는 컨테이너를 죽여 자원을 뺏어온다.

보통 같은 큐 내 있는 Job들 끼리는 자동으로 균등 분배가 발생하지만, 서로 다른 큐인 경우는 다르다. 

즉, 큐 간에는 자동으로 균등 분배가 발생하지 않는다. 

결국 내가 사용해야할 자원을 다른 큐가 사용하고 있다면, 해당 Job이 자원을 해제할 때 까지 기다려야 한다.

이러한 현상을 막기 위해 선점 기능을 제공한다. 선점을 사용하기 위해 아래 설정들을 제공한다.

> yarn.scheduler.fair.preemption
> 선점 기능 활성화 여부 (true, false)
> 
> fairSharePreemptionThreshold
> 큐가 사용할 자원의 임계치 (0~1)
> 
> fairSharePreemptionTimeout
> 큐가 사용할 자원의 임계치 만큼 자원을 사용하지 못하는 시간의 최대치 (second)



선점 기능이 활성화 되어 있어야 하며, Timeout 만큼 Threshold 만큼의 자원을 가져오지 못하면 선점이 발생한다. <br>

***예제)***
![hadoop_4](https://user-images.githubusercontent.com/22383120/53472701-bf9bfb00-3aab-11e9-9fb0-bf465fac481d.PNG)

현재 A 큐가 모든 자원을 사용중이라고 가정하자.

현재 B 큐는 선점 기능이 활성화 되어 있고, Threshold = 0.5, Timeout = 60으로 설정되어 있다.

이때 B 큐에 새로운 Job이 할당될 경우,

1. B 큐의 자원 사용량은 0%이므로, Threshold 보다 낮다. (50% > 0%)
2. 위 상황을 60초 동안 기다리며, 60초가 넘어가면 A큐의 자원을 선점한다.
3. B 큐의 Weight * 50% (Threshold) 만큼 자원을 선점한다.

![hadoop_5](https://user-images.githubusercontent.com/22383120/53472713-c88ccc80-3aab-11e9-96a4-7f797d833b65.PNG)

B 큐는 25%의 자원을 선점해 Job을 실행시킬 수 있다. 이어 A 큐에 Job이 자원을 하제하면 B 큐에게 다시 돌려준다.



## 참고
[https://blog.cloudera.com/blog/2015/09/untangling-apache-hadoop-yarn-part-1/](https://blog.cloudera.com/blog/2015/09/untangling-apache-hadoop-yarn-part-1/)

[http://blog.cloudera.com/blog/2015/10/untangling-apache-hadoop-yarn-part-2/](http://blog.cloudera.com/blog/2015/10/untangling-apache-hadoop-yarn-part-2/)

[http://blog.cloudera.com/blog/2016/01/untangling-apache-hadoop-yarn-part-3/](http://blog.cloudera.com/blog/2016/01/untangling-apache-hadoop-yarn-part-3/)

[http://blog.cloudera.com/blog/2016/06/untangling-apache-hadoop-yarn-part-4-fair-scheduler-queue-basics/](http://blog.cloudera.com/blog/2016/06/untangling-apache-hadoop-yarn-part-4-fair-scheduler-queue-basics/)

[http://blog.cloudera.com/blog/2017/02/untangling-apache-hadoop-yarn-part-5-using-fairscheduler-queue-properties/](http://blog.cloudera.com/blog/2017/02/untangling-apache-hadoop-yarn-part-5-using-fairscheduler-queue-properties/)