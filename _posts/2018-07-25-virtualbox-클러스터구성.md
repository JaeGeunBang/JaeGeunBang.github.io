---
layout: post
title: "virtual box - 클러스터 구성"
categories:
  - Posts
tags:
  - virtualbox
last_modified_at: 2018-07-25T12:57:42+09:00
---

개인 PC에서 elasticsearch, kafka 클러스터 구성을 해보기 위해 virtualbox를 이용했다. <br>

**환경**
```
OS: Ubuntu-16.04
Service: elasticsearch 2대
```
**VirtualBox 셋팅**
- Ubuntu 16.04 서버로 설치함 
- 각 머신의 "설정 - 네트워크"에 들어감
	- 어댑터 1 - NAT으로 설정
    	- 고급에 포트포워딩을 통해 호스트 PC에서 게스트 PC로 접속할 수 있음 (Rule 추가)
        	- ex. 호스트 IP 127.0.0.1, port: 10001, 게스트 IP 192.168.56.161, port: 22 (ssh를 위한 port)
            - 즉, 호스트 PC에서 127.0.0.1, port:10001로 접속하면 게스트 IP 192.168.56.161, 22로 접속할 수 있음
    - 어댑터 2 - 호스트 전용 어댑터로 설정
    	- 가상 머신들의 클러스터 구성을 위함
        - 파일 - 호스트 네트워크 관리자에 VirtualBox Host-Only Eternet Adapter가 있는데, IPv4 주소가 192.168.56.1, 서브넷 마스크 255.255.255.0 으로 설정되어 있을 것
        - 즉, 클러스터 구성을 원하는 가상 머신들의 IP를 192.168.56.X로 설정하면, 서로 같은 네트워크로 인식할 수 있음
        
<br><br>
**설치 후 Ubuntu 셋팅**
- 필요한 패키지 설치
	- JAVA
	- apt update && apt upgrade 등등
- 호스트 네임 바꾸기
	- hostnamectl set-hostname "이름" (호스트네임은 가상 머신별로 다르게 설정함. elasticsearch-1, elasticsearch-2)
- ssh 서버 설치
	- apt install openssh-server
    - ssh 서버 설치 후, 호스트 PC에서 게스트 PC로 ssh이 가능함 (위 포트포워딩을 했다면, putty등을 통해 접속 가능)
- 네트워크 IP 수동 설정
	- /etc/network/interfaces
    ```
    auto enp0s8
    iface enp0s8 inet static
    address 192.168.56.X (원하는 IP로 설정)
    ```
- 위 작업 후 재부팅
- elasticsearch는 디폴트로 9200, 9300 port로 통신하기 때문에, 각 가상 머신의 elasticsearch 서비스 실행 후, telnet을 통해 9200, 9300 port가 통신 중인지 확인함
