---
title: Ambari Agent Image 만들기
author: Moon
date: 2020-11-17 21:21:00 +0700
categories: [Hadoop]
tags: [Hadoop, Ambari, Docker]
math: true
preview: Ambari Hadoop cluster on local, 세번째 Ambari agent image 만들었던 과정
---
# 목적
- 개인적인 목적의 Spark App개발과 새로운 Hadoop 기술 테스트 베드를 목적으로 하는, 로컬 클러스터를 구성해보고자 함
- Hadoop의 Pseudo Distribution 모드가 아닌 Full distribution으로 구성해보고자 함
- CentOS base의 여러 Docker 컨테이너를 띄워 각각이 Cluster의 Node가 될 예정


# 사용된 주요 기술
- [Apache Ambari](https://ambari.apache.org/)
- [Docker](https://www.docker.com/)
- [Hadoop](https://hadoop.apache.org/)
- [Hive](https://hive.apache.org/)
- [Kafka](https://kafka.apache.org/)  
  

# 시작 전 준비되야 할 사항들
- [Base Image 구축 과정](./2020-11-15-struggling-with-ambari.md)을 보고 오면 좋음


# 본문(Server image 만들기)

[이전 과정](./2020-11-15-struggling-with-ambari.md)에서 이미 필요한 모든 의존성과 잡다한 설정이 거의 완료된 상태이기 때문에, 크게 어려울 게 없다. 참고로 Base image는 내 [Docker hub Repository](https://hub.docker.com/u/devmoonduck)에 Push되었고, [`devmoonduck/ambari-local-base`](https://hub.docker.com/r/devmoonduck/ambari-local-base)를 사용하면 된다.

## Base image를 기반으로 Dockerfile 생성
```dockerfile
FROM devmoonduck/ambari-local-base:2.7.4.0
```
특별히 설명할 건 없다. 다음 단계로 넘어가자.  

## ambari agent 설치
[이전 단계](./2020-11-15-struggling-with-ambari.md)에서 이미 Hortonworks Repo를 추가했기 때문에 간단하게 yum으로 설치가 가능하다.
```dockerfile
RUN yum install -y ambari-agent
```

## service 커맨드가 동작하도록 의존성 설치
Ambari server에서 각 Agent에 설치된 서비스를 실행할 때 아래와 같은 커맨드를 날리므로
```bash
service chronyd start
```
service 커맨드가 동작하도록 추가 의존성이 필요하다.
```dockerfile
RUN yum install -y initscripts
```

## 설정파일 지정 위치에 두기
Ambari agent가 이전에 구축했던 Ambari Server를 보도록 [미리 설정한 파일](https://github.com/dev-m00n/ambari-local-agent/tree/master/ambari-conf)을 지정된 위치에 두어야한다.
```dockerfile
COPY ambari-conf/ambari-agent.ini /etc/ambari-agent/conf/
```
주요 설정을 보면 다음과 같다.
```ini
hostname=ambari.server.local
...
```

## ssh key 생성
```dockerfile
RUN ssh-keygen -t rsa -q -f "/root/.ssh/id_rsa" -N ""
```
Ambari server에서 비밀번호 묻지도 따지지도 않고 접근을 허용하도록 Ambari server의 public key를 `autorized_keys`에 등록해준다. [이전 단계](./2020-11-16-struggling-with-ambari2.md')에서 우리는 미리 생성해 놓은 key를 사용하기로 했고 이 key파일은 내 github repository에 올려두었다.
```dockerfile
RUN curl https://raw.githubusercontent.com/dev-m00n/ambari-local-server/master/id_rsa.pub -o /root/.ssh/authorized_keys
```
이제 Ambari server는 각 agent에 비밀번호 입력없이 `ssh {agent-host}`로 접근할 수 있게 된다.

## rpcbind 설치
 Ambari를 통해 모드 컴포넌트들을 설치한 후 서비스를 실행시키면 일부가 실패하는데 이는 `rpcbind` using `systemctl start rpcbind` 가 실패하기 떄문이다. 따라서 이 의존성 역시 설치되어야 한다.
 ```dockerfile
 RUN yum install -y rpcbind
 ```
 그러나 여기엔 문제점이 있는데, rpcbind를 설치하면, systemd의존성을 업데이트 해버려서 `/usr/bin/systemctl` 역시 날라가버린다. 그래서 이전 base image구축할 때 사용했던 custom systemctl을 안전한 곳에 놓아두고 `rpcbind`를 설치한 후 다시 원래 자리에 두기로 한다.
 ```dockerfile
RUN mv /usr/bin/systemctl /tmp/ && yum install -y rpcbind && mv /tmp/systemctl /usr/bin/
 ```

완성된 Dockerfile은 아래와 같다.(몇가지 더 넣긴했지만 없어도 문제가 없는 작업들이다)
```dockerfile
FROM devmoonduck/ambari-local-base:2.7.4.0

LABEL maintainer="devlog.moonduck@gmail.com"

RUN yum install -y initscripts ambari-agent && yum clean all && rm -rf /var/cache/yum 

COPY run-ambari-agent.sh /entry/usr/

COPY ambari-conf/ambari-agent.ini /etc/ambari-agent/conf/

RUN ssh-keygen -t rsa -q -f "/root/.ssh/id_rsa" -N "" && curl https://raw.githubusercontent.com/dev-m00n/ambari-local-server/master/id_rsa.pub -o /root/.ssh/authorized_keys

#Since installing rpcbind overwrites systemctl in /usr/bin/, we need to move custom systemctl to safe place, and get it back to /usr/bin/
RUN mv /usr/bin/systemctl /tmp/ && yum install -y rpcbind && yum clean all && rm -rf /var/cache/yum && mv /tmp/systemctl /usr/bin/
```

## TODO
- Ambari 외부 PostgresQL Container
- 보다 쉬운 설정을 위한 UI기반의 설정([React JS](https://reactjs.org/) 기반)
- 또다른 흥미로운 Hadoop 기반 기술 적용
	- [Apache Iceberg](https://iceberg.apache.org/)
	- [Presto](https://prestodb.io/)
