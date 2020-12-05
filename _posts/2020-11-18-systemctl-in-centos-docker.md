---
title: CentOS에서 Service 동작하도록 하기
author: Moon
date: 2020-11-18 21:50:00 +0700
categories: [Troubleshooting]
tags: [Troubleshooting, Docker, CentOS]
math: true
preview: Failed to get D-Bus connection. Docker기반의 OS container에서는 시스템 서비스를 동작하도록 해보자
---
# 목적
Ambari를 Docker에서 실행시키기 위해서는 systemctl로 OS내부의 service들을 동작시켜야 하는데, Docker image에서는 기본적으로 동작하지 않는다. 물론 systemctl이 동작하지 않아도 경고만 발생할 뿐인 것 같지만, 아직 오랜 시간 운영을 해보지 않아 확실하지 않다.  
그래도 경고를 무시하고 진행하면 언젠가 부메랑이 되어 돌아올지 모르므로 고치는게 좋다고 생각하여 많은 시간을 투자해서 결국 동작하도록 했고, 나중에 까먹지 않기 위해 여기 트러블 슈팅 과정을 남겨둠


# 사용된 주요 기술
- [Docker](https://www.docker.com/)
- [CentOS on Docker](https://hub.docker.com/_/centos)
- [docker-systemctl-replacement](https://github.com/gdraheim/docker-systemctl-replacement)
  

# 본문
먼저 증상을 살펴보자.
CentOS에서 chrony를 설치해보자
```bash
$ docker run -it centos:centos7 bash
[root@f7186c52ca65 /]$ yum install -y chrony
Loaded plugins: fastestmirror
...
Complete!
```
이제 실행해보자. 정상적으로 실행되지 않고 `Operation not permitted`를 확인할 수 있다.
```bash
[root@f7186c52ca65 /]$ systemctl start chronyd
Failed to get D-Bus connection: Operation not permitted
```
구글링해보니, `--privileged` 옵션이 필요하다고 했지만 먹히지 않았다.
```bash
$ docker run -it --privileged centos:centos7 bash
[root@f7186c52ca65 /]$ yum install -y chrony
...
[root@f7186c52ca65 /]$ systemctl start chronyd
Failed to get D-Bus connection: Operation not permitted
```
찾아보니 [docker-systemctl-replacement](https://github.com/gdraheim/docker-systemctl-replacement)라는 것이 있었다.
일단 해당 스크립트를 다운받고 기존 systemctl을 이 파일로 덮어씌운다
```bash
[root@f7186c52ca65 /]$ curl https://raw.githubusercontent.com/gdraheim/docker-systemctl-replacement/master/files/docker/systemctl.py -o systemctl

[root@f7186c52ca65 /]$ chmod +x systemctl
[root@f7186c52ca65 /]$ mv -f systemctl /usr/bin/
```
한 번 실행해본다.
```bash
[root@1ddc481921f1 /]$ systemctl enable chronyd
[root@1ddc481921f1 /]$ systemctl start chronyd
ERROR:systemctl: !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
ERROR:systemctl: Oops, 1 unsupported directory settings. You need to create those before using the service.
ERROR:systemctl: !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```
Error 내용이 뭐가 잘못된건지 모르겠다.
그래서 [Code](https://github.com/gdraheim/docker-systemctl-replacement/blob/master/files/docker/systemctl.py)를 까봤다.

에러메시지로 찾아본다. dirproblems에 걸렸다.
```python
if dirproblems:
    logg.error(" Oops, %s unsupported directory settings. You need to create those before using the service.", dirproblems)
                time.sleep(1)
```
타고 올라가서 뭐가 dirproblems를 트리거 시킨걸까 찾아봤다.
```python
for setting in ("RootDirectory", "RootImage", "BindPaths", "BindReadOnlyPaths",
            "RuntimeDirectory", "StateDirectory", "CacheDirectory", "LogsDirectory", "ConfigurationDirectory",
            "RuntimeDirectoryMode", "StateDirectoryMode", "CacheDirectoryMode", "LogsDirectoryMode", "ConfigurationDirectoryMode",
            "ReadWritePaths", "ReadOnlyPaths", "TemporaryFileSystem"):
            setting_value = conf.get(section, setting, "")
    if setting_value:
		logg.warning("%s: %s directory path not implemented: %s=%s", unit, section, setting, setting_value)
		dirproblems += 1
for setting in ("PrivateTmp", "PrivateDevices", "PrivateNetwork", "PrivateUsers", "DynamicUser", 
	"ProtectSystem", "ProjectHome", "ProtectHostname", "PrivateMounts", "MountAPIVFS"):
	setting_yes = conf.getbool(section, setting, "no")
	if setting_yes:
		logg.warning("%s: %s directory option not supported: %s=yes", unit, section, setting)
		dirproblems += 1
```
해당 서비스(chronyd)에 관련된 설정으로 보인다. 서비스에 설정이 yes이거나 어떤 value가 있는 것으로 보인다.
서비스 설정 홈으로 가서 `chrony`의 설정을 살펴본다.
```bash
[root@1ddc481921f1 system]$ cd /usr/lib/systemd/system
[root@1ddc481921f1 system]$ vi chronyd.service
```
찾았다.
```ini
...
[Service]
...
PrivateTmp=yes
ProtectHome=yes
ProtectSystem=full

[Install]
...
```

`no`로 바꾸고 저장한다.
```ini
...
[Service]
...
PrivateTmp=no
ProtectHome=no
ProtectSystem=full

[Install]
...
```
다시 실행해본다
```bash
[root@1ddc481921f1 system]$ systemctl start chronyd
[root@1ddc481921f1 system]$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 15:11 pts/0    00:00:00 bash
chrony      86     1  0 15:16 ?        00:00:00 /usr/sbin/chronyd
root       106     1  0 15:26 pts/0    00:00:00 ps -ef
```

정상적으로 데몬이 실행됨을 볼 수 있다.

여기서는 chrony만 시도해봤지만, 대부분의 서비스가 이 트러블 슈팅으로 해결될 수 있을거라 생각된다.

# 주의사항
Ambari를 local에 구축하며 몇 번의 Troubleshooting이 있었다.
먼저, 이 과정(systemctl replacement) 이후 의존성을 설치할 때 주의해야한다.
예를 들어, `rpcbind`를 설치하는 경우, 이후 이 과정에서 진행한 모든 것이 원점으로 돌아간다.
왜냐하면, `rpcbind`를 설치하면서 몇 가지 의존성을 업데이트하는데 여기서 `systemd`가 같이 업데이트 되기때문이다.
```bash
$> yum install -y rpcbind
...
Installing:
 rpcbind             x86_64        0.2.0-49.el7            base            60 k
Installing for dependencies:
 libtirpc            x86_64        0.2.4-0.16.el7          base            89 k
Updating for dependencies:
 systemd             x86_64        219-78.el7_9.2          updates        5.1 M
 systemd-libs        x86_64        219-78.el7_9.2          updates        418 k
```
이 과정이후엔 `/usr/bin/systemctl`은 OS image 기본으로 되돌아가며, 당연히 service들이 동작하지 않는다.

따라서 의존성 설치하기 전 미리 systemctl을 백업해놓고 다시 덮어씌우는 과정을 까먹지말자