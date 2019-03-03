+++
date          = "2019-03-03T13:25:00+09:00"
draft         = false
title         = "TGI Kubernetes 007: Controller 만들기"
tags          = ["Kubernetes", "controller"]
categories    = ["tgik"]
slug          = "tgik-007"
notoc         = true
socialsharing = true
nocomment     = false
+++

[일곱 번째 에피소드](https://github.com/heptio/tgik/blob/master/episodes/007/README.md)는 간단하게 쿠버네티스 컨트롤러를 만들어 보는 데모이다.

데모 리포지토리가 오래되고 API 패키지명이 다르기도 하고 하여, `go mod`로 재작성하였다. 저장소에는 각 데모에 해당하는 컨트롤러의 소스가 각ㄲ `pod-watcher`와 `secret-controller`의 두 가지로 태깅되어있다.

- [ziwon/k8s-controller](https://github.com/ziwon/k8s-controller)

쿠버네티스 클러스터는 [kind](https://github.com/kubernetes-sigs/kind)를 사용하여 테스트하였다.

- [ziwon/yak8s](https://github.com/ziwon/yak8s)

## Pod-Watcher

먼저, 적당한 디렉토리에 저장소를 클론한다. 그리고 `pod-watcher`를 체크 아웃한다. (`go mod`에 대한 자세한 설명은 여기서는 생략합니다.)

```sh
$ export GO111MODULE=auto
$ git clone git@github.com:ziwon/k8s-controller.git
$ git checkout pod-watcher
```

다음으로, 로컬 PC 환경에서 쿠버네티스 클러스터를 띄운다.
```sh
$ git clone https://github.com/ziwon/yak8s
$ make kind-cluster-up
Creating cluster 'kind-1' ...
 ✓ Ensuring node image (kindest/node:v1.13.2) 🖼
 ✓ [lb] Creating node container 📦
 ✓ [lb] Fixing mounts 🗻
 ✓ [lb] Starting systemd 🖥
 ✓ [lb] Waiting for docker to be ready 🐋
 ✓ [lb] Pre-loading images 🐋
 ✓ [control-plane1] Creating node container 📦
 ✓ [control-plane1] Fixing mounts 🗻
 ✓ [control-plane1] Starting systemd 🖥
 ✓ [control-plane1] Waiting for docker to be ready 🐋
 ✓ [control-plane1] Pre-loading images 🐋
 ...
 ✓ [lb] Starting the external load balancer ⛵
 ✓ [control-plane1] Creating the kubeadm config file ⛵
 ✓ [control-plane1] Starting Kubernetes (this may take a minute) ☸
 ✓ [control-plane2] Joining control-plane node to Kubernetes ☸
 ✓ [worker1] Joining worker node to Kubernetes ☸
 ✓ [worker2] Joining worker node to Kubernetes ☸
 ✓ [worker3] Joining worker node to Kubernetes ☸
 ✓ [worker4] Joining worker node to Kubernetes ☸
Cluster creation complete. You can now use the cluster with:

export KUBECONFIG="$(kind get kubeconfig-path --name="1")"
kubectl cluster-info
/Library/Developer/CommandLineTools/usr/bin/make kind-cluster-info
Kubernetes master is running at https://localhost:63578
KubeDNS is running at https://localhost:63578/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

클러스터를 띄웠으니, 컨트롤러가 사용할 `KUBECONFIG` 환경변수를 잡아준다.

```sh
$ export KUBECONFIG="$(kind get kubeconfig-path --name="1")"
```

이제 컨트롤러를 실행시켜보자. 다음과 같이 클러스터 내의 Pod 컴포넌트 정보들이 컨트롤러 캐시에 싱크되고 있는 것을 확인할 수 있다.

```sh
$ cd ziwon/k8s-controller
$ git checkout pod-watcher
$ go run *.go
Alias tip: gor *.go
2019/03/03 15:12:15 k8s-controller version UNKNOWN
2019/03/03 15:12:15 waiting for cache sync
2019/03/03 15:12:15 onAdd: kube-system/weave-net-ctjpn
2019/03/03 15:12:15 onAdd: kube-system/kube-apiserver-kind-1-control-plane1
2019/03/03 15:12:15 onAdd: kube-system/kube-proxy-mkvq5
2019/03/03 15:12:15 onAdd: kube-system/kube-scheduler-kind-1-control-plane2
2019/03/03 15:12:15 onAdd: kube-system/kube-proxy-r88qg
2019/03/03 15:12:15 onAdd: kube-system/kube-proxy-gs25r
2019/03/03 15:12:15 onAdd: kube-system/weave-net-x2jcn
2019/03/03 15:12:15 onAdd: kube-system/weave-net-qpw2p
2019/03/03 15:12:15 onAdd: kube-system/kube-proxy-qxct5
2019/03/03 15:12:15 onAdd: kube-system/etcd-kind-1-control-plane1
2019/03/03 15:12:15 onAdd: kube-system/coredns-86c58d9df4-w8s8q
2019/03/03 15:12:15 onAdd: kube-system/etcd-kind-1-control-plane2
2019/03/03 15:12:15 onAdd: kube-system/kube-apiserver-kind-1-control-plane2
2019/03/03 15:12:15 onAdd: kube-system/kube-scheduler-kind-1-control-plane1
2019/03/03 15:12:15 onAdd: kube-system/kube-controller-manager-kind-1-control-plane2
2019/03/03 15:12:15 onAdd: kube-system/kube-controller-manager-kind-1-control-plane1
2019/03/03 15:12:15 onAdd: kube-system/kube-proxy-lrjws
2019/03/03 15:12:15 onAdd: kube-system/weave-net-rpfnw
2019/03/03 15:12:15 onAdd: kube-system/coredns-86c58d9df4-8g2s2
2019/03/03 15:12:15 onAdd: kube-system/kube-proxy-tw8ld
2019/03/03 15:12:15 onAdd: kube-system/weave-net-ljdnm
2019/03/03 15:12:15 onAdd: kube-system/weave-net-vl2nw
2019/03/03 15:12:15 caches are synced
2019/03/03 15:12:15 waiting for stop signal
```

클러스터에 Pod를 하나 배포한다.

```sh
make run-k8s
kubectl run --restart=Never --image=gcr.io/kuar-demo/kuard-amd64:blue kuard
pod/kuard created
```

다음과 같이 컨트롤러에서 새로 배포된 Pod가 추가됨을 확인할 수 있다.

```sh
2019/03/03 15:13:30 onAdd: default/kuard
2019/03/03 15:13:30 onUpdate: default/kuard
2019/03/03 15:13:30 onUpdate: default/kuard
2019/03/03 15:13:36 onUpdate: default/kuard
```

이제 배포된 Pod을 클러스터에서 제거하면, 역시 삭제되었다는 이벤트를 컨트롤러를 통해 수신하게 된다.

```sh
$ kubectl delete po kuard
```

```sh
2019/03/03 15:16:19 onUpdate: default/kuard
2019/03/03 15:16:19 onDelete: default/kuard
```

이상, 컨트롤러의 동작을 간단히 확인해 보았다. 클러스터에는 여러 개의 컨트롤러를 붙일 수 있으며, 단지 각각의 컨트롤러가 동일한 작업을 하지 않으면 문제 없다고 한다. 또한 컨트롤러는 여러 언어로 된 라이브러리가 있기 때문에, 굳이 Go언어로 작성될 필요는 없다고 한다.

반응형 컨트롤러를 작성하기 위한 여러가지 유틸리티와 라이브러리가 있으며, 한 가지 베스트 프랙티스 중 하나는 Queue를 사용하는 것이다. 클러스터 자체가 매우 bursty하고 여러 가지 변경이 매우 많기 때문에, 클러스터에 대한 모든 변경을 하나씩 일일히 처리하기 보다는 비슷한 종류의 변경들은 하나로 묶어서 큐를 통해 처리하게 되면, rate limit을 사용해서 클러스터 변경에 대해 서버가 얼마나 세게 치는지 알 수 있다고 한다. 이를 위해 쿠버네티스 클라이언트에서 **[Workqueue](https://github.com/kubernetes/client-go/tree/master/examples/workqueue)**를 제공하고 있다. 

### Pod-Watcher 코드 설명

이제 코드를 간단히 살펴보자.

다음은 `k8s-controller.go` 파일에서 설정 파일을 잡아주는 부분으로,

```go
	if kubeconfig != "" {
		config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
	} else {
		config, err = rest.InClusterConfig()
	}
```

위 코드는 쿠버네티스 클라이언트를 `KUBECONFIG` 환경변수 또는 `kubeconfig` 플래그를 통해서 설정하거나 또는 만약 컨트롤러가 쿠버네티스 클러스터 내에 있다면, 기본값으로 restClient를 설정하게 된다.

다음으로 설정된 환경변수에 대해 새로운 클라이언트를 생성한다.

```go
client := kubernetes.NewForConfigOrDie(config)
```

그리고 `sharedInformer`를 생성하는데, 쿠버네티스 오브젝트에 대한 미러를 캐시하다가 특정 오브젝트의 변경이 있으면 알려주는 주는 역할을 한다. 쿠버네티스 내에서는 각각 자신들만의 컨트롤러가 쿠버네티스 객체에 대해 미러를 유지하는데, 이는 매우 큰 클러스터의 경우 매우 메모리 집약적인 작업이다. 따라서 여러 컨트롤러가 공유해서 사용하는 뭔가가 필요한데, `SharedInformer`가 그 역할을 한다고 볼 수 있다.

```go
	sharedInformers := informers.NewSharedInformerFactory(client, 10*time.Minute)
	k8sController := NewK8SController(client, sharedInformers.Core().V1().Pods())
```

이제, `controller.go`의 소스를 살펴보자.

먼저 `podGetter`는 쿠버네티스 클라이언트를 통해 해당 Pod 정보를 획득하고, `podLister`는 `SharedInformer`의 캐시를 통해 Pod 목록을 획득한다.

```go
type K8SController struct {
	podGetter       corev1.PodsGetter
	podLister       listercorev1.PodLister
	podListerSynced cache.InformerSynced
}
```

다음으로 `SharedInformer`에 이벤트 핸들러를 추가한다. 이 부분은 이벤트 옵저버 패턴의 일반적인 형식으로 볼 수 있다.

```go
	podInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				c.onAdd(obj)
			},
			UpdateFunc: func(oldObj, newObj interface{}) {
				c.onUpdate(oldObj, newObj)
			},
			DeleteFunc: func(obj interface{}) {
				c.onDelete(obj)
			},
		},
	)
```

그리고, `onAdd`, `onUpdate`, `onDelete` 등 각각의 이벤트 핸들러를 구현한다. `cache.MetaNamespaceKeyFunc`는 `default/kuard`에서처럼, 네임스페이스와 객체의 이름을 바인딩한 값을 캐쉬에 대한 키로 가져온다.

```go
func (c *K8SController) onAdd(obj interface{}) {
	log.Printf("onAdd: object: %#v: %v", obj)
	key, err := cache.MetaNamespaceKeyFunc(obj)
	if err != nil {
		log.Printf("onAdd: error getting key for: %#v: %v", obj, err)
		runtime.HandleError(err)
	}
	log.Printf("onAdd: %v", key)
}
```

다음은 캐시 싱크를 수행하는 함수이다. 완료될 때까지 기다리고, 만약 큐를 이용한다면, 싱크 전에 큐를 로딩해주고 워커를 실행해야한다. 

```go
func (c *K8SController) Run(stop <-chan struct{}) {
	log.Print("waiting for cache sync")
	if !cache.WaitForCacheSync(stop, c.podListerSynced) {
		log.Print("timed out waiting for cache sync")
		return
	}
	log.Print("caches are synced")

	// wait until we're told to stop
	log.Print("waiting for stop signal")
	<-stop
	log.Print("received stop signal")
}
```

## 시크릿 컨트롤러

다음으로 작성해 볼 컨트롤러는 시크릿 컨트롤러이다.

예를 들어, 여러 개발자가 사용하는 쿠버네티스 클러스터를 셋업했고, 네임스페이스마다 개발자를 할당하여 뭔가를 한다고 할 때, 공유 데이터베이스를 사용하지 않고 개발자들에게 쿠버네티스 시크릿을 모든 개발자들의 네임스페이스에 전파하는 것이 이슈가 될 수 있다. 이를 해결하기 위해 이를테면, 일종의 시크릿을 배포하는 컨트롤러를 만들 수 있을 것이다. 


자세한 소스 코드는 `secret-controller` 태그를 참고한다. 

```sh
$ git checkout secret-controller
```

소스를 보면, Secret 컴포넌트에 대한 이벤트를 핸들링하는 **[handleSecretChange(obj interface{})](https://github.com/ziwon/k8s-controller/blob/secret-controller/controller.go#L110)** 컨트롤러 메소드가 추가되었다. 

```go
func (c *K8SController) handleSecretChange(obj interface{}) {
	secret, ok := obj.(*apicorev1.Secret)
	if !ok {
		// TODO: this is probably a `DeletedFinalStateUnknown`.  Figure out what
		// to do.
		return
	}

	if secret.ObjectMeta.Namespace != secretSyncSourceNamespace {
		log.Printf("Skipping secret in wrong namespace")
		return
	}

	if secret.Type != secretSyncType {
		log.Printf("Skipping secret of wrong type")
		return
	}

	log.Printf("Do something with this secret")
	nsList, err := c.namespaceGetter.Namespaces().List(metav1.ListOptions{})

	if err != nil {
		log.Printf("Error listeing namespaces: %v", err)
		return
	}

	for _, ns := range nsList.Items {
		nsName := ns.ObjectMeta.Name
		if _, ok := namespaceBlacklist[nsName]; ok {
			log.Printf("Skipping namespace on blacklist: %v", nsName)
			continue
		}
		log.Printf("We should copy %s to namespace %s", secret.ObjectMeta.Name, ns.ObjectMeta.Name)
		c.copySecretToNamespace(secret, nsName)
	}
}
```

시크릿을 복사할 때, `kube-public`, `kube-system` 네임스페이스는 제외한 다른 네임스페이스에 복사하는 작업을 수행하고 있다. 실제 시크릿을 복사하는 코드는 시간관계상 생략되었지만, 그 구현 내용은 다음과 같이 주석을 참고한다.

```go
func (c *K8SController) copySecretToNamespace(secret *apicorev1.Secret, nsName string) {
	// TODO:
	// 1. Make a deep copy of the secret
	// 2. Remove things like object version that'll prevent us from writing
	// 3. Write in new namespace
	// 4. Do a create or update for the new object
}
```

이제, 클러스터에 `secretsync`라는 네임스페이스를 만든다.

```sh
$ kubectl create namespace secretsync
```

시크릿을 만드는 방법과 관련된 옵션은 다음과 같이 확인할 수 있다.

```sh
$ kubectl create --namespace=secretsync secret generic --help
Create a secret based on a file, directory, or specified literal value.

A single secret may package one or more key/value pairs.

When creating a secret based on a file, the key will default to the basename of the file, and the
value will default to the file content. If the basename is an invalid key or you wish to chose your
own, you may specify an alternate key.

When creating a secret based on a directory, each file whose basename is a valid key in the
directory will be packaged into the secret. Any directory entries except regular files are ignored
(e.g. subdirectories, symlinks, devices, pipes, etc).

Examples:
  # Create a new secret named my-secret with keys for each file in folder bar
  kubectl create secret generic my-secret --from-file=path/to/bar

  # Create a new secret named my-secret with specified keys instead of names on disk
  kubectl create secret generic my-secret --from-file=ssh-privatekey=~/.ssh/id_rsa
--from-file=ssh-publickey=~/.ssh/id_rsa.pub

  # Create a new secret named my-secret with key1=supersecret and key2=topsecret
  kubectl create secret generic my-secret --from-literal=key1=supersecret
--from-literal=key2=topsecret

  # Create a new secret named my-secret using a combination of a file and a literal
  kubectl create secret generic my-secret --from-file=ssh-privatekey=~/.ssh/id_rsa
--from-literal=passphrase=topsecret

  # Create a new secret named my-secret from an env file
  kubectl create secret generic my-secret --from-env-file=path/to/bar.env
```

위 가이드 메세지를 따라, , `--from-literal` 옵션으로, `--type`의 값은 `k8s.ziwon.dev/secretsync`이고, 이름이 `supersecret`인 시크릿을 생성한다.

```sh
$ kubectl create --namespace=secretsync secret generic supersecret --from-literal=ziwon=awsome --type=k8s.ziwon.dev/secretsync
secret/supersecret created
```

만들어진 시크릿은 다음과 같이 확인할 수 있다. 

```sh
$ k get secrets
NAME                  TYPE                                  DATA   AGE
default-token-vhc89   kubernetes.io/service-account-token   3      12m
supersecret           k8s.ziwon.dev/secretsync              1      11m
```

시크릿 컨트롤러를 실행하면, 다음과 같이 `secretsync/supersecret`에 대해 핸들링하는 컨트롤러의 동작을 확인할 수 있다.

```sh
$ go run *.go
...
2019/03/03 18:09:22 onAdd: secretsync/supersecret
2019/03/03 18:09:22 Do something with this secret
2019/03/03 18:09:22 We should copy supersecret to namespace default
```

이상, 간단하게 Pod Watcher 컨트롤러와 Secret Controller를 만들어 보았다.

## 기타 
- [A Deep Dive Into Kubernetes Controllers](https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html)