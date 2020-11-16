# 똥컴에 Ambari Hadoop Cluster를 Local에 구축해보는 경험 수기 - 2단계, Ambari Server Image 구축
- Base Image는 [여기](./2020-11-15-struggling-with-ambari.md)서 했음


## 목적
- 개인적인 목적의 Spark App개발과 새로운 Hadoop 기술 테스트 베드로서의 목적을 하는 Docker기반의 클러스터를 구성해보고자 함
- Hadoop의 Pseudo Distribution 모드가 아닌 Full distribution으로 구성해보고자 함
- CentOS base의 여러 컨테이너를 띄워 각각이 Cluster의 Node가 될 예정

## 나의 노트북 주요사양
- [Pentium 2020m](https://ark.intel.com/content/www/us/en/ark/products/71142/intel-pentium-processor-2020m-2m-cache-2-40-ghz.html) [passmark](https://www.cpubenchmark.net/cpu.php?cpu=Intel+Pentium+2020M+%40+2.40GHz&id=1855)
- Intel HD graphics
- DDR3 RAM 16GB
- Windows 10

상기 스펙을 보다시피, 현재 최신 보급형에도 한참 못미치는 스펙으로, 워낙 저스펙 CPU라 사실 제대로 돌아갈거라고 예상하고 있진 않음

## 사용된 주요 기술 스택
- [Apache Ambari](https://ambari.apache.org/)
- [Docker](https://www.docker.com/)
- [Hadoop](https://hadoop.apache.org/)
- [Hive](https://hive.apache.org/)
- [Kafka](https://kafka.apache.org/)

## 시작 전 준비되야 할 사항들
- [Base Image 구축 과정](./2020-11-15-struggling-with-ambari.md)을 보고 오면 좋음


## 본문
### Server image 만들기

[이전 과정](./2020-11-15-struggling-with-ambari.md)에서 정의한 Base Image는 [내 docker hub repository](https://hub.docker.com/r/devmoonduck/ambari-local-base)에 Push 되었다. 여기서는 이 Base image를 기반으로 Ambari 서버를 구축할 것이다.

#### Base image를 기반으로 Dockerfile 생성
```dockerfile
FROM devmoonduck/ambari-local-base:2.7.4.0
```
특별히 설명할 건 없다. 다음 단계로 넘어가자.  

#### ambari server 설치
[이전 단계](./2020-11-15-struggling-with-ambari.md)에서 이미 Hortonworks Repo를 추가했기때문에 간단하게 yum으로 설치가 가능하다.
```dockerfile
RUN yum install -y ambari-server
```

#### 미리 설정된 설정 파일을 지정된 위치에 놓기
나의 경우 ambari db로 외부 PostgresQL을 사용할 예정이다. 여러가지 이유가 있지만 Production 상황에서 절대 In-memory db나 Ambari server내에 Embedding한 DB를 사용할리가 없기에 이와 같은 선택을 했다.  
그런데 Ambari server를 설치하고 나면 아래와 같은 명령어로 Command line으로 interaction 하며 설정을 입력해줘야한다.
```bash
$> ambari-server setup
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1):
...
```
Image 실행시 이렇게 일일히 입력할 순 없으므로, 미리 설정파일을 만들어놓고 실행시켜도 노프라블럼이다.
참고로 위와 같은 프로세스를 통해 `/etc/ambari-server/conf`에 `ambari.properties`와 `password.dat`가 생성된다.
그러므로 우리는 `ambari.properties`와 `password.dat`를 만들어놓고 해당 경로에 놓기만 하면 된다. 위 파일들은 [여기](https://github.com/dev-m00n/ambari-local-server/tree/master/ambari-conf)에 있다.
```dockerfile
COPY ambari-conf/ambari.properties ambari-conf/password.dat /etc/ambari-server/conf/
```

다시 한 번 강조하면, 나는 외부 PostgresQL을 사용할 예정이고, 이 역시 Docker container가 될 예정이다.

이렇게 생성된 Dockerfile을 실행하면 따로 설정없이 바로 실행할 수 있다.

그러나 `ambari-server start`를 입력하면 분명히 오류를 내뿜으면서 제대로 실행이 되지 않을 것이다.

이유는 Ambari server에서 필요한 DB user, database, table등을 생성해주어야하며, Ambari에서 Query File을 제공한다. [여기](https://github.com/apache/ambari/blob/trunk/ambari-server/src/main/resources/Ambari-DDL-Postgres-CREATE.sql) 참고

외부 PostgresQL은 다음 포스트에서 다룰 예정이며, 여기서는 이미 PostgresQL에서 위 링크의 DDL커맨드를 정상적으로 실행했다고 가정한다.

여기까지 따라왔다면 분명히 Ambari server가 정상적으로 실행될 것이다. 실행은 다음과 같이 한다.
```bash
ambari-server start
```

하지만 몇 가지 빠진 것이 있다. 당장은 실행되지만, 나중에 필요한 것들이 있다.

예를 들면, Ambari server에서 Hadoop을 설치할 때 agent로 ssh로 접근하여 이것 저것 설치하고 설정하는데, 이를 위해서는 Agent측에 Ambari server의 SSH public key가 `authorized_keys`에 등록되어 있어야 한다.

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
- Ambari Agent Image
- Ambari 외부 PostgresQL Container
- 보다 쉬운 설정을 위한 UI기반의 설정([React JS](https://reactjs.org/) 기반)
- 또다른 흥미로운 Hadoop 기반 기술 적용
	- [Apache Iceberg](https://iceberg.apache.org/)
	- [Presto](https://prestodb.io/)