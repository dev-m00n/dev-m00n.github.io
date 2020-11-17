---
title: 다목적 Hadoop local cluster 구축 - Base Image 만들기(Ambari on Docker base image)
author: Moon
date: 2020-11-15 21:17:00 +0700
categories: [Hadoop]
tags: [Hadoop, Ambari, Docker]
math: true
---
# 똥컴에 Ambari Hadoop Cluster를 Local에 구축해보는 경험 수기 - 1단계, Base Image 구축

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
- 나는 2020M이라는 노인 중의 노인급 CPU를 사용하고 있지만, 이 글을 보고 시도하려는 당신은 이런 패륜을 저지르지는 않았으면 한다. 참고로 CPU는 CPU가상화를 지원해야 Docker를 사용할 수 있으므로, 이게 안된다면 더이상 진행하지 않는게 좋다.
- RAM은 양심적으로 16G부터...
- 나와 같은 Windows 10사용자는 Docker를 사용하기 위해 Linux가 필요하며, Linux를 Windows에서 실행하기 위해 [WSL(Windows Subsystem for Linux)](https://docs.microsoft.com/en-us/windows/wsl/install-win10)를 설치해야한다.


## 본문
### Base image 만들기

Ambari server와 agent에서 공통으로 사용될 의존성과 설정등을 Base Image에서 정의할 것이다.

#### Ambari 공통 의존성 설치

2020년 11월 기준, 최신 Ambari 버전은 2.7.5이지만, 최신판은 돈을 지불해야하므로, Money쭈구리 답게 한 버전 낮게 간다. 바로 이전 버전인 2.7.4는 무료이다.
[Ambari prerequisite](https://cwiki.apache.org/confluence/display/AMBARI/Ambari+Development)을 보면 다음 의존성 앱들이 CentOS내에 설치되어 있어야 한다.
- JDK 8
- maven
- GCC-C++
- rpm-build

하지만 이는 CentOS Docker Image가 아닌 일반 서버용 CentOS 기준이기때문에, 아래와 같은 의존성이 추가로 필요하게 될 것이다.(여러번의 설치를 시도하며 알게 된 추가 의존성들이다.)
- openssh server/client
- chrony

그러면 Dockerfile은 아래와 같이 된다.
```dockerfile
FROM centos:centos7

RUN yum install -y "java-1.8.0-openjdk" "maven" "gcc-c++" "rpm-build" "openssh-server" "openssh-clients" "chrony"
```

하지만 이후 여러번의 구축시도 끝에 Base Image에는 추가적으로 해야할 일이 꽤 있다는 것을 알게 되었다.

#### yum repo에 Hortonworks repository 추가
yum으로 Ambari설치를 위한 ambari repo를 yum repository에 추가  
참고: [Link](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.4.0/bk_ambari-installation/content/download_the_ambari_repo_lnx7.html)

```dockerfile
ARG AMBARI_VERSION="2.7.4.0"

RUN curl "http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/$AMBARI_VERSION/ambari.repo" -o /etc/yum.repos.d/ambari.repo
```
#### Host SSH key 생성
Host의 SSH key가 없기때문에 만들어줘야한다.
```dockerfile
RUN ssh-keygen -A
```

#### Chrony를 실행하기 위한 추가적인 Hacky work
Docker에서 각종 init service들이 돌아가지 않기때문에 약간의 Hack이 필요하다.  
참고: [docker-systemctl-replacement](https://github.com/gdraheim/docker-systemctl-replacement)  
이게 왜 필요하냐면, Cluster 노드들간의 시간 동기화를 위해 NTP 혹은 Chrony가 필요한데, 이는 Container안에 백그라운드에 실행되는 데몬이며, 이는 SRP(Single resposibility princible)를 위해하는 것이기때문에 권장되진 않으나, Ambari에서 요구하기때문에 위 Hack을 통해 systemctl이 container에서 동작하도록 한다.
그럼 위 [systemctl script](https://github.com/gdraheim/docker-systemctl-replacement/blob/master/files/docker/systemctl.py)를 받아 Dockerfile과 같은 디렉토리에 넣어놓고 아래와 같은 Script가 추가되었다.
```dockerfile
COPY systemctl /usr/bin/systemctl
RUN chmod 555 /usr/bin/systemctl

ENTRYPOINT [ "systemctl" ]
```

4. Chrony service가 제대로 동작하지 않는다.
위의 Hack으로 다른 Service가 동작하게 되었지만, `systemctl start chronyd`는 아래와 같은 오류를 뿜어내며 정상동작하지 않았다.
```
Oops, 1 unsupported directory settings. You need to create those before using the service.
```
에러가 참 의미없음에 통탄한다. 도대체 어떤 unsupported directory setting이고 뭘 create 해야하는지 이 에러문구로는 유추하기가 불가능했다.
그래서 해당 스크립트의 Code를 파헤쳐보니 아래와 같은 로직을 보고 추적해나가서 chrony service를 설정해줘야한다는 것을 알게되었다.
```python
for setting in ("PrivateTmp", "PrivateDevices", "PrivateNetwork", "PrivateUsers", "DynamicUser", 
```
chrony service 설정은 `/usr/lib/systemd/system/chronyd.service`에 있으며 까보니 역시나 저 설정들이 있었다.
```
PrivateTmp=yes
ProtectHome=yes
```
`no`로 바꾸고 실행해보니 service가 정상적으로 올라왔다.

### 완성된 Base Image
여기까지한 것에 이것저것 좀 더 개선한 최종본은 내 Git repo에 push 하였다.  
[Ambari local base image source code](https://github.com/dev-m00n/ambari-local-base)

Dockerfile은 아래와 같으며 Repository에 각종 Script파일이 올라와 있으니 참고하길 바란다.
```dockerfile
FROM centos:centos7

ARG AMBARI_VERSION="2.7.4.0"

LABEL maintainer="devlog.moonduck@gmail.com" 

# Install required dependencies to install ambari
RUN yum clean all -y && yum install -y "java-1.8.0-openjdk" "maven" "gcc-c++" "rpm-build" "openssh-server" "openssh-clients" "chrony" && yum clean all -y && rm -rf /var/cache/yum 

# Add Ambari in yum repository
RUN curl "http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/$AMBARI_VERSION/ambari.repo" -o /etc/yum.repos.d/ambari.repo

ENV container docker

RUN ssh-keygen -A

#Make systemctl work
COPY systemctl /usr/bin/systemctl
RUN chmod 555 /usr/bin/systemctl

#Make chronyd work
COPY chronyd.service /usr/lib/systemd/system/chronyd.service 

#Create directory for user defined scripts
RUN mkdir -p /entry/usr && mkdir /entry/init
COPY scripts/run.sh /entry
COPY scripts/add_admin.sh /entry/init

ENTRYPOINT [ "/entry/run.sh" ]
```
여기까지는 Ambari Agent와 Ambari Server에서 공통으로 사용될 부분을 정의한 것이며 다음 장에서 Ambari server의 Image를 구축해볼 예정이다.


## TODO
- Ambari Server Image
- Ambari Agent Image
- 보다 쉬운 설정을 위한 UI기반의 설정([React JS](https://reactjs.org/) 기반)
- 또다른 흥미로운 Hadoop 기반 기술 적용
	- [Apache Iceberg](https://iceberg.apache.org/)
	- [Presto](https://prestodb.io/)