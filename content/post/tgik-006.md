+++
date          = "2019-02-22T10:25:00+09:00"
draft         = false
title         = "TGI Kubernetes 006: kubeadm"
tags          = ["Kubernetes", "kubeadm", "single master"]
categories    = ["tgik"]
slug          = "tgik-006"
notoc         = true
socialsharing = true
nocomment     = false
+++

[여섯 번째 에피소드](https://github.com/heptio/tgik/blob/master/episodes/006/README.md)는 `kubeadm`을 이용해서 쿠버네티스 클러스터를 설치하는 내용이다. 

<center>•••</center>

## 오픈소스 릴리즈 - Velero (aka. Ark) & Sonobuoy
앞서, Heptio가 릴리즈한 오픈소스에 대해 간략히 설명하는데, 해당 깃헙 저장소의 README 파일을 내용을 요약하면 아래와 같다.

- **[Velero](https://github.com/heptio/velero)**: 클러스터 리소스 및 영구 볼륨을 백업하고 복원하는 도구
	-  클러스터의 백업본을 가져오고 손실된 경우 복원
	-  클러스터 자원을 다른 클러스터에 복사, 
	-  개발 환경 및 테스트 환경을 위해 프로덕션 환경을 복제

- **[Sonobuoy](https://github.com/heptio/sonobuoy)**: 비파괴적인 방식으로 Kubernetes 적합성 테스트 셋을 실행하는 진단 도구
	- 클러스터에 대한 명확한 이해를 위한 보고서 생성
	- 통합 end-to-end 적합성 테스트
	- 사용자 정의 가능하고 확장 가능한 플러그인을 통해 사용자 정의 데이터 수집
	- Kubernetes 버전 1.11, 1.12 및 1.13을 지원

## kubeadm으로 클러스터 설치

### VM 인스턴스 준비

먼저, ssh 접속이 가능한 VM 인스턴스를 2~3개 정도 준비한다. [설치 문서](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#before-you-begin)에서도 다음과 같이 명시하였지만, `kubeadm`의 경우 cpu 개수가 2개 이하일 경우 마스터 노드 초기화가 되지 않는다.  

> One or more machines running a deb/rpm-compatible OS, for example Ubuntu or CentOS
2 GB or more of RAM per machine. Any less leaves little room for your apps.
2 CPUs or more on the master
Full network connectivity among all machines in the cluster. A public or private network is fine.

IAM 롤은 인스턴스 생성 후에 추가 또는 변경할 수 있는데, 이 단계에서는 생략한다. 시큐리티 설정 화면에서 모든 VM들이 서로 연결될 수 있도록 AWS 기본 시큐리티 그룹과 SSH 접속을 위한 시큐리티 그룹을 각각 생성하여 추가한다. 이상, AWS 상에서 일반적인 EC2 인스턴스를 셋업하는 과정이다.

## 마스터 노드 셋업

생성된 인스턴스 하나를 마스터 노드로 셋업하고, 나머지는 워커 노드로 셋업한다. 마스터 노드의 셋업 과정은 다음과 같다.

### SSH 접속 

마스터 노드로 사용할 인스턴스에 ssh 접속한다.

`ssh -i ~/.ssh/id_rsa ubuntu@ec2-52-78-150-83.ap-northeast-2.compute.amazonaws.com`

### 도커 설치

다음으로 사용자 (`ubuntu`) 계정으로 도커를 설치한다. (참고로 도커 버전이 `18.06`이 아닌 최신 버전일 경우, `kubeadm init` 실행 시에 경고 메세지를 볼 수 있는데, 단순 테스트 클러스터이므로 무시하고 넘어간다.)

```sh
 sudo apt update
 sudo apt install -y apt-transport-https ca-certificates software-properties-common curl
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
 sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
 sudo apt update
 sudo apt install -y linux-image-extra-$(uname -r)
 sudo apt purge lxc-docker docker-engine docker.io
 sudo rm -rf /etc/default/docker
 sudo apt install -y docker-ce
 sudo service docker start
 sudo usermod -aG docker ${USER}
```

설치된 도커가 정상적으로 구동 중인지 확인해본다.

```sh
$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2019-02-21 10:59:36 UTC; 9min ago
     Docs: https://docs.docker.com
 Main PID: 3967 (dockerd)
    Tasks: 8
   CGroup: /system.slice/docker.service
           └─3967 /usr/bin/dockerd -H fd://

Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.367726467Z" level=warning msg="Your kernel does not support swap memory limit"
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.367918689Z" level=warning msg="Your kernel does not support cgroup rt period"
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.368057290Z" level=warning msg="Your kernel does not support cgroup rt runtime"
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.373962370Z" level=info msg="Loading containers: start."
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.495068635Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a preferred IP address"
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.550642079Z" level=info msg="Loading containers: done."
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.617021161Z" level=info msg="Docker daemon" commit=6247962 graphdriver(s)=overlay2 version=18.09.2
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.617452962Z" level=info msg="Daemon has completed initialization"
Feb 21 10:59:36 ip-172-31-12-57 systemd[1]: Started Docker Application Container Engine.
Feb 21 10:59:36 ip-172-31-12-57 dockerd[3967]: time="2019-02-21T10:59:36.647031257Z" level=info msg="API listen on /var/run/docker.sock"
```

### kubeadm, kubectl 설치

다음으로 `root` 계정으로 전환하여, `kubeadm`과 `kubectl`을 설치한다.

```sh
$ apt-get update && apt-get install -y apt-transport-https curl
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y kubelet kubeadm kubectl
$ apt-mark hold kubelet kubeadm kubectl
```

그리고 설치된 `kubelet` 서비스의 상태를 확인해보자. 
 
 ```sh
$ systemctl status kubelet.service
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Thu 2019-02-21 11:25:38 UTC; 6min ago
     Docs: https://kubernetes.io/docs/home/
 Main PID: 8119 (kubelet)
    Tasks: 16 (limit: 4704)
   CGroup: /system.slice/kubelet.service
           └─8119 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-con

Feb 21 11:31:49 ip-172-31-31-2 kubelet[8119]: W0221 11:31:49.001095    8119 cni.go:203] Unable to update cni config: No networks found in /etc/cni/net.d
Feb 21 11:31:49 ip-172-31-31-2 kubelet[8119]: E0221 11:31:49.001212    8119 kubelet.go:2192] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config unin
Feb 21 11:31:54 ip-172-31-31-2 kubelet[8119]: W0221 11:31:54.002280    8119 cni.go:203] Unable to update cni config: No networks found in /etc/cni/net.d
Feb 21 11:31:54 ip-172-31-31-2 kubelet[8119]: E0221 11:31:54.002852    8119 kubelet.go:2192] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config unin
Feb 21 11:31:59 ip-172-31-31-2 kubelet[8119]: W0221 11:31:59.003915    8119 cni.go:203] Unable to update cni config: No networks found in /etc/cni/net.d
Feb 21 11:31:59 ip-172-31-31-2 kubelet[8119]: E0221 11:31:59.004424    8119 kubelet.go:2192] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config unin
Feb 21 11:32:04 ip-172-31-31-2 kubelet[8119]: W0221 11:32:04.005500    8119 cni.go:203] Unable to update cni config: No networks found in /etc/cni/net.d
Feb 21 11:32:04 ip-172-31-31-2 kubelet[8119]: E0221 11:32:04.005984    8119 kubelet.go:2192] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config unin
Feb 21 11:32:09 ip-172-31-31-2 kubelet[8119]: W0221 11:32:09.007029    8119 cni.go:203] Unable to update cni config: No networks found in /etc/cni/net.d
```

### kubeadm 초기화

정상적으로 설치가 되었으면, `kubeadm init` 명령을 이용해 현재 노드를 마스터 노드로 초기화한다. 이 때, 호스트 네트워크와 Pod 네트워크가 중복되는 경우 `--pod-network-cidr` 옵션으로 Pod 네트워크를 위한 별도의 CIDR 값 (예. `10.10.0.0/16`) 등을 지정할 수 있다. `kubeadm init` 외에 다른 명령어들은 [여기](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/)를 참고한다. 

```sh
$ kubeadm init 
[init] Using Kubernetes version: v1.13.3
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.2. Latest validated version: 18.06
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [ip-172-31-31-2 localhost] and IPs [172.31.31.2 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [ip-172-31-31-2 localhost] and IPs [172.31.31.2 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ip-172-31-31-2 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.31.31.2]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 23.502070 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "ip-172-31-31-2" as an annotation
[mark-control-plane] Marking the node ip-172-31-31-2 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node ip-172-31-31-2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: 0kgx7r.fd4z6rmq6me24e5f
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.31.31.2:6443 --token 0kgx7r.fd4z6rmq6me24e5f --discovery-token-ca-cert-hash sha256:d7b433408e75b50b0722e7b0cdef9d0199fa8079de51e6def9c2effb763c72bb
```

초기화 과정의 로그 메세지들을 간략히 살펴보자.

#### 쿠버네티스 버전

먼저 설치되는 쿠버네티스의 버전은 `v1.13.3`이다.
 
```sh
[init] Using Kubernetes version: v1.13.3
```

#### 도커 버전

유효한 도커 버전은 `18.06` 인데, 버전이 맞지 않다는 경고 메세지를 볼 수 있다.

```sh
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.2. Latest validated version: 18.06
```

#### kubelet 서비스 활성화

`kubelet` 환경변수와 설정파일을 기록하고 `kubelet` 서비스를 시작한다.
```sh
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
```

#### 인증서 생성

다음으로 쿠버네티스 컴포넌트들이 시큐한 방식으로 통신하기 위한 인증서를 만든다. 보다시피, [Kubernetes the Hardway](https://github.com/kelseyhightower/kubernetes-the-hard-way)에서는 컴포넌트마다 일일히 인증서를 만들어주고 등록해줘야 하는 번거로운 작업이 `kubeadm`에서는 간단하게 완료된다.

```sh
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [ip-172-31-31-2 localhost] and IPs [172.31.31.2 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [ip-172-31-31-2 localhost] and IPs [172.31.31.2 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ip-172-31-31-2 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.31.31.2]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
```

#### kubeconfig 생성

그리고 API 서버와 통신하기 위한 `kubeconfig` 들을 생성한다.

```sh
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
```

#### 정적 Pod 실행

컨트롤 플레인의 모든 컴포넌트들의 시작점인 `/etc/kubernetes/manifests`를 기준으로 정적 Pod를 실행한다. `kubectl` 은 두 가지 모드로 동작하는데, 한 가지는 Kubernetes API 서버에 대한 원격 에이전트 모드이고, 다른 하나는 정적 Pod 모드로 다른 원격 서버의 파일 시스템이 아닌 로컬디스크의 파일시스템으로 정비된 Pod을 실행되는 모드이다.
 
```sh
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
```

참고로 `kube-apiserver`의 yaml 파일의 내용은 다음과 같이 단순한 하나의 Pod로 구성되어 있다. 즉, 쿠버네티스 컨트롤 플레인 역시도 일반 애플리케이션들처럼 쿠버네티스 컴포넌트로 생성되고 관리되고 있다는 점을 알 수 있다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=172.31.31.2
    - --allow-privileged=true
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver:v1.13.3
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 172.31.31.2
        path: /healthz
        port: 6443
        scheme: HTTPS
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: kube-apiserver
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
    - mountPath: /etc/ca-certificates
      name: etc-ca-certificates
      readOnly: true
    - mountPath: /etc/kubernetes/pki
      name: k8s-certs
      readOnly: true
    - mountPath: /usr/local/share/ca-certificates
      name: usr-local-share-ca-certificates
      readOnly: true
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  - hostPath:
      path: /etc/kubernetes/pki
      type: DirectoryOrCreate
    name: k8s-certs
  - hostPath:
      path: /usr/local/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-local-share-ca-certificates
  - hostPath:
      path: /usr/share/ca-certificates
      type: DirectoryOrCreate
    name: usr-share-ca-certificates
status: {}
```


### 컨트롤 플레인 셋업 완료

컨트롤 플레인의 헬스체크가 끝나고, 클러스터 설정 정보인 `kubeadm-config` ConfigMap을 `kube-system` 네임스페이스에 저장한다. `/var/run/dockershim.sock` CRI 소켓 정보를 주석으로 등록하고 컨트롤 플레인 노드를 마킹하여 마스터 노드 셋업을 완료된다.

```sh
[apiclient] All control plane components are healthy after 23.502070 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "ip-172-31-31-2" as an annotation
[mark-control-plane] Marking the node ip-172-31-31-2 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node ip-172-31-31-2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
```

### RBAC 및 기타

그리고 부트스트랩 토큰을 생성하고, 생성된 토큰으로 RBAC 룰을 구성한다. 그리고 클러스터 정보를 `kube-public` 네임스페이스에 ConfigMap 컴포넌트로 등록한다.

```sh
[bootstrap-token] Using token: 0kgx7r.fd4z6rmq6me24e5f
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
```

이후, EC2 다른 워커 노드에서 위 ConfigMap의 컴포넌트를 다음과 같이 확인할 수 있다.

```sh
$ curl --insecure  https://172.31.31.2:6443/api/v1/namespaces/kube-public/configmaps/cluster-info
{
  "kind": "ConfigMap",
  "apiVersion": "v1",
  "metadata": {
    "name": "cluster-info",
    "namespace": "kube-public",
    "selfLink": "/api/v1/namespaces/kube-public/configmaps/cluster-info",
    "uid": "74cce7cf-35cb-11e9-b7a0-0a1483891ff6",
    "resourceVersion": "324",
    "creationTimestamp": "2019-02-21T11:25:56Z"
  },
  "data": {
    "jws-kubeconfig-0kgx7r": "eyJhbGciOiJIUzI1NiIsImtpZCI6IjBrZ3g3ciJ9..K6bdmWXxnjfxLXzzRpWMCGdYxmJAyP9fYEfHCDm8N7E",
    "kubeconfig": "apiVersion: v1\nclusters:\n- cluster:\n    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNU1ESXlNVEV4TWpVeU9Wb1hEVEk1TURJeE9ERXhNalV5T1Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS1RjCjJmSWVYS1pyMEJhbS9mUnVRdWpyQ3FUOXlRRTJGckRNcGNvK3pyNi9ZaGJkU3hYM3A4OE1tZUduK2hZVUluOGEKamhEejJ0VTlHMldaQm5PTFd6dmFjZkhqYUw4ZGJya2FUYUo5NjM4SHVuVkhxQmNMR3lySnFLY203Z2xYdm5rRApxazVES1l2NFJObzJHRnRuelFTNEdXdGhhbXk1QlBadiszQzdRWVlsRDRKWWpYaWdhak5GNFpHOU1INENwNlRICnhrbDRJZ1FqTzJ3aTJOeEE5TDNpV2VZRnZEMk0zUmppNzhKSkFuN3lKREZlR0NtRHYycGE3ZHprcUJhdFAzTWEKWUozbWpUcndaOGF1eDNDa0FNNDltamM2ZEJ5WkpkSFY4a2xTSUNtb2NUK0g5REtnUWdTZXV5bitXZm8wNkg0cQpZT3FqOVYvam9peXMzcHZaTXE4Q0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFERmJlbXV1NXhZNVVMNi9EU0VrK1UyU1V0SWwKQ3JLVEJDc2ZiVkFaNVlGQUdHYi96RFFNb0RTc1pQSVJ5aGdHNGhYMVdnZWV1YWo3aXBSNEt1VHNXc2V0Q2ZpeQpiMjZVMjl6SHFsUlZEN2JtYVk0OENpbURLTGdvbWZKczZmY20xTWdSL1FvemM4QUFlUUZOWEpoaG13V2g5alVXCkpBem85eFVRQWwxSW0rNldXcllrRzk3TkNwMkYxdFFGMGZtcEFjTWQ1cXErdXEwcmdQVmt0QkRkMUNieHdidFcKK25Ea1FjZW4wbGZSNUo3U0NkME45SUFWc2l6RTNYYmxiNmRsMTU2S1NNaEZRb1lZTm8xR0djSkpvd3RxL1FXTgpGcHovaDRLSGdxQnhPVjBVb0ZXYThCaHJFcTduRCtlaUhNYU9ycmgxNTdqS01rME94eXZMcDBLWC9FUT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=\n    server: https://172.31.31.2:6443\n  name: \"\"\ncontexts: []\ncurrent-context: \"\"\nkind: Config\npreferences: {}\nusers: []\n"
  }
}
```

#### 필수 Addon 설치

마지막으로 필수 애드온인 `CoreDNS`와 `kube-proxy`를 설치한다. 참고로 Pod 네트워크가 셋업되기 전이므로 `CoreDNS` 서비스 등 클러스터 네트워킹 서비스가 정상적으로 시작되지는 않는다.

```sh
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
```

#### 노드 조인 명령

그리고 마스터 노드 초기화 완료시에 최종 출력된, 토큰값과 명령어를 기억해 둔다.

```sh
You can now join any number of machines by running the following on each node
as root:
  kubeadm join 172.31.31.2:6443 --token 0kgx7r.fd4z6rmq6me24e5f --discovery-token-ca-cert-hash sha256:d7b433408e75b50b0722e7b0cdef9d0199fa8079de51e6def9c2effb763c72bb
 ```

### cluster 조회

마스터 노드가 생성되었으므로, `ubuntu` 사용자 계정으로 전환해서 `kubelet`을 실행해보자. 이를 위해선 사용자 홈디렉토리에 `kubeconfig` 파일을 다음과 같이 셋업해주어야 한다. 

 ```sh
 $ mkdir -p $HOME/.kube
 $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
 ```
 
또는 사용자 계정에서 단순히 `KUBECONFIG` 환경변수를 설정해주어도 무방하다.

```sh
$ export KUBECONFIG=/etc/kubernetes/admin.conf
```
 
이제 클러스터 노드 정보를 조회해보자. `kubelet`이 정상적으로 실행되고 있고, 마스터 노드가 `NotReady` 상태임을 알 수 있다.

```sh
$ kubectl get node
NAME             STATUS     ROLES    AGE     VERSION
ip-172-31-31-2   NotReady   master   9m48s   v1.13.3
```

## 워커 노드 추가

### 토큰 확인

워커 노드를 추가하기 이전에, 마스터 노드에서 토큰 목록을 확인해보자. 보다시피 생성된 `0kgx7r.fd4z6rmq6me24e5f` 토큰의 만료시간은 24시간이다. 이 토큰을 이용해 워커 노드를 구성할 수 있다.

```sh
root@ip-172-31-31-2:~# kubeadm token list
TOKEN                     TTL       EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
0kgx7r.fd4z6rmq6me24e5f   23h       2019-02-22T11:25:56Z   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

### 도커 및 kubeadm, kubelet 설치

먼저, 워커 노드로 사용할 EC2 인스턴스에 ssh 로그인하여 마스터 노드와 동일하게 도커 및 kubeadm, kubelet을 설치해준다. (동일한 과정이므로 생략)

```sh
$ ssh -i ~/.ssh/path/to/pem ubuntu@ec2-54-180-103-119.ap-northeast-2.compute.amazonaws.com
```

그리고 워커 노드에 kubeadm 설치가 완료되었으면, 마스터 노드 생성시에 출력된 토큰값과 명령어를 이용해 워커 노드에서 마스터 노드로 조인한다. 보다시피, 명령어 자체는 `swarn join`과 유사하다.

```sh
root@ip-172-31-22-155:~# kubeadm join 172.31.31.2:6443 --token 0kgx7r.fd4z6rmq6me24e5f --discovery-token-ca-cert-hash sha256:d7b433408e75b50b0722e7b0cdef9d0199fa8079de51e6def9c2effb763c72bb
[preflight] Running pre-flight checks
        [WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.2. Latest validated version: 18.06
[discovery] Trying to connect to API Server "172.31.31.2:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://172.31.31.2:6443"
[discovery] Requesting info from "https://172.31.31.2:6443" again to validate TLS against the pinned public key
[discovery] Cluster info signature and contents are valid and TLS certificate validates against pinned roots, will use API Server "172.31.31.2:6443"
[discovery] Successfully established connection with API Server "172.31.31.2:6443"
[join] Reading configuration from the cluster...
[join] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet] Downloading configuration for the kubelet from the "kubelet-config-1.13" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "ip-172-31-22-155" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

### 마스터 노드 확인

마스터 노드 쉘에서 클러스터 노드 정보를 조회하면 다음과 같이 `ip-172-31-22-155` 노드 하나가 추가된 것을 확인할 수 있다. 그러나 네트워킹이 셋업되지 않았기 때문에, Pod과 다른 컴포넌트들간의 통신이 불가능한 `NotReady` 상태이다. 

```sh
ubuntu@ip-172-31-31-2:~$ kubectl get node
NAME               STATUS     ROLES    AGE     VERSION
ip-172-31-22-155   NotReady   <none>   3m36s   v1.13.3
ip-172-31-31-2     NotReady   master   18m     v1.13.3
```

### Pod Network 설정

이제 Pod Network 애드온을 설치해서 Pod가 서로 통신할 수 있도록 한다. 참고로 `kubeadm`은 CNI (Container Network Interface) 기반 네트워크만 지원하며 kubenet을 지원하지 않는다. [여기](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network)를 참조해서 `weave-net` 데몬셋을 설치한다.
 
 ```sh
 ubuntu@ip-172-31-31-2:~$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
serviceaccount/weave-net created
clusterrole.rbac.authorization.k8s.io/weave-net created
clusterrolebinding.rbac.authorization.k8s.io/weave-net created
role.rbac.authorization.k8s.io/weave-net created
rolebinding.rbac.authorization.k8s.io/weave-net created
daemonset.extensions/weave-net created
```

설치가 되고 나면, 마스트 노드 쉘에서 다음과 같이 모든 노드가 `Ready` 상태로 전환됨을 확인할 수 있다.

```sh
ubuntu@ip-172-31-31-2:~$ kubectl get node
NAME               STATUS   ROLES    AGE   VERSION
ip-172-31-22-155   Ready    <none>   11m   v1.13.3
ip-172-31-31-2     Ready    master   26m   v1.13.3
```

참고로, `weave-net`의 데몬셋 yaml 파일은 다음과 같다.

```yaml
ubuntu@ip-172-31-31-2:~$ kubectl get ds weave-net -n kube-system --export -o yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  annotations:
    cloud.weave.works/launcher-info: |-
      {
        "original-request": {
          "url": "/k8s/v1.10/net.yaml?k8s-version=Q2xpZW50IFZlcnNpb246IHZlcnNpb24uSW5mb3tNYWpvcjoiMSIsIE1pbm9yOiIxMyIsIEdpdFZlcnNpb246InYxLjEzLjMiLCBHaXRDb21taXQ6IjcyMWJmYTc1MTkyNGRhOGQxNjgwNzg3NDkwYzU0YjkxNzliMWZlZDAiLCBHaXRUcmVlU3RhdGU6ImNsZWFuIiwgQnVpbGREYXRlOiIyMDE5LTAyLTAxVDIwOjA4OjEyWiIsIEdvVmVyc2lvbjoiZ28xLjExLjUiLCBDb21waWxlcjoiZ2MiLCBQbGF0Zm9ybToibGludXgvYW1kNjQifQpTZXJ2ZXIgVmVyc2lvbjogdmVyc2lvbi5JbmZve01ham9yOiIxIiwgTWlub3I6IjEzIiwgR2l0VmVyc2lvbjoidjEuMTMuMyIsIEdpdENvbW1pdDoiNzIxYmZhNzUxOTI0ZGE4ZDE2ODA3ODc0OTBjNTRiOTE3OWIxZmVkMCIsIEdpdFRyZWVTdGF0ZToiY2xlYW4iLCBCdWlsZERhdGU6IjIwMTktMDItMDFUMjA6MDA6NTdaIiwgR29WZXJzaW9uOiJnbzEuMTEuNSIsIENvbXBpbGVyOiJnYyIsIFBsYXRmb3JtOiJsaW51eC9hbWQ2NCJ9Cg==",
          "date": "Thu Feb 21 2019 11:51:45 GMT+0000 (UTC)"
        },
        "email-address": "support@weave.works"
      }
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"extensions/v1beta1","kind":"DaemonSet","metadata":{"annotations":{"cloud.weave.works/launcher-info":"{\n  \"original-request\": {\n    \"url\": \"/k8s/v1.10/net.yaml?k8s-version=Q2xpZW50IFZlcnNpb246IHZlcnNpb24uSW5mb3tNYWpvcjoiMSIsIE1pbm9yOiIxMyIsIEdpdFZlcnNpb246InYxLjEzLjMiLCBHaXRDb21taXQ6IjcyMWJmYTc1MTkyNGRhOGQxNjgwNzg3NDkwYzU0YjkxNzliMWZlZDAiLCBHaXRUcmVlU3RhdGU6ImNsZWFuIiwgQnVpbGREYXRlOiIyMDE5LTAyLTAxVDIwOjA4OjEyWiIsIEdvVmVyc2lvbjoiZ28xLjExLjUiLCBDb21waWxlcjoiZ2MiLCBQbGF0Zm9ybToibGludXgvYW1kNjQifQpTZXJ2ZXIgVmVyc2lvbjogdmVyc2lvbi5JbmZve01ham9yOiIxIiwgTWlub3I6IjEzIiwgR2l0VmVyc2lvbjoidjEuMTMuMyIsIEdpdENvbW1pdDoiNzIxYmZhNzUxOTI0ZGE4ZDE2ODA3ODc0OTBjNTRiOTE3OWIxZmVkMCIsIEdpdFRyZWVTdGF0ZToiY2xlYW4iLCBCdWlsZERhdGU6IjIwMTktMDItMDFUMjA6MDA6NTdaIiwgR29WZXJzaW9uOiJnbzEuMTEuNSIsIENvbXBpbGVyOiJnYyIsIFBsYXRmb3JtOiJsaW51eC9hbWQ2NCJ9Cg==\",\n    \"date\": \"Thu Feb 21 2019 11:51:45 GMT+0000 (UTC)\"\n  },\n  \"email-address\": \"support@weave.works\"\n}"},"labels":{"name":"weave-net"},"name":"weave-net","namespace":"kube-system"},"spec":{"minReadySeconds":5,"template":{"metadata":{"labels":{"name":"weave-net"}},"spec":{"containers":[{"command":["/home/weave/launch.sh"],"env":[{"name":"HOSTNAME","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"spec.nodeName"}}}],"image":"docker.io/weaveworks/weave-kube:2.5.1","name":"weave","readinessProbe":{"httpGet":{"host":"127.0.0.1","path":"/status","port":6784}},"resources":{"requests":{"cpu":"10m"}},"securityContext":{"privileged":true},"volumeMounts":[{"mountPath":"/weavedb","name":"weavedb"},{"mountPath":"/host/opt","name":"cni-bin"},{"mountPath":"/host/home","name":"cni-bin2"},{"mountPath":"/host/etc","name":"cni-conf"},{"mountPath":"/host/var/lib/dbus","name":"dbus"},{"mountPath":"/lib/modules","name":"lib-modules"},{"mountPath":"/run/xtables.lock","name":"xtables-lock"}]},{"env":[{"name":"HOSTNAME","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"spec.nodeName"}}}],"image":"docker.io/weaveworks/weave-npc:2.5.1","name":"weave-npc","resources":{"requests":{"cpu":"10m"}},"securityContext":{"privileged":true},"volumeMounts":[{"mountPath":"/run/xtables.lock","name":"xtables-lock"}]}],"hostNetwork":true,"hostPID":true,"restartPolicy":"Always","securityContext":{"seLinuxOptions":{}},"serviceAccountName":"weave-net","tolerations":[{"effect":"NoSchedule","operator":"Exists"}],"volumes":[{"hostPath":{"path":"/var/lib/weave"},"name":"weavedb"},{"hostPath":{"path":"/opt"},"name":"cni-bin"},{"hostPath":{"path":"/home"},"name":"cni-bin2"},{"hostPath":{"path":"/etc"},"name":"cni-conf"},{"hostPath":{"path":"/var/lib/dbus"},"name":"dbus"},{"hostPath":{"path":"/lib/modules"},"name":"lib-modules"},{"hostPath":{"path":"/run/xtables.lock","type":"FileOrCreate"},"name":"xtables-lock"}]}},"updateStrategy":{"type":"RollingUpdate"}}}
  creationTimestamp: null
  generation: 1
  labels:
    name: weave-net
  name: weave-net
  selfLink: /apis/extensions/v1beta1/namespaces/kube-system/daemonsets/weave-net
spec:
  minReadySeconds: 5
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: weave-net
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: weave-net
    spec:
      containers:
      - command:
        - /home/weave/launch.sh
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: docker.io/weaveworks/weave-kube:2.5.1
        imagePullPolicy: IfNotPresent
        name: weave
        readinessProbe:
          failureThreshold: 3
          httpGet:
            host: 127.0.0.1
            path: /status
            port: 6784
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 10m
        securityContext:
          privileged: true
          procMount: Default
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /weavedb
          name: weavedb
        - mountPath: /host/opt
          name: cni-bin
        - mountPath: /host/home
          name: cni-bin2
        - mountPath: /host/etc
          name: cni-conf
        - mountPath: /host/var/lib/dbus
          name: dbus
        - mountPath: /lib/modules
          name: lib-modules
        - mountPath: /run/xtables.lock
          name: xtables-lock
      - env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: docker.io/weaveworks/weave-npc:2.5.1
        imagePullPolicy: IfNotPresent
        name: weave-npc
        resources:
          requests:
            cpu: 10m
        securityContext:
          privileged: true
          procMount: Default
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /run/xtables.lock
          name: xtables-lock
      dnsPolicy: ClusterFirst
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        seLinuxOptions: {}
      serviceAccount: weave-net
      serviceAccountName: weave-net
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        operator: Exists
      volumes:
      - hostPath:
          path: /var/lib/weave
          type: ""
        name: weavedb
      - hostPath:
          path: /opt
          type: ""
        name: cni-bin
      - hostPath:
          path: /home
          type: ""
        name: cni-bin2
      - hostPath:
          path: /etc
          type: ""
        name: cni-conf
      - hostPath:
          path: /var/lib/dbus
          type: ""
        name: dbus
      - hostPath:
          path: /lib/modules
          type: ""
        name: lib-modules
      - hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
        name: xtables-lock
  templateGeneration: 1
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
status:
  currentNumberScheduled: 0
  desiredNumberScheduled: 0
  numberMisscheduled: 0
  numberReady: 0
```

### 로컬 kubeconfig 셋업

싱글 마스터 노드와 싱글 워커 노드로 구성된 쿠버네티스 클러스터가 준비되었다. 이제 로컬 PC에서 위 클러스터를 사용해보자. 이를 위해서 마스터 노드의 `kubeconfig` 파일을 로컬에 캐시한다.

#### kubeconfig 로컬 복사

```sh
$ scp -i ~/.ssh/path/to/pem ubuntu@ec2-13-209-88-210.ap-northeast-2.compute.amazonaws.com:/home/ubuntu/.kube/config kubeconfig
$ export KUBECONFIG=kubeconfig
```

다음으로 `kubeconfig` 파일을 열어서 마스터 API 서버의 주소인 `https://172.31.31.2:6443`를 `https://kubernetes:6443`로 변경한다. 

#### kubernetes 호스트 등록

그리고 다음과 같이 `/etc/hosts` 파일을 열어서 kubernetes를 로컬 주소로 등록한다.

```sh
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
127.0.0.1 kubernetes
255.255.255.255	broadcasthost
::1             localhost
```

#### 6443 로컬 바인딩 

마스터 API 서버의 시큐어 포트 6443을 로컬 포트 6443에 바인딩한다.

```sh
ssh ubuntu@ec2-13-209-88-210.ap-northeast-2.compute.amazonaws.com -L 6443:localhost:6443
```

#### 로컬에서 kubelet 실행
 
그리면, 다음과 같이 로컬 PC에서도 동일하게 클러스터 노드 조회 정보를 확인할 수 있다.

```sh
$ kubectl get node
NAME               STATUS   ROLES    AGE   VERSION
ip-172-31-22-155   Ready    <none>   31m   v1.13.3
ip-172-31-31-2     Ready    master   36m   v1.13.3
```

## HA 구성

`kubeadm`을 이용한 고가용성 컨트롤 플레인 HA 구성에 대한 문서와 오픈 소스 예제는 다음을 참고한다.

- https://kubernetes.io/docs/setup/independent/ha-topology/
- https://kubernetes.io/docs/setup/independent/setup-ha-etcd-with-kubeadm/
- https://github.com/cookeem/kubeadm-ha
- https://github.com/Lentil1016/kubeadm-ha
- https://github.com/scholzj/terraform-aws-kubernetes