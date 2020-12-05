---
title: Ambari Server Image 만들기
author: Moon
date: 2020-11-16 21:17:00 +0700
categories: [Hadoop]
tags: [Hadoop, Ambari, Docker]
math: true
preview: Ambari Hadoop cluster on local, 두번째 Ambari server image 만들었던 과정
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

## ambari server 설치
[이전 단계](./2020-11-15-struggling-with-ambari.md)에서 이미 Hortonworks Repo를 추가했기 때문에 간단하게 yum으로 설치가 가능하다.
```dockerfile
RUN yum install -y ambari-server
```

## 미리 설정된 설정 파일을 지정된 위치에 놓기
Ambari Server를 설치하고 나면 첫 관문은 설정이다.
원래는 아래와 같이 `ambari-server setup` command를 통해 Ambari server와 CLI로 인터랙션하며 설정해줘야한다.
```bash
$> ambari-server setup
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1):
...
```
이 과정을 완료하면 `/etc/ambari-server/conf`에 `ambari.properties`와 `password.dat`가 생성되는데, 미리 만든 `ambari.properties`와 `password.dat`를 지정된 위치에 놓아둔다면, 굳이 인터랙션을 통해 하나하나 일일히 설정할 필요가 없게된다.

나의 경우 별도의 PostgresQL 컨테이너를 올릴 생각이며, Ambari server가 이 Postgres를 사용하도록 설정하기위해 `ambari.properties`파일을 아래와 같이 수정했다. 참고로 `ambari.properties`는 원본은 [여기](https://github.com/apache/ambari/blob/trunk/ambari-server/conf/unix/ambari.properties) 있는데, 설치후 Ambari에 의해 생성된 properties는 내 [Repository](https://github.com/dev-m00n/ambari-local-server/tree/master/ambari-conf)에 올려두었다. 
바꿔야할 주요 설정은 아래와 같다.
```properties
java.home=/usr/lib/jvm/jre
...
server.jdbc.rca.driver=org.postgresql.Driver
server.jdbc.rca.url=jdbc:postgresql://db.cluster.local:5432/hadoop
server.jdbc.rca.user.name=ambari
server.jdbc.rca.user.passwd=/etc/ambari-server/conf/password.dat
server.jdbc.url=jdbc:postgresql://db.cluster.local:5432/hadoop
server.jdbc.user.name=ambari
server.jdbc.user.passwd=/etc/ambari-server/conf/password.dat
...
stack.java.home=/usr/lib/jvm/jre
```
기타 잡다한, 특히 timeout관련된 것은 머신 성능이 안좋다면 늘려주는게 좋다. 내 똥컴에서는 몇번이나 타임아웃나고 해서 아예 비상식적으로 늘려버렸다.(안그러면 설치 중 타임아웃으로 설치가 취소됨)
```properties
agent.package.install.task.timeout=36000
...
agent.task.timeout=18000
...
```
`password.dat`는 아래와 같은 암호화되어 있지 않은 db 비밀번호가 파일에 저장되어 있다.
```
bigdata
```
이제 위 두 파일을 적절한 위치에 올려주면 따로 ambari server를 설정하지 않아도 된다.

```dockerfile
COPY ambari-conf/ambari.properties ambari-conf/password.dat /etc/ambari-server/conf/
```

이렇게 생성된 Dockerfile을 실행하면 따로 설정없이 바로 실행할 수 있다.

그러나 `ambari-server start`를 입력하면 분명히 오류를 내뿜으면서 제대로 실행이 되지 않을 것이다.

이유는 Ambari server에서 필요한 DB user, database, table등을 생성해주어야하며, Ambari에서 Query File을 제공한다. [여기](https://github.com/apache/ambari/blob/trunk/ambari-server/src/main/resources/Ambari-DDL-Postgres-CREATE.sql) 참고

외부 PostgresQL은 다음 포스트에서 다룰 예정이며, 여기서는 이미 PostgresQL에서 위 링크의 DDL커맨드를 정상적으로 실행했다고 가정한다.

여기까지 따라왔다면 분명히 Ambari server가 정상적으로 실행될 것이다. 실행은 다음과 같이 한다.
```bash
ambari-server start
```

하지만 몇 가지 빠진 것이 있다. 당장은 실행되지만, 나중에 필요한 것들이 있다.

예를 들면, Ambari server에서 각 Agent에 Hadoop을 설치할 때 agent로 ssh로 접근하여 이것 저것 설치하고 설정하는데, 이를 위해서는 Agent측에 Ambari server의 SSH public key가 `authorized_keys`에 등록되어 있어야 한다.

이를 위한 첫 작업으로 ssh key를 container 내부에서 생성하고 다시 Host로 복사해와서 `Dockerfile`과 같은 디렉토리에 위치 시킨다.이제 이 키를 container에 복사하는 구문을 추가한다.
```dockerfile
COPY id_rsa id_rsa.pub /root/.ssh/
```
이 과정을 하는 이유는, Agent에서 Host의 public key를 알아내는 과정을 생략하기 위함이다. 만약 빌드시마다 다른 ssh key가 생성되면, Agent Image가 빌드될 때 이 key를 알아내는 과정이 있어야하며, 꽤나 까다롭기에, 늘 같은 key를 쓰도록 하는 Hacky한 방법이다.

이제 전체 syntax를 보자. [`run-ambari-server.sh`](https://github.com/dev-m00n/ambari-local-server/blob/master/run-ambari-server.sh)는 그냥 `ambari-server start`를 하는, 한 줄 짜리 스크립트이다. 이전 Base Image에서 `/entry/usr` 아래 script들 모두를 실행하도록 했기때문에, 그냥 거기다가 두기만하면 container가 instance화 될 때 실행될 것이다.

```dockerfile
FROM devmoonduck/ambari-local-base:2.7.4.0

LABEL maintainer="devlog.moonduck@gmail.com"

RUN yum install -y ambari-server && yum clean all && rm -rf /var/cache/yum

# entrypoint script will execute run-ambari-server.sh
COPY run-ambari-server.sh /entry/usr/

COPY ambari-conf/ambari.properties ambari-conf/password.dat /etc/ambari-server/conf/

COPY id_rsa id_rsa.pub /root/.ssh/

RUN chmod 644 /root/.ssh/id_rsa.pub && chmod 600 /root/.ssh/id_rsa
```

## TODO
- ~~Ambari Agent Image~~ [여기](./2020-11-17-struggling-with-ambari3.md)서 완료
- Ambari 외부 PostgresQL Container
- 보다 쉬운 설정을 위한 UI기반의 설정([React JS](https://reactjs.org/) 기반)
- 또다른 흥미로운 Hadoop 기반 기술 적용
	- [Apache Iceberg](https://iceberg.apache.org/)
	- [Presto](https://prestodb.io/)
