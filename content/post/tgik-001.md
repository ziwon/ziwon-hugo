+++
date          = "2019-01-13T17:12:00+00:00"
draft         = false
title         = "TGI Kubernetes:001 A Quick Tour"
tags          = ["Kubernetes"]
categories    = ["tgik"]
slug          = "tgik-001"
notoc         = true
socialsharing = true
nocomment     = false
+++

# TGIK: Thanks God It's Kubernetes

유튜브를 검색하다, 지난 2017년부터 Kubernetes라는 주제로 Heptio에서 매주 금요일마다 라이브 스트리밍을 하고 있다는 것을 알게 되었다. 아무래도 제한적인 실무 경험이나 개인적인 스터디만으로는 한계가 있기 때문에, 하나씩 팔로우업 해보고자 한다.
 
 - [TGIK 깃헙 리포지토리](https://github.com/heptio/tgik)
 - [TGIK 유트브 공식채널](https://www.youtube.com/playlist?list=PLvmPtYZtoXOENHJiAQc6HmV2jmuexKfrJ)
 - [TGIK 전체 목록](https://github.com/heptio/tgik/blob/master/playlist.md), [recollir's Playlist](https://github.com/recollir/tgik-playlist/blob/master/playlist.md) 
 

# A Quick Tour 

첫 번째 에피소드의 내용은 다음 링크를 참고한다.

 - https://github.com/heptio/tgik/tree/master/episodes/001
 
내용은 `kubeadm`을 기반으로 AWS에 Kubernetes 클러스터를 구동해보는 것이다. 이 내용은 APN(AWS Partner Network)의 기술 파트너인 Heptio가 AWS에 작성한 쿠버네티스 가이드 문서인 [Quick Start for Kubernetes by Heptio](https://aws.amazon.com/quickstart/architecture/heptio-kubernetes/)에 소개되어 있다. 
 
영상에서 진행하는 데모의 깃헙 저장소와 단계별 상세 디플로이 문서의 링크는 다음과 같다.
 
 - [GitHub - aws-quickstart](https://github.com/heptio/aws-quickstart)
 - [Step by Step Guide - heptio-kubernetes-on-the-aws-cloud](https://aws-quickstart.s3.amazonaws.com/quickstart-heptio/doc/heptio-kubernetes-on-the-aws-cloud.pdf)
 

전체적인 아키텍처는 다음과 같다. AWS의 베스트 프랙티스를 따라, Public Network와 Private Network로 디자인하고 있다. Kubernets API를 외부에서 접속할 수 있도록 마스터 노드에 ELB를 연결한 것이 특이하다.
 
 <center>
  ![](https://d1.awsstatic.com/partner-network/QuickStart/datasheets/heptio-kubernetes-architecture.4ce06fe6b45b290e2b46ddf5dc7a664d45d7aae2.png)
</center>

데모에서는 CloudFormation을 사용해 다음의 컴포넌트들을 AWS에 구성하게 된다. (이상, [kubernetes-cluster-with-new-vpc.template 참조](https://github.com/heptio/aws-quickstart/blob/master/templates/kubernetes-cluster-with-new-vpc.template))

- 단일 가용영 영역에 가상 프라이빗 클라우드 (VPC)
- 서브넷 2개, 퍼블릭 1개 및 프라이빗 1개
- 퍼브릭서브넷에서 바스티온 호스트로 작동하는 EC2 인스턴스 하나
- 프라이빗 서브넷의 마스터 노드에 대한 자동 복구가 가능한 EC2 인스턴스 하나
- 프라이빗 서브넷의 추가 노드에 대한 자동 스케일링 그룹의 EC2 인스턴스 1-20개
- Kubernetes API에 대한 HTTPS 액세스를 위한 Elastic Load Balancing(ELB) 로드 밸런싱 장치 1개
- 모든 노드에 Ubuntu 16.04 LTS
- Linux에서 Kubernetes를 부트스트래핑하기 위한 Kubeadm
- Kubernetes가 의존하는 컨테이너 런타임에 대한 도커
- Pod 네트워킹을 위한 Calico 또는 Weave. 디폴트는 Calico이다.
- 클러스터 DNS에 대한 CoreDNS 또는 KubeDNS 기본값은 CoreDNS이며, CoreDNS는 CoreDNS로 대체되며, CoreDNS를 지원할 수 없는 환경에만 제공.
- SSH 액세스를 위해 포트 22를 허용하는 스택 전용 security group, HTTPS 액세스를 위한 포트 6443 및 모든 포트의 노드 간 연결을 허용하는 단일 security group.

깃헙의 다음 스크립트를 이용하면 CloudFormation 기반으로 바로 디플로이 할 수 있다.

- https://github.com/aws-quickstart/quickstart-heptio/blob/master/bin/boot-master-branch.sh

```shell

#!/usr/bin/env bash

# This is where Heptio stores templates/scripts for the master branch of this repository
S3_BUCKET=heptio-aws-quickstart-test
S3_PREFIX=heptio/kubernetes/master/

# Where to place your cluster
REGION=us-west-2
AZ=us-west-2b

# What to name your CloudFormation stack
STACK=Heptio-Kubernetes

# What SSH key you want to allow access to the cluster (must be created ahead of time in your AWS EC2 account)
KEYNAME=mykey

# What IP addresses should be able to connect over SSH and over the Kubernetes API
INGRESS=0.0.0.0/0

aws cloudformation create-stack \
  --region $REGION \
  --stack-name $STACK \
  --template-url "https://${S3_BUCKET}.s3.amazonaws.com/${S3_PREFIX}templates/kubernetes-cluster-with-new-vpc.template" \
  --parameters \
    ParameterKey=AvailabilityZone,ParameterValue=$AZ \
    ParameterKey=KeyName,ParameterValue=$KEYNAME \
    ParameterKey=AdminIngressLocation,ParameterValue=$INGRESS \
    ParameterKey=QSS3BucketName,ParameterValue=${S3_BUCKET} \
    ParameterKey=QSS3KeyPrefix,ParameterValue=${S3_PREFIX} \
  --capabilities=CAPABILITY_IAM
  ```

> 실제로 따라 해보지는 않았다. 테스트 클러스터라고 해도 Kops나 EKS를 사용할 것 같다. 영상에서도 CloudFormation과 Kops 중 어떤 게 더 좋냐고 물어보고 있는데, 이 예제는 익숙한 툴을 이용해 Kubernetes를 재빨리 디플로이 하기 위해 만든 것이며, (보안적인 어떤 면에서는 이 예제의 방식보다 덜 안전할 수도 있지만) Kops가 프로덕션에서 적절하게 튜닝할 수 있는 부가 기능이 많다고 한다. 또한, HA 클러스터 등 좀 더 안정적이기 때문에 프로덕션 레벨에서는 Kops를 사용할 수 있다고 한다. - [11:10](https://youtu.be/9YYeE-bMWv8?t=670)

Kubernetes 클러스터가 구성된 후에는 Pod 등을 설명하고 스케쥴링하는 Kubernetes 관리자라면 비교적 익숙한 기본 명령어들을 다루고 있다.

- [command line snippets](https://gist.github.com/jbeda/9a2097c726584560fa13f6c1ae13abfb)

```shell
# This tells kubecfg to read its config from the local directory
export KUBECONFIG=./kubeconfig

# Looking at the cluster
kubectl get nodes
kubectl get pods --namespace=kube-system

# Running a single pod
kubectl run --generator=run-pod/v1 --image=gcr.io/kuar-demo/kuard-amd64:1 kuard
kubectl get pods
kubectl run --generator=run-pod/v1 --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --dry-run -o yaml
kubectl get pods kuard -o yaml
kubectl port-forward kuard 8080:8080
kubectl delete pod kuard

# Running a deployment
kubectl run --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --replicas=5 --dry-run -o yaml
kubectl run --image=gcr.io/kuar-demo/kuard-amd64:1 kuard --replicas=5
kubectl get pods

# Running a service
kubectl expose deployment kuard --type=LoadBalancer --port=80 --target-port=8080 --dry-run -o yaml
kubectl get service kuard -o wide

# Wait a while for the ELB to be ready
KUARD_LB=$(kubectl get service kuard -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')

---
# Doing a deployment
## Window 1
## On the mac, install with `brew install watch`
watch -n 0.1 kubectl get pods

## Window 2
## On the mac, install with `brew install jq`
while true ; do curl -s http://${KUARD_LB}/env/api | jq '.env.HOSTNAME'; done

## Window 3
kubectl scale deployment kuard --replicas=10
kubectl set image deployment kuard kuard=gcr.io/kuar-demo/kuard-amd64:2
kubectl rollout undo deployment kuard

---
# Clean up
kubectl delete deployment,service kuard
```

개인적으로 Pod 스케쥴링과 스케일링에 간단한 node앱들을 사용했었는데 데모에서처럼 "Request 상세, Server 환경변수, Liveness Probe, Readiness Probe, DNS 쿼리, KeyGen Workload, MemQ 서버, 파일 시스템 브라우저" 등 여러가지를 살펴볼 수 있는 kuard를 이용하는 것도 괜찮아 보인다.

- [kuard](https://github.com/kubernetes-up-and-running/kuard)

## sig-scalability

### etcd
2000대 워커 노드 제한에 대한 병목은 무엇인가라는 질문이 있었는데, 5000대까지 테스트해보았으며, etcd의 확장성 가장 중요하다고 한다. 즉, 노드 간의 일관된 코디네이션을 할 수 있는 양이 관건이라고 한다. 클라이언트 수, 초당 요청수, 저장 데이터 크기에 따른 클러스터 크기별 etcd의 하드웨어 구성은 다음 문서를 참고한다. 문서에 따르면, 일반적인 클러스터의 경우 2~4개의 코어, 8GB 메모리면 충분하며, 디스크는 50개의 순차적인 IOPS가 필요하다. (디스크는 etcd 클러스터 배치 성능 및 안정성에 있어서 가장 중요한 요인이다. etcd의 합의 프로토콜은 메터데이터를 로그에 지속적으로 저장하는 것에 달려 있으므로, etcd 클러스터 멤버는 모든 요청을 디스크에 기록해야 한다. 따라서, 쓰기 지연이 발생하지 않고, 하트비트가 타임아웃이 되지 않고 일렉션이 발생하지 않도록 해야 한다. 장애 발생시, Disk 대역폭이 많을수록 복구 시간이 단축된다.)

- [Hardware recommendations](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md)

### 메트릭 목표
Kubernetes의 확장성에 관한 자세한 내용은 [Kubernetes Scaling and Performance Goals](https://github.com/kubernetes/community/blob/master/sig-scalability/goals.md)를 참고하라고 한다. 간단히 살펴보면, 주 메트릭과 파생 메트릭으로 나뉜다. 

주 메트릭의 경우, 클러스터별 최대 코어 수는 20만이, 코어별 최대 Pod 수는 10개가 목표이다. 코어 비율은 1:1에서 4:1(GB/코어)라고 가정한다. 경험상 평균적으로 많은 Pod/프로세스가 코어별 1~4GB RAM이 필요한 것으로 나타났다고 한다. 노드별 관리 오버헤드는 최소 0.5코어, 1GB RAM으로 5% 미만이 목표이며,  Docker, KubeProxy, Kubelet, 메트릭 수집, 커널 제외한 수치이다. (예: 32개 코어, 64GB 머신, 1.6개 코어, 노드 관리를 위한 3.2GB RAM), 그리고 클러스터별 오버헤드는 1% 미만, 코어 2개 이상, 4GB RAM가 목표이며, apiserver, 스케줄러, 컨트롤러 등을 포함한 모든 비노드별 구성 요소 포함하고 DNS, 힙스터 등, HA 제외한 수치이다. 

파생 메트릭의 경우, 노드별 최대 코어수는 현재 64개, 향후 수백개가 목표이고, 머신별 최대 Pod 수는 500개라고 한다. 클러스터별 최대 머신수는 5000개이고, 클러스터별 최대 Pod의 수는 50만개이다. Pod의 엔드투엔드 스타트업 시간은 5초 이하가 목표이고, 스케쥴링의 스루풋은 초당 100개의 Pod이 목표라고 한다. 최대 클러스터 포화 시간은 90 분이 목표이다.

### 클라우드별 테스팅 환경
또한, 클라우드 프로바이더별로 Kubernetes 확장성의 테스팅 환경은 다음 문서를 참고한다.

- [Scalability Testing/Analysis Environment and Goals](https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/provider-configs.md)

## Kubernetes 1.7

그리고 [Kubernetes 1.7 릴리즈 노트](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.7.md#major-themes)를 간략히 설명하고 있다. 1.7의 내용은 확장성과 보안이 두가지 주요 주제였다고 한다. 시스템의 커널, 즉 시스템의 핵심을 변경하지 않고, 각 컴포넌트를 플러그인하고 서로 빌드해가는 확장성을 계속해서 개선해 나갔으며, 또한 처음부터 여러 컴포넌트, 여러 어플리케이션, 여러 팀들이 개발을 해왔기 때문에 보안 이슈가 다른 컴포넌트 등으로 전파되지 않도록 해왔다고 한다. 이상, 다소 오래된 내용이라 넘어가기로 한다. 

> Kubernetes 1.7은 Kubernetes의 광범위한 생산 사용에 의해 동기화된 보안, 뛰어난 애플리케이션 및 확장성 기능을 추가하는 릴리스다. 이 릴리스의 보안 강화에는 암호화된 시크릿(alpha), Pod간 통신을 위한 네트워크 정책, API 리소스에 대한 Kubelet 액세스를 제한하는 노드 작성자, Kubelet 클라이언트/서버 TLS 인증서 로테이션(alpha)이 포함된다. 상태 저장 애플리케이션의 주요 기능에는 StatefulSet 자동 업데이트, DaemonSet의 향상된 업데이트, 보다 빠른 StatefulSet 확장을 위한 버스트 모드,  (alpha) 로컬 스토리지에 대한 지원이 포함된다. 확장성 기능에는 API 집계(beta), ThirdPartyResources를 위한 CustomResourceDefinitions(beta), 확장 가능한 Admission controller(alpha), 플러그형 cloud provider(alpha) 및 CRI(컨테이너 런타임 인터페이스) 개선이 포함된다.

1.7 릴리즈에 대한 좀 자세한 내용은 [쿠버네티스 1.7의 새로운 기능 : 로컬 스토리지, 암호화 등](https://www.itworld.co.kr/news/105494) 참고한다.

## ksonnet

`ksonnet`은 시간 관계상 다루지 못한 것 같다. 링크된 홈페이지에 소개된 것처럼, 다양한 클러스터 및 환경에서 설정을 관리하는 cli 툴인 것으로 보인다.

> Built on the JSON templating language Jsonnet, ksonnet provides an organizational structure and specialized features for managing configurations across different clusters and environments.

- https://github.com/ksonnet/ksonnet
- [쿠버네티스 리소스 배포와 관리를 위한 ksonnet](http://bcho.tistory.com/1302?fbclid=IwAR2KJYAo7TBUTFynZ1NFY0xF7T2R6RJBS77wRlj8u5RoBUyGtbWlnMFSBbs)

이상으로, 간략히 TGIK 에피소드 1을 살펴보았다.
