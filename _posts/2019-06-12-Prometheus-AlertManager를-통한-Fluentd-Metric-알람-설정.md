---
layout: post
title: "Prometheus-AlertManager를-통한-Fluentd-Metric-알람-설정"
categories:
  - Posts
tags:
  - fluentd
  - prometheus
last_modified_at: 2019-06-12T12:57:42+09:00
---



Fluentd와 Prometheus를 셋팅하면 아래와 같이 Prometheus Web brower에서 아래와 같은 Metric을 볼 수 있다.

![p_1](https://user-images.githubusercontent.com/22383120/59328835-5f625980-8d28-11e9-9148-702403843289.PNG)

위 metric을 활용하여 아래 Grafana와 연동할 수 있다.

![p_2](https://user-images.githubusercontent.com/22383120/59328841-61c4b380-8d28-11e9-8ffa-dddf8f1e9bc7.png)

하지만 서버 대수가 많아질 수록 Grafana를 통해 일일이 모든 지표를 확인하기가 쉽지 않다. 

그렇기 때문에 Prometheus Alertmanager를 통해 주요 지표에 대해서는 알람을 보내주는 것이 좋다.

간단하게 Fluentd 서버가 다운되었을 때 Slack, G-mail로 알람을 받아보자. Fluentd 의 다운 여부는 up 지표를 통해 알 수 있다.



### Prometheus Alertmanager

프로메테우스에서 수집한 메트릭에 특정 조건이 발생 시 알람을 전송하며, 알람 매니저는 전달받은 알람을 조작할 수 있으며, 목적지에 알람을 전송해준다. 

알람 조작을 통해 중복 알람 제거, 알람 그룹핑, 라우팅 등을 수행할 수 있다.



##### 특징

- 그룹핑
  - 비슷한 알람을 하나의 알람으로 카테고라이즈한다.
  - 이는 수 천대가 넘는 서버가 같은 알람을 보냈을 때, 하나의 알람만 보낼 수 있도록 해준다.
- 억제 (Inhibition)
  - 기존에 Critical한 알람이 발생했었다면, 이 후에 발생한 덜 중요한 (warning 같은거) 알람은 mute할 수 있다.
- 정적 (Slience)
  - 주어진 시간 동안 알람을 발생시키지 않는다.
  - slience는 알람 매니저 Web Interface에서 설정할 수 있다.
- HA 구성
  - 알람 매니저의 클러스터를 만들어 하나의 알람 매니저가 다운되더라도 서비스를 계속 운영할 수 있다.



##### 설치

공식 홈페이지에서 알람 매니저를 다운받고 압축을 풀면, 아래와 같이 구성되어 있다.

```
alertmanager
alertmanager.yml
amtool
data
LICENSE
NOTICE
```

alertmanager.yml는 아래와 같이 설정한다.

```yaml
global:
  slack_api_url: 'https://hooks.slack.com/services/xxxx'
  smtp_require_tls: false

route:
  receiver: 'default-receiver'
  group_by: [alertname, instance, type]
  group_wait: 30s 
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - receiver: 'email-notifications'
    group_by: [severity, job]
    continue: true
  - receiver: 'slack-notifications'
    continue: true

receivers:
- name: 'default-receiver'
- name: 'slack-notifications'
  slack_configs:
  - channel: '#channel'
    text: 'test'
- name: 'email-notifications'
  email_configs:
  - to: 'receiver_1@xxx.com'
    from: 'sender_1@yyy.com'
  email_configs:
  - to: 'receiver_2@xxx.com'
    from: 'sender_2@yyy.com'
    
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: [alertname, instance, type]
```

`global`은 공용으로 쓰이는 parameter들을 정의한다.

- `slack_api_url` 은 slack api url을 적는다.
- `smtp_require_tls`는 false로 하여 smtp 전송을 tls로 하지 않지 않겠다고 설정한다.

`route`은 routing tree를 정의한다. 알람이 발생하면 정의한 routing tree의 루트부터 시작해 자식 트리를 쫒아 내려가면서 조건에 맞는 라우팅을 찾아 알람을 보낸다.

- 여기서 root는 default-receiver, child는 slack-notifications, email-notification이 된다.
- root에서 설정한 값들은 child에게도 모두 공통 적용된다.
  - `group_by`은 group으로 묶을 label 이름을 적는다. [alertname, instance, type] 은 각 "InstanceDown", "ip:port", "fluentd" 이며, 세 label이 똑같은 알람들은 grouping 된다.
  - `group_wait`은 알람이 처음 발생했을 때 기다릴 시간이다. 기본 값은 30초 이다.
  - `group_interval` 은 최초 알람 발생 후 똑같은 group에 새로운 알람이 발생했을 때 기다릴 시간이다. 기본 값은 5분이다.
    - 만약 group이 "fluentd"일 경우, alertname이 "InstanceDown" 알람이 발생 했으며, 이후 group은 똑같고 alertname이 "InstanceUp" 인 알람은 5분 뒤에 보낸다는 의미이다.
  - `repeat_interval`은 똑같은 알람이 발생했을 때 기다릴 시간이다. 기본 값은 4시간이다.
    - 만약 group이 "fluentd"이고, alertname이 "InstanceDown" 알람이 발생 했는데, 이후 완전 똑같은 알람이 발생하면 이는 4시간 뒤에 보낸다는 의미이다.
  - `receiver`알람을 보낼 곳의 목록이다.
    - `continue`는 같은 형제 receiver에 알람을 이어서 보낸다는 의미이다. false로 되어있다면, slack에 알람을 보낸 후 email엔 알람을 보내지 않는다. 기본 값은 false이다.

`receivers`는 알람을 보낼 곳을 좀 더 자세히 정의한다.

- 여기서 각 `slack_config`와 `email_config`를 정의하고 필요한 값을 채운다.

`inhibit_rule`는 알람이 source에 매칭된 상태에서, 이 후 target에 매칭되었다면 해당 알람은 무시할 수 있다. 



slack은 Incoming WebHook을 설치 후 생성된 URL을 기재하면 된다.

<https://sktelecom-oslab.github.io/Virtualization-Software-Lab/PrometheusAlert/>



gmail은 아래 링크를 참고한다. security 전송을 하지 않을 거라면 굳이 AUTH_TOKEN을 설정하지 않아도 된다.

<https://www.robustperception.io/sending-email-with-the-alertmanager-via-gmail>



##### 알람 규칙

알람 규칙은 설치한 prometheus 폴더에 rule.yml 만들어 정의한다.

rule.yml

```yaml
groups:
- name: Instances
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 10s 
    labels:
      severity: page
      type: in_out
    annotations:
      description: '{{ $labels.instance }} of test'
```

`expr`은 metric의 규칙을 적는다. 여기선 up이 0이 되었을 때 알람을 발생 시킨다. 

`for` 은 해당 metric을 10초동안 기다린 후 알람을 전송하겠다는 의미이다.

`labels`는 이후 알람 grouping에 사용 할 labels들을 기재한다. 기본 label은 alertname (InstanceDown), instance(IP:PORT), job(fluentd)이 있다.

`annotations`은 해당 알람에 추가 정보를 입력한다.



해당 파일을 생성 후 prometheus.yml에 `rule_files:` 에 해당 파일을 추가해준다.

```yaml
rule_files:
  - "rule.yml"
```



이 후 프로메테우스, 프로메테우스 알람매니저를 실행하며, fluentd를 실제 stop 하면 slack, g-mail에 알람이 오는 것을 확인할 수 있다. 

확인을 빠르게 하고 싶다면 `group_wait`, `group_interval`, `  repeat_interval`을 짧게 설정한다.



![p_3](https://user-images.githubusercontent.com/22383120/59331828-d0f1d600-8d2f-11e9-8dc4-56f9e5a6d5be.PNG)



![p_4](https://user-images.githubusercontent.com/22383120/59331992-39d94e00-8d30-11e9-86d9-c8f834fcb8b9.PNG)





### 필요한 Metric의 Alert 설정

![1](https://user-images.githubusercontent.com/22383120/61356099-95a77180-a8b0-11e9-90af-d86966f7e5d5.png)
![2](https://user-images.githubusercontent.com/22383120/61356104-96d89e80-a8b0-11e9-860a-aa8c90b9f55f.png)



fluentd metric 수집을 위해 다음 4개의 알람을 설정했다.

1. BufferLengthChecking

   - Buffer 크기를 체크한다. 모니터링 결과 Buffer 크기가 보통은 1에서 머물며, 만약 2이상 넘어가는 시간이 10분이 초과하면 알람을 받게 된다.

   - 표현식

     > max(max_over_time(fluentd_output_status_buffer_queue_length[1m])) by (hostname) > 1

     - 1분 동안 최대 버퍼 크기를 hostname 별로 측정한다.

2. FluentdDown

   - Fluentd service가 down 됐을 경우 알람을 받게 된다.

   - 표현식

     > up == 0

     - 30초 동안 down 상태를 측정한다.

3. LargeLogCountChecking

   - record 의 수가 0일 경우 알람을 받게된다. 즉, 이는 record가 일시적으로 들어오지 않는 상태를 의미한다.

   - 표현식

     > rate(fluentd_output_status_num_records_total[1m]) == 0

     - 1분 동안 records의 수의 평균을 측정한다.

4. RetryCountChecking

   - retry 의 수가 1 이상일 경우 알람을 받게 된다. 즉, 네트워크 이슈나 목적지 서버의 문제로 네트워크 재시도가 발생했을때를 의미한다.

   - 표현식

     > max(rate(fluentd_output_statis_retry_count[1m])) by (hostname) > 0

     - 1분 동안 retry 평균 카운트 수의 최대 값을 hostname 별로 측정한다.



#### 참고

<https://prometheus.io/docs/alerting/configuration/>