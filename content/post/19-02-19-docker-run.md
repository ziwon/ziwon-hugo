+++
date          = "2019-02-11T11:55:00+09:00"
draft         = false
title         = "[번역] docker run"
tags          = ["docker"]
categories    = ["docker"]
slug          = "docker-run"
notoc         = true
socialsharing = true
nocomment     = false
+++

> Docker run의 여러가지 매개변수에 대해 자세히 알고자 번역해보았다.


# docker run

## 설명

새 컨테이너에서 명령을 실행한다.

## 사용법

```sh
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

## 옵션

|이름, 약식                | 기본값  |설명|
|-------------------------|--------|-----------------------------------------------------------------------|
|`--add-host`             |        |커스텀 호스트-대-IP 매핑을 추가 (host:ip)                                  |
|`--attach, -a`   		   |        |STDIN, STDOUT 또는 STDERR를 연결.                                       |
|`--blkio-weight`  		   |        |10과 1000 사이의 IO (상대적 가중치) 차단 혹은 사용하지 않으려면 0 (기본값 0)   |
|`--blkio-weight-device`  | 		|블록 IO 가중치 (상대적인 디바이스 가중치)                                    |
|`--cap-add`			   |        |리눅스 기능 추가                                                         |
|`--cap-drop`			   |        |리눅스 기능 중단                                                         |
|`--cgroup-parent`		   |		|컨테이너의 선택적인 부모 cgroup                                            |
|`--cidfile`			   |		|컨테이너 ID를 파일에 작성                                                 |
|`--cpu-count`			   |        |CPU 수 (윈도우만 해당)                                                   |
|`--cpu-percent`		   |		|CPU 백분율 (윈도우만 해당)                                                |
|`--cpu-period `		   |		|CPU CFS(Completely Fair Scheduler) period 제한 (API 1.25+)              |
|`--cpu-quota`			   |		|CPU CFS(Completely Fair Scheduler) 할당량 제한 (API 1.25+)               |
|`--cpu-rt-period`		   |		|마이크로 초단위로 실시간 period 제한                                       |
|`--cpu-rt-runtime`		   |		|마이크로 초단위로 실시간 런타임 제한                                         |
|`--cpu-shares , -c`	   |		|CPU 공유 (상대적 가중치)                                                  |
|`--cpus`				   |		|CPU 수	                                                                 |
|`--cpuset-cpus`		   |		|실행을 허용할 CPU (0-3, 0, 1)                                             |
|`--cpuset-mems`		   |        |실행을 허용할 MEM (0-3, 0, 1)                                            |
|`--detach, -d`			   |		|백그라운드에서 컨테이너를 실행하고 컨테이너 ID를 출력                          |
|`--detach-keys`	       |        |컨테이너를 분리하기 위한 키순서 재정의                                       |
|`--device`               |        |컨테이너에 호스트 디바이스 추가                                             |
|`--device-cgroup-rule`   |        |cgroup 허용 디바이스 목록에 규칙 추가                                      |
|`--device-read-bps`	   |        |디바이스로부터 읽기 속도 (초당 바이트) 제한                                  |
|`--device-read-iops`     |        |다바이스로부터 읽기 속도 (초당 IO) 제한                                     |
|`--device-write-bps` 	   |		|디바이스에 쓰기 속도 (초당 바이트) 제한                                      |
|`--device-write-iops` 	   |        |디바이스에 쓰기 속도 (초당 IO) 제한                                        |
|`--disable-content-trust`|`true`  |이미지 확인 생략                                                         |
|`--dns`				   |		|사용자 지정 DNS 서버 설정                                                |
|`--dns-option`           |        |DNS 옵션 설정                                                          |
|`--dns-search`           |        |사용자 지정 DNS 검색 도메인 설정                                          |
|`--entrypoint`           |        |이미지의 기본 ENTRYPOINT를 재정의                                        |
|`--env , -e`             |        |환경변수 설정                                                           |
|`--env-file`             |        |환경변수에서 파일 읽기                                                    |
|`--expose`               |        |포트 또는 포트 범위 노출                                                  |
|`--group-add`            |        |조인할 추가적인 그룹을 추가                                               |
|`--health-cmd`           |        |헬스 체크하기 위해 실행하는 명령                                           |
|`--health-interval` 	   |        |검사를 실행하는 시간 (ms|s|m|h) (기본값 0초)                               |
|`--health-retries`       |        |언헬시 보고에 필요한 연속 실패                                             |
|`--health-start-period`  |        |health-retries 카운트다운을 시작하기 전에 컨테이너를 초기화하는 시작 시간 (ms|s|m|h) (기본값 0s) |
|`--health-timeout`       |        |한 번의 체크가 실행될 수 있는 최대 시간 (ms|s|m|h)                           |
|`--help`                 |        |사용법 출력                                                              |
|`--hostname , -h`        |        |컨테이너 호스트 이름                                                      |
|`--init`                 |        |시그널을 전달하고 및 프로세스를 수확을 위한 컨테이너 내의 init 실행             |
|`--interactive , -i`     |        |STDIN을 연결하지 않아도 오픈시킴                                           |
|`--io-maxbandwidth`      |        |시스템 드라이브의 최대 입출력 대역폭 제한 (윈도우만)                          |
|`--io-maxiops`           |        |시스템 드라이브의 최대 입출력 제한 (윈도우만)                                |
|`--ip`                   |        |IPv4 주소 (예. 172.30.100.104)                                          |
|`--ip6`                  |        |IPv6 주소 (예. 2001:db8::33)                                           |
|`--ipc`                  |        |사용할 IPC 모드                                                         |
|`--isolation`            |        |컨테이너 격리 기술                                                        |
|`--kernel-memory`        |        |커널 메모리 제한                                                          |
|`--label, -l`            |        |컨테이너 메타데이터 설정                                                    |
|`--label-file`           |        |라인으로 구분된 라벨 파일 읽기                                              |
|`--link`                 |        |또다른 컨터이너에 대한 링크 추가                                             |
|`--link-local-ip`        |        |컨테이너 IPv4/IPv6 링크 로컬 주소                                          |
|`--log-driver`           |        |컨테이너에 대한 로깅 드라이버                                               |
|`--log-opt`              |        |로깅 드라이버 옵션                                                        |
|`--mac-address`          |        |컨테이너 맥 어드레스 (예. 92:d0:c6:0a:29:33)                               |
|`--memory , -m`          |        |메모리 제한                                                              |
|`--memory-reservation`   |        |메모리 소프트 제한                                                        |
|`--memory-swap`          |        |메모리 플러스 스왑과 같은 스왑 제한: '-1'로 무제한 스왑 가능                   |
|`--memory-swappiness`    | `-1`   |컨테이너 메모리 스왑니스 조정 (0 ~ 100)                                   |
|`--mount`                |        |컨테이너에 파일 시스템 마운트를 연결                                        |
|`--name`                 |        |컨테이너에 이름을 할당                                                    |
|`--net`                  |        |네트워크에 컨테이너를 연결                                                 |
|`--net-alias`            |        |컨테이너에 대한 네트워크 범위 앨리어스 추가                                    |
|`--network`              |        |네트워크에 컨테이를 연결                                                    |
|`--network-alias`        |        |컨테이너에 대한 네트워크 범위 앨리어스 추가                                   |
|`--no-healthcheck`       |        |컨터이너에 지정된 HEATHCHECK 비활성화                                       |
|`--oom-kill-disable`     |        |OOM 킬러 비활성화                                                         |
|`--oom-score-adj`        |        |호스트의 OOM 기본 설정 조정 (-1000 ~ 1000)                                 |
|`--pid`                  |        |사용할 PID 네임스페이스                                                    |
|`--pids-limit`           |        |컨테이너 pids 제한 조정 (무제한은 -1 설정)                                  |
|`--platform`             |        |서버가 멀티 플랫폼 가능한 경우 플랫폼 설정                                   |
|`--privileged`           |        |이 컨테이너에 확장된 권한을 부여                                            |
|`--publish , -p`         |        |컨테이너 포트를 호스트에 게시                                               |
|`--publish-all , -P`     |        |노출된 모든 포트를 임의 포트에 게시                                          |
|`--read-only`            |        |컨테이너 루트 파일시스템을 읽기 전용으로 마운트                                |
|`--restart`              | `no`   |컨테이너가 종료될 때 적용할 재시작 정책                                       |
|`--rm`                   |        |컨테이너가 종료되면 자동으로 제거                                            |
|`--runtime`              |        |컨테이너에 사용하는 런타임                                                  |
|`--security-opt`         |        |보안 옵션                                                                |
|`--shm-size`             |        | /dev/shm 크기                                                          |
|`--sig-proxy`            |`true`  |프로세스에 수신된 시그널을 프록시                                           |
|`--stop-signal`          |`SIGTERM` | 컨테이너 정지할 시그널                                                  |
|`--stop-timeout`         |        |컨테이너를 정지하는 타임아웃 (초단위)                                        |
|`--storage-opt`          |        |컨테이너의 저장소 드라이버 옵션                                             |
|`--sysctl`               |        |sysctl 옵션                                                             |
|`--tmpfs`                |        | tmpfs 디렉토리 마운트                                                   |
|`--tty , -t`             |        | pseudo-TTY 할당                                                        |
|`--ulimit`               |        | Ulimit 옵션                                                           |
|`--user , -u`            |        |사용자 이름 또는 UID (형식: <이름|uid>)                                    |
|`--userns`               |        |사용할 사용자 네임스페이스                                                  |
|`--uts`                  |        |사용할 UTS 네임스페이스                                                    |
|`--volume , -v`          |        |마운트하는 볼륨을 바인딩                                                    |
|`--volume-driver`        |        |컨테이너용 선택적 볼륨 드라이버                                              |
|`--volumes-from`         |        |지정된 컨테이너로부터 볼륨 마운트                                            |
|`--workdir , -w`         |        |컨테이너 내의 작업 디렉토리                                                 |

## 부모 명령
|명령어                	|설명                                                                    |
|----------------------|-----------------------------------------------------------------------|
|docker					| Docker CLI의 기본 명령어                                                |

