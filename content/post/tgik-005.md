+++
date          = "2019-02-11T11:55:00+09:00"
draft         = false
title         = "TGI Kubernetes 005: Pod Params and Probes"
tags          = ["Kubernetes", "RBAC", "tgik"]
categories    = ["tgik"]
slug          = "tgik-005"
notoc         = true
socialsharing = true
nocomment     = false
+++

> 다섯 번 째 TGIK 에피소드는 특별한 내용이 없는 것 같네요. TGIK 진행된 분량이 많은 관계로 핵심만 추려서 빨리빨리 정리해야되지 않을까 싶네요.


[다섯 번 째 에피소드](https://github.com/heptio/tgik/tree/master/episodes/005)는 Pod을 실행하는 모든 매개변수에 대해 설명한다. 구체적으로 readinessProbe, livenessProve,  restart 등 매개변수에 대해 설명한다.

`kubectl run --help`는 스위스 군용 칼처럼 주어지는 매개변수에 따라 여러 가지 일을 한다. 사람들이 잘 모르는 한 가지 살펴볼만한 매개변수는 [`--generator`](https://kubernetes.io/docs/reference/kubectl/conventions/#generators)이다. `generator` 변수에 따라 여러가지 객체를 생성할 수 있다. 또한 다소 복잡해보이지만, `restart` 매개변수에 따라 생성되는 리소스가 달라진다.

| Generated<br/> Resource      | Cluster v1.4과 그 이후  |Cluster v1.3       |Cluster v1.2|Cluster v1.1 그 이전
|-------------|------------------------|------------------|-------------|----------------------|
|Pod          | `--restart=Never`      |`--restart=Never` | `--generator=run-pod/v1` | `--restart=OnFailure` 또는 `--restart=Never` |
|Replication<br/>Controller | `--generator=run/v1` | `--generator=run/v1` | `--generator=run/v1` | `--restart=Always` |
|Deployment | `--restart=Always` | `--restart=Always` | `--restart=Always` | N/A |
|Job | `--restart=OnFailure` | `--restart=OnFailure` | `--restart=OnFailure` OR `--restart=Never` | N/A |
|Cron Job | `--schedule=<cron>` | N/A | N/A | N/A |

## 쿠버네티스 오브젝트를 yaml파일로 추출하는 방법

kubectl 에 많은 옵션들이 있지만, 원하는 모든 것들을 kubectl로 할 수 있는 것은 아니다. Pop을 설정하는 다양한 값들을 커맨드 라인에 구겨넣는 게 쉽지 않은 일이며 kubectl은 제한된 뷰만을 보여줄 뿐이다.

실행 중인 Pod이 있고, 모킹을 해서 뭔가 다른 작업을 하고 싶을 경우 명령적인 모드에서 Pop을 실행하고 선억적인 모드로 전환하여 구체적인 다른 작업을 설정할 수 있다.

```sh
kubectl get po kuard -o yaml > kurad.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2019-02-11T06:43:45Z"
  labels:
    run: kuard
  name: kuard
  namespace: default
  resourceVersion: "4582"
  selfLink: /api/v1/namespaces/default/pods/kuard
  uid: 60b510d5-2dc8-11e9-82e8-0242cfd2ed41
spec:
  containers:
  - image: gcr.io/kuar-demo/kuard-amd64:blue
    imagePullPolicy: IfNotPresent
    name: kuard
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-g95gt
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: kind-1-worker2
  priority: 0
  restartPolicy: Never
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-g95gt
    secret:
      defaultMode: 420
      secretName: default-token-g95gt
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2019-02-11T06:43:45Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2019-02-11T06:43:52Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2019-02-11T06:43:52Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2019-02-11T06:43:45Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://74ff093a0684537893e20c67d789142c68fa2c8d1ead84909e9aec2c4d7d0985
    image: gcr.io/kuar-demo/kuard-amd64:blue
    imageID: docker-pullable://gcr.io/kuar-demo/kuard-amd64@sha256:c9996fc6b5c26d537e47dffd1b9a724682721883bfc9340d5cf94767ba76f103
    lastState: {}
    name: kuard
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: "2019-02-11T06:43:51Z"
  hostIP: 172.17.0.6
  phase: Running
  podIP: 10.42.0.1
  qosClass: BestEffort
  startTime: "2019-02-11T06:43:45Z"
```

위 파일을 편집기로 열면 보게 되는 것 중에 하나가 `resourceVersion`이다. 이는 리소스를 변경할 때 기준이 되는 특정 리소스의 버전이다. 이를 기본 Yaml 파일을 사용해서 Pod을 실행하고자 한다면 동작하지 않는다.

그래서 다음과 같이 `--export` 옵션을 사용하면 서버가 추가한 모든 것들을 제거해낼 수 있다. 

```sh
kubectl get po kuard -o yaml --export > kuard.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"creationTimestamp":"2019-02-11T06:43:45Z","labels":{"run":"kuard"},"name":"kuard","namespace":"default","resourceVersion":"4582","selfLink":"/api/v1/namespaces/default/pods/kuard","uid":"60b510d5-2dc8-11e9-82e8-0242cfd2ed41"},"spec":{"containers":[{"image":"gcr.io/kuar-demo/kuard-amd64:blue","imagePullPolicy":"IfNotPresent","name":"kuard","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File","volumeMounts":[{"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","name":"default-token-g95gt","readOnly":true}]}],"dnsPolicy":"ClusterFirst","enableServiceLinks":true,"nodeName":"kind-1-worker2","priority":0,"restartPolicy":"Never","schedulerName":"default-scheduler","securityContext":{},"serviceAccount":"default","serviceAccountName":"default","terminationGracePeriodSeconds":30,"tolerations":[{"effect":"NoExecute","key":"node.kubernetes.io/not-ready","operator":"Exists","tolerationSeconds":300},{"effect":"NoExecute","key":"node.kubernetes.io/unreachable","operator":"Exists","tolerationSeconds":300}],"volumes":[{"name":"default-token-g95gt","secret":{"defaultMode":420,"secretName":"default-token-g95gt"}}]},"status":{"conditions":[{"lastProbeTime":null,"lastTransitionTime":"2019-02-11T06:43:45Z","status":"True","type":"Initialized"},{"lastProbeTime":null,"lastTransitionTime":"2019-02-11T06:43:52Z","status":"True","type":"Ready"},{"lastProbeTime":null,"lastTransitionTime":"2019-02-11T06:43:52Z","status":"True","type":"ContainersReady"},{"lastProbeTime":null,"lastTransitionTime":"2019-02-11T06:43:45Z","status":"True","type":"PodScheduled"}],"containerStatuses":[{"containerID":"docker://74ff093a0684537893e20c67d789142c68fa2c8d1ead84909e9aec2c4d7d0985","image":"gcr.io/kuar-demo/kuard-amd64:blue","imageID":"docker-pullable://gcr.io/kuar-demo/kuard-amd64@sha256:c9996fc6b5c26d537e47dffd1b9a724682721883bfc9340d5cf94767ba76f103","lastState":{},"name":"kuard","ready":true,"restartCount":0,"state":{"running":{"startedAt":"2019-02-11T06:43:51Z"}}}],"hostIP":"172.17.0.6","phase":"Running","podIP":"10.42.0.1","qosClass":"BestEffort","startTime":"2019-02-11T06:43:45Z"}}
  creationTimestamp: null
  labels:
    run: kuard
  name: kuard
  selfLink: /api/v1/namespaces/default/pods/kuard
spec:
  containers:
  - image: gcr.io/kuar-demo/kuard-amd64:blue
    imagePullPolicy: IfNotPresent
    name: kuard
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-g95gt
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: kind-1-worker3
  priority: 0
  restartPolicy: Never
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-g95gt
    secret:
      defaultMode: 420
      secretName: default-token-g95gt
status:
  phase: Pending
  qosClass: BestEffort
```

그러나 Pod의 표준적인 정의(canonical definition) 등 제거해야할 요소가 여전히 있다. 선언적 방식으로 Pod를 배포하는데에 사용할 수는 있지만, 처음부터 선언적 방식으로 하고자 한다면 다음과 같이 `--dry-run` 옵션을 사용한다. 

```sh
kubectl run --generator=run-pod/v1 --image=gcr.io/kuar-demo/kuard-amd64:blue kuard -o yaml --dry-run > kuard.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: kuard
  name: kuard
spec:
  containers:
  - image: gcr.io/kuar-demo/kuard-amd64:blue
    name: kuard
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
}
```

여기서 살펴볼 것은 `restartPolicy`이다. 데모에서는 우분투 노드에 접속하여 docker를 죽여도 restartPolicy에 의해 Pod이 재시작되는 것을 보여준다. 다음으로 노드를 삭제(AWS에서는 Terminated) 시켰을 때, 즉 clean shutdown 시켰을 때, AWS 오토스케일링 그룹에 의해 새로운 노드가 부팅이 되지만, Pod은 사라진다. 그러나 `ReplicaSet`으로 선언하면 노드가 사라져도 다른 노드에서 새로 만든다. 보통 이를 Rescheduling이라고 알고 있는데, 실제로는 다른 노드에서 다른 이름으로 새로 생성하는 Recreating이라고 한다.   

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
    run: kuard
  name: kuard
spec:
  replicas: 2
  selector:
    matchLabels:
      run: kuard
  template:
    metadata:
      labels:
        run: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard
        resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Never
```

참고로, Joe Beda님은 동의를 못하겠다고 하지만 ReplicaSet의 경우 다음과 같이 restartPolicy가 Never로 설정이 되지 않는다. 

```sh
The ReplicaSet "kuard" is invalid: spec.template.spec.restartPolicy: Unsupported value: "Never": supported values: "Always"
```

## Probe

이제 Probe에 대해 설명한다. 가장 미니멀한 Deployment는 다음 명령으로 생성할 수 있다. 

```sh
$ kubectl run  --image=gcr.io/kuar-demo/kuard-amd64:blue kuard -o yaml --dry-run > kuard.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    run: kuard
  name: kuard
spec:
  replicas: 5
  selector:
    matchLabels:
      run: kuard
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard
        resources: {}
status: {}
```

위에서 `replicas`의 값을 5로 설정하였는데, Deployment는 ReplicaSet를 생성하고, ReplicaSet은 Pod을 생성한다. 알다시피, Probe에는 두 가지가 있는데, Readiness Probe는 트래픽을 받을 준비가 된 것이고, Liveness Probe는 서버가 정상적인지  기술힌다. 서버가 초기화 중이거나 또는 셧다운 중일 때는 Live이지만, Ready가 아닌 상태가 될 수 있다. Liveness Probe가 깨지면 그 Pod은 재시작된다. (예를 들어, 디비 커넥션 풀이 한도가 차서 서버에 더 이상 접속이 되지 않는 초보적인 오류의 경우, 해당 서버에 접속하여 수동으로 재시작하곤 하는데, Kubernetes에서는 자동으로 재시작되게 할 수 있다.) 그리고 Service에서 Readiness Probe에서 실패가 되고 있는 Pod들은 Endpoint 집합에서 축출하게 된다. 질문에서 잠깐 언급된 것처럼 Readiness Probe 상태가 실패인 것들을 오토스케일링 그룹 관점에서 축출하는 용도로는 사용하지 않는다고 한다. 

다음은 Probe 설정에 관한 몇 가지 필드들이다.

- `initialDelaySeconds`: liveness 또는 readiness 상태 프로브가 시작되기 전에 컨테이너가 시작된 후의 시간 (초)
- `periodSeconds`: 프로브 수행 빈도 (초). 기본값은 10초. 최소값은 1초
- `timeoutSeconds`: 프로브가 초과되는 시간 (초). 기본값은 1초. 최소값은 1초
- `successThreshod`: 실패 후 프로브가 성공한 것으로 간주되는 프로브의 최소 연속적인 성공 횟수. 기본값은 1. liveness는 1이어야 함. 최소값은 1.
- `failureThreshold`: Pod가 시작되고 프로브가 실패하면, 쿠버네티스는 failureThreshold 횟수 동안 실패를 시도한다. liveness 프로브의 경우, 포기는 포드를 다시 시작하는 것을 의미. 준비 상태 검사의 경우 포드는 Unready로 표시.  

[HTTP probes](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#httpgetaction-v1-core)에는 추가적인 필드가 있다.

- `host`: 연결할 호스트 이름. 기본적으로 포드 IP로 설정. 
- `scheme`: 호스트 연결에 사용할 스킴. 기본값은 HTTP
- `path`: HTTP 서버에 액세스하기 위한 경로
- `httpHeaders`: 요청에 설정할 사용자 지정 헤더. HTTP는 반복 헤더를 사용할 수 있음.
- `port`: 컨테이너에 억세스하는 포트의 번호 또는 이름. 1 ~ 65535 사이의 값