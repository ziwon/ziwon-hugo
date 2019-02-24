+++
date          = "2019-02-24T16:59:00+09:00"
draft         = true
title         = "[번역] 도커 볼륨"
tags          = ["docker", "storage", "persistence", "volumes", "mounts"]
categories    = ["docker"]
slug          = "docker-storage-volumes"
notoc         = true
socialsharing = true
nocomment     = false
+++

도커 스토리지 목차는 다음과 같다.

- [스토리지 소개](/post/docker-storage-overview/)
- **[* 볼륨 사용](/post/docker-storage-volumes/) ([원문](https://docs.docker.com/storage/volumes/))**
- 바인드 마운트
- tmp 마운트
- 컨테이너에 데이터 저장하기
	- 스토리지 드라이버에 대해
	- 스토리지 드라이버 선택
	- AUFS 스토리지 드라이버 사용
	- Btrfs 스토리지 드라이버 사용
	- 디바이스 맵퍼 스토리지 드라이버 사용
	- OverlayFS 스토리지 드라이버 사용
	- ZFS 스토리지 드라이버 사용
	- VFS 스토리지 드라이버 사용

<center>•••</center>

# 볼륨 사용

볼륨은 도커 컨테이너에 의해 생성되고 사용되는 데이터를 영속하기 위해 선호되는 메커니즘이다. 바인딩 마운트는 호스트 머신의 디렉토리 구조에 따라 달라지지만, 볼륨은 도커에 의해 완전히 관리된다. 볼륨은 바인딩 마운트에 비해 몇 가지 장점이 있다.

- 볼륨은 바인딩 마운트보다 백업 또는 마이그레이션이 더 쉽다.
- Docker CLI 명령이나 Docker API를 사용하여 볼륨을 관리할 수 있다.
- 볼륨은 Linux 컨테이너와 Windows 컨테이너 모두에서 작동한다.
- 볼륨은 여러 컨테이너 사이에서 더 안전하게 공유될 수 있다.
- 볼륨 드라이버는 원격 호스트나 클라우드 프로바이더에게 볼륨을 저장하거나, 볼륨의 내용을 암호화하거나, 다른 기능을 추가할 수 있게 해준다.
- 새 볼륨은 컨테이너에 의해 그 내용물을 미리 채울 수 있다.

또한, 볼륨은 사용하는 컨테이너의 크기를 증가시키지 않고, 주어진 컨테이너의 라이프사이클 밖에 볼륨의 내용이 존재하기 때문에, 컨테이너의 쓰기 가능한 레이어에서 데이터를 영속하는 것보다 더 나은 선택이라고 할 수 있다.

<center><img src="https://docs.docker.com/storage/images/types-of-mounts-volume.png"/></center>

컨테이너가 비영속적인 상태 데이터를 생성하는 경우, `tmpfs` 마운트를 사용하여 데이터를 어디에나 영속적으로 저장하지 않도록 하고, 컨테이너의 쓰기 가능한 레이어에 쓰기를 막아 컨테이너 성능을 향상시키는 것을 고려해볼 수 있다.

볼륨은 `rprivate` 바인딩 전파를 사용하는데, 볼륨에 대한 바인딩 전파는 설정할 수 없다.

## -v 또는 --mount 플래그 선택

원래, `-v` 또는 `--volume` 플래그는 독립형 컨테이너에 사용되었고 `--mount` 플래그는 스웜 서비스에 사용되었다. 그러나 도커 17.06부터 독립형 컨테이너와 함께 `--mount`를 사용할 수도 있다. 일반적으로 `--mount`가 더 명확하고 장황(verbose)하다. 가장 큰 차이점은 `-v` 구문이 모든 옵션을 하나의 필드에 함께 결합하는 반면 `--mount` 구문은 이들을 구분한다는 것이다. 다음은 각 플래그의 구문 비교이다.

> 새 사용자들은 `--volume` 구분보다 간단한 구문인 `--mount` 구문을 사용해야 한다.

볼륨 드라이버 옵션을 지정해야 한다면, `--mount`를 사용해야 한다.

- `-v` 또는 `--volume`: 콜론(:)으로 구분된 3개의 필드로 구성된다. 필드의 순서가 정확해야 하고, 각 필드의 의미는 한눈에 알기 어렵다.
  - 이름을 지닌 볼륨의 경우, 첫 번째 필드는 볼륨의 이름이며, 지정된 호스트 머신에서 고유하다. 익명 볼륨의 경우, 첫 번째 필드는 생략한다.
  - 두 번째 필드는 파일이나 디렉토리가 컨테이너에 마운트되는 경로다.
  - 세 번째 필드는 선택사항이며, `ro`와 같이 쉼표로 구분된 옵션 목록이다. 이러한 옵션은 아래에서 논의한다.
- `--mount`: 복수 키밸류 쌍으로 구성되고 쉼표로 구분되고 각각은 `<key>=<value>` 튜퓰로 구성된다. `--mount` 구문은 `-v` 또는 `--volume`보다 장황하지만, 키의 순서는 중요하지 않고, 플래그의 값도 알기 쉽다.
  - `bind`, `volume` 또는 `tmpfs`가 될 수 있는 마운트의 `type`. 여기 논하는 주제는 볼륨이므로, `type`은 항상 `volume`이다.
  - 마운트의 `source`. 이름을 지닌 볼륨의 경우, 이는 볼륨의 이름이다. 익명 볼륨의 경우 이 필드는 생략한다. `source` 또는 `src`로 지정할 수 있다.
  - `destination`는 파일이나 디렉토리가 컨테이너에 마운트되는 경로를 값으로 취한다. `destination`, `dst` 또는 `target`으로 지정할 수 있다.
  - `readonly` 옵션이 있는 경우, [바인딩 마운트가 읽기 전용으로 컨테이너에 마운트](https://docs.docker.com/storage/volumes/#use-a-read-only-volume)된다.
  - 한 번 이상 지정할 수 있는 `volume-opt` 옵션은 옵션 이름과 값으로 구성된 키밸류 쌍을 취한다.

> **외부 CSV 파서에서 값을 이스케이프(escape)하기** <br/>
> 볼륨 드라이버가 쉼표로 구분된 목록을 옵션으로 받아들이는 경우, 외부 CSV 파서에서 값을 이스케이프해야 한다. `volume-opt`에서 이스케이프하려면, 큰따옴표(")로 묶고 전체 마운트 매개변수를 작은따옴표(')로 묶는다.

> 예를 들어, `local` 드라이버는 마운트 옵션으로 `o` 매개변수의 쉼표로 구분된 목록을 수용한다. 예제는 목록에서 이스케이프할 수 있는 올바른 방법을 보여준다.

> ```sh
$ docker service create \
	--mount 'type=volume,src=<VOLUME-NAME>,dst=<CONTAINER-PATH>,volume-driver=local,volume-opt=type=nfs,volume-opt=device=<nfs-server>:<nfs-path>,"volume-opt=o=addr=<nfs-address>,vers=4,soft,timeo=180,bg,tcp,rw"'
    --name myservice \
    <IMAGE>
```

아래 예제는 가능한 경우 `--mount` 및 `-v` 구문을 모두 표시하고 `--mount`가 먼저 제시된다.

### -v와 --mount 양식의 차이

바인딩 마운트와 달리, 볼륨의 모든 옵션은 `--mount`와 `-v` 플래그 모두에 대해 사용할 수 있다.

서비스가 포함된 볼륨을 사용할 때는 `--mount`만 지원된다.

## 볼륨 생성과 관리

바인드 마운트와 달리 컨테이너 스코프를 벗어난 볼륨을 생성하고 관리할 수 ​​있다.

**볼륨 생성**

```sh
$ docker volume create my-vol
```

**볼륨 목록**

```sh
$ docker volume ls

local               my-vol
```

**볼륨 조사**

```sh
$ docker volume inspect my-vol
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/my-vol/_data",
        "Name": "my-vol",
        "Options": {},
        "Scope": "local"
    }
]
```

**볼륨 삭제**

```sh
$ docker volume rm my-vol
```

## 볼륨이 있는 컨테이너 시작 

아직 존재하지 않는 볼륨으로 컨테이너를 시작하면, 도커가 볼륨을 생성한다. 다음 예제는 볼륨 `myvol2`를 컨테이너의 `/app/`에 마운트한다.

아래의 `-v`와 `--mount` 예는 동일한 결과를 나타낸다. 첫 번째 컨테이너를 실행한 후 `devtest` 컨테이너와 `myvol2` 볼륨을 제거하지 않으면 둘 다 실행할 수 없다.

**--mount**

```sh
$ docker run -d \
  --name devtest \
  --mount source=myvol2,target=/app \
  nginx:latest
```

**-v**

```sh
$ docker run -d \
  --name devtest \
  -v myvol2:/app \
  nginx:latest
```

`docker inspect devtest`를 사용해서 볼륨이 생성되고 정확하게 마운트되었는지 확인한다. `Mounts` 섹션을 찾아보라.

```json
"Mounts": [
    {
        "Type": "volume",
        "Name": "myvol2",
        "Source": "/var/lib/docker/volumes/myvol2/_data",
        "Destination": "/app",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    }
],
```

위에서 마운트가 볼륨이고, 정확한 소스와 목적지를 확인할 수 있고, 마운트가 읽기-쓰기임을 알 수 있다.

컨테이너를 멈추고 볼륨을 제거한다. 볼륨 제거는 별도의 단계임을 유의하라.

```sh
$ docker container stop devtest

$ docker container rm devtest

$ docker volume rm myvol2
```

## 볼륨이 있는 서비스 시작

서비스를 시작하고 볼륨을 정의할 때 각 서비스 컨테이너는 자체 로컬 볼륨을 사용한다. 로컬 볼륨 드라이버를 사용한다면 어떤 컨테이너도 이 데이터를 공유할 수 없지만, 일부 볼륨 드라이버는 공유 스토리지를 지원한다. Docker for AWS와 Docker for Azure는 모두 Cloudstor 플러그인을 사용하는 영구 스토리지(persistent storage)를 지원한다.

다음 예는 각각 `mysvol2`라는 로컬 볼륨을 사용하는 네 개의 복제본으로 `nginx` 서비스를 시작한다.

```sh
$ docker service create -d \
  --replicas=4 \
  --name devtest-service \
  --mount source=myvol2,target=/app \
  nginx:latest
```

`docker service ps devtest-service`를 사용해서 서비스가 실행 중인지 확인하라.

```sh
$ docker service ps devtest-service

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
4d7oz1j85wwn        devtest-service.1   nginx:latest        moby                Running             Running 14 seconds ago
```

서비스를 제거하면 모든 작업이 중지된다.

```sh
$ docker service rm devtest-service
```

서비스를 제거해도 서비스가 생성한 볼륨은 제거되지 않는다. 볼륨 제거는 별도의 단계이다.

### 서비스에 대한 구문 차이

`docker service create` 명령은 `-v` 또는 `--volume` 플래그를 지원하지 않는다. 볼륨을 서비스의 컨테이너에 마운트 할 때 `--mount` 플래그를 사용해야 한다.

### 컨테이너를 사용하여 볼륨 채우기

위와 같이 새 볼륨을 만드는 컨테이너를 시작하고 컨테이너에 마운트할 디렉토리 (예: 위 `/app/`)에 파일 또는 디렉토리가 있으면 해당 디렉토리의 내용이 볼륨에 복사된다. 그런 다음 컨테이너가 볼륨을 마운트하고 사용하며, 볼륨을 사용하는 다른 컨테이너도 미리 채워진 컨텐츠에 액세스할 수 있다.

이를 설명하기 위해 예제는 `nginx` 컨테이너를 시작하고 컨테이너의 `/usr/share/nginx/html` 디렉토리의 내용으로 새 볼륨 `nginx-vol`를 채운다. 이 디렉토리는 `nginx`가 기본 HTML 컨텐츠를 저장하는 곳이다.

**--mount**

```sh
$ docker run -d \
  --name=nginxtest \
  --mount source=nginx-vol,destination=/usr/share/nginx/html \
  nginx:latest
```

**-v**

```sh
$ docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html \
  nginx:latest
```

이 예제 중 하나를 실행한 후 다음 명령을 실행하여 컨테이너와 볼륨을 정리한다. 볼륨 제거는 별도의 단계이다.

```sh
$ docker container stop nginxtest

$ docker container rm nginxtest

$ docker volume rm nginx-vol
```

## 읽기 전용 볼륨 사용

일부 개발 애플리케이션의 경우, 컨테이너가 바인드 마운트에 쓰기 작업을 수행해야 변경 사항이 다시 도커 호스트에 전파된다. 다른 경우에는 컨테이너가 데이터에 대한 읽기 액세스만 요구한다. 여러 컨테이너가 동일한 볼륨을 마운트할 수 있으며 일부 컨테이너는 읽기/ 쓰기로 마운트하고 다른 컨테이너는 읽기 전용으로 마운트 할 수 있음을 기억하라.

이 예제는 위의 것을 수정하지만 컨테이너의 마운트 지점 다음에 옵션 목록 (기본값은 비어 있음)에 `ro`를 추가하여 디렉토리를 읽기 전용 볼륨으로 마운트한다. 여러 옵션이 있는 경우 쉼표로 구분한다.

`--mount` 및 `-v` 예제의 결과는 동일하다.

**--mount**

```sh
$ docker run -d \
  --name=nginxtest \
  --mount source=nginx-vol,destination=/usr/share/nginx/html,readonly \
  nginx:latest
```

**-v**

```sh
$ docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html:ro \
  nginx:latest
```

`docker inspect nginxtest `를 사용해서 읽기전용 마운트가 정확하게 생성되었는지 확인한다. `Mounts` 섹션을 찾아보라.

```json
"Mounts": [
    {
        "Type": "volume",
        "Name": "nginx-vol",
        "Source": "/var/lib/docker/volumes/nginx-vol/_data",
        "Destination": "/usr/share/nginx/html",
        "Driver": "local",
        "Mode": "",
        "RW": false,
        "Propagation": ""
    }
],
```

컨테이너를 멈추고 제거한 다음 볼륨을 제거하라. 볼륨 제거는 별도의 단계이다.

```sh

$ docker container stop nginxtest

$ docker container rm nginxtest

$ docker volume rm nginx-vol
```

## 머신 간에 데이터 공유

내결함성(fault-tolerant) 어플리케이션을 빌드할 때 같은 파일에 액세 할 수 있도록 동일한 서비스의 여러 복제본을 구성해야할 수도 있다.

<center><img src="https://docs.docker.com/storage/images/volumes-shared-storage.svg"/></center>

애플리케이션을 개발할 때 이를 수행 할 수 있는 몇 가지 방법이 있다. 하나는 애플리케이션에 로직을 추가하여 Amazon S3와 같은 클라우드 객체 저장소 시스템에 파일을 저장하는 것이다. 다른 하나는 NFS 또는 Amazon S3와 같은 외부 스토리지 시스템에 파일 쓰기를 지원하는 드라이버로 볼륨을 생성하는 것이다.

볼륨 드라이버를 사용하면 애플리케이션 로직에서 기본 저장소 시스템을 추상화할 수 있다. 예를 들어, 서비스에서 NFS 드라이버와 함께 볼륨을 사용하는 경우, 애플리케이션 로직을 변경하지 않고 클라우드에 데이터를 저장하는 예와 같이 다른 드라이버를 사용하도록 서비스를 업데이트할 수 있다.

## 볼륨 드라이버 사용

`docker volume create`를 사용하여 볼륨을 만들거나 아직 생성되지 않은 볼륨을 사용하는 컨테이너를 시작할 때 볼륨 드라이버를 지정할 수 있다. 다음 예제는 독립형 볼륨을 만들 때 처음 `vieux/sshfs` 볼륨 드라이버를 사용하고 새 볼륨을 만드는 컨테이너를 시작할 때 사용한다. 

### 초기 셋업

이 예제에서는 두 개의 노드가 있다고 가정한다. 첫 번째 노드는 도커 호스트이고 두 번째 노드는 SSH를 사용하여 연결할 수 있다. 도커 호스트에서 `vieux/sshfs` 플러그인을 설치한다.

```sh
$ docker plugin install --grant-all-permissions vieux/sshfs
```

### 볼륨 드라이버를 사용해서 볼륨 생성 

이 예제는 SSH 암호를 지정하지만 두 호스트에 공유 키가 설정된 경우, 암호를 생략할 수 있다. 각 볼륨 드라이버는 0개 이상의 설정 가능한 옵션이 있을 수 있고, 각 옵션은 `-o` 플래그를 사용하여 지정된다.

```sh
$ docker volume create --driver vieux/sshfs \
  -o sshcmd=test@node2:/home/test \
  -o password=testpassword \
  sshvolume
```

### 볼륨 드라이버를 사용하여 볼륨을 생성하는 컨테이너 시작

이 예제는 SSH 암호를 지정하지만 두 호스트에 공유 키가 설정된 경우, 암호를 생략할 수 있다. 각 볼륨 드라이버에는 0개 이상의 설정 가능한 옵션이 있을 수 있다. 볼륨 드라이버에서 옵션을 전달해야하는 경우, `-v` 대신 `-mount` 플래그를 사용하여 볼륨을 마운트해야 한다.

```sh
$ docker run -d \
  --name sshfs-container \
  --volume-driver vieux/sshfs \
  --mount src=sshvolume,target=/app,volume-opt=sshcmd=test@node2:/home/test,volume-opt=password=testpassword \
  nginx:latest
```

## 데이터 볼륨 백업, 복원, 마이그레이션

볼륨은 백업, 복원 및 마이그레이션에 유용하다. 해당 볼륨을 마운트하는 새 컨테이너를 작성하려면 `--volumes-from` 플래그를 사용하라.

### 컨테이너 백업
예를 들어, 다음 명령에서 우리는

- 새 컨테이너를 시작하고 `dbstore` 컨테이너에서 볼륨을 마운트한다.
- `/backup`으로 로컬 호스트 디렉토리 마운트한다.
- `dbdata` 볼륨의 내용을 `/backup` 디렉토리의 `backup.tar` 파일로 압축하는 명령을 전달한다.

```sh
$ docker run --rm --volumes-from dbstore -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
```

명령이 끝나고 컨테이너가 정지되면, `dbdata` 볼륨의 백업이 남겨진다.

### 백업으로부터 데이터 복원

방금 생성된 백업을 동일한 컨테이너 또는 다른 곳에 만든 백업으로 복원할 수 있다. 예를 들어 `dbstore2`라는 새 컨테이너를 만든다.

```sh
docker run -v /dbdata --name dbstore2 ubuntu /bin/bash
```

그런 다음 새 컨테이너의 데이터 볼륨에서 백업 파일의 압축을 푼다.

```sh
$ docker run --rm --volumes-from dbstore2 -v $(pwd):/backup ubuntu bash -c "cd /dbdata && tar xvf /backup/backup.tar --strip 1"
```

위의 기술을 사용하여 원하는 도구로 백업, 마이그레이션 및 복원 테스트를 자동화할 수 있다.

## 볼륨 제거

도커 데이터 볼륨은 컨테이너가 삭제된 후에도 유지된다. 다음은 고려해 보아야할 두 가지 유형의 볼륨이다.

- **네임드 볼륨(named volume)**에는 컨테이너 외부의 특정 소스 양식이 있다. (예: `awesome:/bar`).
- **익명 볼륨**에는 특정 소스가 없으므로 컨테이너를 삭제할 때 도커 엔진 데몬에 제거하도록 지시한다.

### 익명 볼륨 제거
익명 볼륨을 자동으로 제거하려면 `--rm` 옵션을 사용하라. 예를 들어, 이 명령은 익명 `/foo` 볼륨을 작성한다. 컨테이너가 제거되면 도커 엔진은 `/foo` 볼륨을 제거하지만 `awesome` 볼륨은 제거하지 않는다.

```sh
$ docker run --rm -v /foo -v awesome:/bar busybox top
```

다음 명령은 사용하지 않는 볼륨을 모두 제거하고 공간을 확보한다.

```sh
$ docker volume prune
```






