+++
date          = "2019-02-05T11:55:00+09:00"
draft         = false
title         = "TGI Kubernetes 004: RBAC"
tags          = ["Kubernetes", "RBAC",]
categories    = ["tgik"]
slug          = "tgik-004"
notoc         = true
socialsharing = true
nocomment     = false
+++


[네번째 에피소드](https://github.com/heptio/tgik/tree/master/episodes/004)는 쿠버네티스의 RBAC을 중심으로 인증(Authorization)과 권한(Authentication)에 대해 설명하고 있다. 쿠버네티스의 RBAC에 대해 충분히 이해하고 있다면, 이번 TGIK는 스킵하고 [Adding users on "Quick Start for Kubernetes on AWS"](https://www.linkedin.com/pulse/adding-users-quick-start-kubernetes-aws-jakub-scholz)만 봐도 좋을 듯하다. 개인적으로 쿠버네티스 RBAC에 대해서는 CNCF의 [Role based access control (RBAC) policies in Kubernetes](https://www.youtube.com/watch?v=CnHTCTP8d48) 웨비나를 추천하고 싶다. 

<center>•••</center>

## KubeConfig 

`kubeconfig` 파일의 기본 위치는 `$HOME/.kube/config`이며, `KUBECONFIG` 환경변수 또는 `--kubeconfig` 옵션으로 지정할 수 있다. 

앞선 TGIK 데모에서는 Heptio의 QuickStart를 사용해 쿠버네티스 클러스터를 생성할 때, 클러스터 작업을 실행하기 위해 다음과 같은 방법으로 마스터 노드에서 복사한 `kubeconfig` 파일을 로컬에 복사하였다.

QuickStart에서 CloudFormation으로 프로비저닝할 때, 로컬 복사를 위한 커맨드가 다음과 같이 자동으로 생성되는데, 보다시피 `nc` 명령을 이용해 netcat 링크를 만들어 내부 마스터 서버에 접속하고 있다. (`netcat`을 이용하지 않고, `-W` 옵션으로도 클라이언트의 입출력을 마스터 호스트로 전달할 수 있다.)

```shell
SSH_KEY="path/to/laptop.pem"; scp -i $SSH_KEY -o ProxyCommand="ssh -i \"${SSH_KEY}\" ubuntu@13.209.9.190 nc %h %p" ubuntu@10.0.21.21:~/kubeconfig ./kubeconfig
```

다음은 복사된 `kubeconfig` 파일의 내용이다. 

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <base64-encoded-ca-cert>
    server: https://13.125.59.233:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: admin
  name: admin@kubernetes
current-context: admin@kubernetes
kind: Config
preferences: {}
users:
- name: admin
  user:
  	client-certificate-data: <base64-encoded-csr>
  	client-key-data: <base64-encoded-private-key>
```

보다시피, yaml 파일 형식이며, `certificate-authority-data`, `client-certificate-data`, `client-key-data`처럼 인증서 및 개인키는 base64로 인코딩되어 있다. 

### certificate-authority-data

첫 번째 base64로 인코딩된 데이터는 API 서버의 TLS 인증서를 발급한 CA(certificate authority, 인증 기관)이다. 대부분의 TLS 연결은 기본적으로 `/etc/ssl`에서 신뢰하는 CA를 찾는데, 쿠버네티스의 경우 `kubeconfig` 파일의 `certificate-authority-data` 에 API 서버의 TLS 인증서를 발급한 Root CA을 기술하고 있다.

Root CA외에 API 서버에서 인자로 전달되는 인증서, 공개키, 개인키 데이터는 다음과 같다. (쿠버네티스 1.13.x 기준)

```shell
$ kube-apiserver 
    --authorization-mode=Node,RBAC 
    --advertise-address=172.17.0.3 
    --allow-privileged=true 
    --client-ca-file=/etc/kubernetes/pki/ca.crt 
    --enable-admission-plugins=NodeRestriction 
    --enable-bootstrap-token-auth=true 
    --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt 
    --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt 
    --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key 
    --etcd-servers=https://127.0.0.1:2379 
    --insecure-port=0 
    --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt 
    --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key 
    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname 
    --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt 
    --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key 
    --requestheader-allowed-names=front-proxy-client 
    --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt 
    --requestheader-extra-headers-prefix=X-Remote-Extra- 
    --requestheader-group-headers=X-Remote-Group 
    --requestheader-username-headers=X-Remote-User 
    --secure-port=6443 
    --service-account-key-file=/etc/kubernetes/pki/sa.pub 
    --service-cluster-ip-range=10.96.0.0/12 
    --tls-cert-file=/etc/kubernetes/pki/apiserver.crt 
    --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

보다시피 API 서버의 경우, TLS 연결을 위한 인증서와 개인키는 각각 `--tls-cert-file`, `--tls-private-key-file` 인자를 통해 전달되고 있다. 그리고 API 서버의 `--client-ca-file` 인자는 `/etc/kubernetes/pki/ca.crt` 파일로 전달되고 있는데, 이는 쿠버네티스의 Root CA이며, 이는 앞서 살펴본 `kubeconfig`의 `certificate-authority-data`와 동일하다.

모든 쿠버네티스 클러스터에는 쿠버네티스 컨트롤 플레인 또는 사용자에게 인증서를 제공하는데 사용할 수 있는 클러스터 내장 인증 기관(certificate authority)이 있다. CA는 API 컴포넌트에서 API 서버의 인증서를 확인하고, API 서버가 kubelet 클라이언트 인증서를 확인하는데 사용한다. 이를 지원하기 위해 CA 인증서 번들은 클러스터의 모든 노드에 분산되며 `default` 서비스 계정에 연결된 시크릿으로서 배포된다. 쿠버네티스에는 기본적으로 세 가지 Root CA가 필요하다. 이들은 `/etc/kubernetes/pki`에 저장된다. 

<center>

| path       			| CN 					  | description                  |
|----------------------|------------------------|------------------------------|
|ca.crt,key  			|kubernetes-ca			  |Kubernetes general CA         |
|etcd/ca.crt,key       |etcd-ca				  |For all etcd-related function |
|front-proxy-ca.crt,key|kubernetes-front-proxy-ca|For the front-end proxy       |

(참고 - [PKI Certificates and Requirements](https://kubernetes.io/docs/setup/certificates/))
</center>


이외에도 API 서버는 다음과 같이 인증과 관련된 다른 커맨드라인 인자를 지정할 수 있다. (참고 - [kube-apiserver
](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/))

```txt
--cert-dir string                           The directory where the TLS certs are located. If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. (default "/var/run/kubernetes")
--client-ca-file string                     If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
--etcd-certfile string                      SSL certification file used to secure etcd communication.
--etcd-keyfile string                       SSL key file used to secure etcd communication.
--etcd-cafile string                        SSL Certificate Authority file used to secure etcd communication.
--kubelet-certificate-authority string      Path to a cert file for the certificate authority.
--kubelet-client-certificate string         Path to a client cert file for TLS.
--kubelet-client-key string                 Path to a client key file for TLS.
--proxy-client-cert-file string             Client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins. It is expected that this cert includes a signature from the CA in the --requestheader-client-ca-file flag. That CA is published in the 'extension-apiserver-authentication' configmap in the kube-system namespace. Components recieving calls from kube-aggregator should use that CA to perform their half of the mutual TLS verification.
--proxy-client-key-file string              Private key for the client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins.
--requestheader-allowed-names stringSlice   List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.
--requestheader-client-ca-file string       Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by --requestheader-username-headers
--service-account-key-file stringArray      File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. If unspecified, --tls-private-key-file is used. The specified file can contain multiple keys, and the flag can be specified multiple times with different files.
--ssh-keyfile string                        If non-empty, use secure SSH proxy to the nodes, using this user keyfile
--tls-ca-file string                        If set, this certificate authority will used for secure access from Admission Controllers. This must be a valid PEM-encoded CA bundle. Alternatively, the certificate authority can be appended to the certificate provided by --tls-cert-file.
--tls-cert-file string                      File containing the default x509 Certificate for HTTPS. (CA cert, if any, concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to /var/run/kubernetes.
--tls-private-key-file string               File containing the default x509 private key matching --tls-cert-file.
--tls-sni-cert-key namedCertKey             A pair of x509 certificate and private key file paths, optionally suffixed with a list of domain patterns which are fully qualified domain names, possibly with prefixed wildcard segments. If no domain patterns are provided, the names of the certificate are extracted. Non-wildcard matches trump over wildcard matches, explicit domain patterns trump over extracted names. For multiple key/certificate pairs, use the --tls-sni-cert-key multiple times. Examples: "example.crt,example.key" or "foo.crt,foo.key:*.foo.com,foo.com". (default [])
```

다음과 같이, `kubectl`에서 클러스터 노드 정보를 조회할 때 디버그 레벨 `--v=10`을 지정해보자.

<details>
<summary>kubectl get nodes --v=10</summary>
```shell
$ kubectl get nodes --v=10
I0204 13:05:46.390541   15326 loader.go:359] Config loaded from file /Users/luno/Workspace/GitHub/ziwon/quickstart-heptio/kubeconfig
I0204 13:05:46.391193   15326 loader.go:359] Config loaded from file /Users/luno/Workspace/GitHub/ziwon/quickstart-heptio/kubeconfig
I0204 13:05:46.392557   15326 cached_discovery.go:104] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/servergroups.json
I0204 13:05:46.393214   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/batch/v1/serverresources.json
I0204 13:05:46.393495   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/autoscaling/v2beta1/serverresources.json
I0204 13:05:46.393529   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/certificates.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393559   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/batch/v1beta1/serverresources.json
I0204 13:05:46.393583   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiregistration.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393599   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/policy/v1beta1/serverresources.json
I0204 13:05:46.393613   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/storage.k8s.io/v1/serverresources.json
I0204 13:05:46.393622   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/admissionregistration.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393638   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/storage.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393657   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1beta2/serverresources.json
I0204 13:05:46.393653   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/scheduling.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393707   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiextensions.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393740   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/networking.k8s.io/v1/serverresources.json
I0204 13:05:46.393750   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/v1/serverresources.json
I0204 13:05:46.393771   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/autoscaling/v1/serverresources.json
I0204 13:05:46.393787   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/rbac.authorization.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393802   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/rbac.authorization.k8s.io/v1/serverresources.json
I0204 13:05:46.393816   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiregistration.k8s.io/v1/serverresources.json
I0204 13:05:46.393845   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authentication.k8s.io/v1/serverresources.json
I0204 13:05:46.393899   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/events.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393912   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authentication.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393922   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authorization.k8s.io/v1/serverresources.json
I0204 13:05:46.393941   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1beta1/serverresources.json
I0204 13:05:46.393946   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/extensions/v1beta1/serverresources.json
I0204 13:05:46.393966   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authorization.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.393984   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1/serverresources.json
I0204 13:05:46.394926   15326 loader.go:359] Config loaded from file /Users/luno/Workspace/GitHub/ziwon/quickstart-heptio/kubeconfig
I0204 13:05:46.395217   15326 cached_discovery.go:104] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/servergroups.json
I0204 13:05:46.395311   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/scheduling.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395382   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/autoscaling/v1/serverresources.json
I0204 13:05:46.395421   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/rbac.authorization.k8s.io/v1/serverresources.json
I0204 13:05:46.395452   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/policy/v1beta1/serverresources.json
I0204 13:05:46.395477   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/batch/v1beta1/serverresources.json
I0204 13:05:46.395521   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/batch/v1/serverresources.json
I0204 13:05:46.395529   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/autoscaling/v2beta1/serverresources.json
I0204 13:05:46.395547   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/networking.k8s.io/v1/serverresources.json
I0204 13:05:46.395550   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/admissionregistration.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395559   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authentication.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395550   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/storage.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395551   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/certificates.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395592   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authorization.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395599   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authorization.k8s.io/v1/serverresources.json
I0204 13:05:46.395620   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/extensions/v1beta1/serverresources.json
I0204 13:05:46.395625   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiextensions.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395639   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/events.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395651   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/storage.k8s.io/v1/serverresources.json
I0204 13:05:46.395677   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiregistration.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395651   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authentication.k8s.io/v1/serverresources.json
I0204 13:05:46.395652   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/v1/serverresources.json
I0204 13:05:46.395660   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiregistration.k8s.io/v1/serverresources.json
I0204 13:05:46.395660   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/rbac.authorization.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.395743   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1/serverresources.json
I0204 13:05:46.395722   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1beta1/serverresources.json
I0204 13:05:46.395753   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1beta2/serverresources.json
I0204 13:05:46.395900   15326 cached_discovery.go:104] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/servergroups.json
I0204 13:05:46.396066   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/v1/serverresources.json
I0204 13:05:46.396129   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiregistration.k8s.io/v1/serverresources.json
I0204 13:05:46.396186   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiregistration.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.396276   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/extensions/v1beta1/serverresources.json
I0204 13:05:46.396383   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1/serverresources.json
I0204 13:05:46.396474   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1beta2/serverresources.json
I0204 13:05:46.396549   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apps/v1beta1/serverresources.json
I0204 13:05:46.396602   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/events.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.396649   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authentication.k8s.io/v1/serverresources.json
I0204 13:05:46.396705   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authentication.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.396761   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authorization.k8s.io/v1/serverresources.json
I0204 13:05:46.396821   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/authorization.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.396873   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/autoscaling/v1/serverresources.json
I0204 13:05:46.396934   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/autoscaling/v2beta1/serverresources.json
I0204 13:05:46.396988   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/batch/v1/serverresources.json
I0204 13:05:46.397045   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/batch/v1beta1/serverresources.json
I0204 13:05:46.397103   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/certificates.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.397157   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/networking.k8s.io/v1/serverresources.json
I0204 13:05:46.397221   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/policy/v1beta1/serverresources.json
I0204 13:05:46.397285   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/rbac.authorization.k8s.io/v1/serverresources.json
I0204 13:05:46.397350   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/rbac.authorization.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.397403   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/storage.k8s.io/v1/serverresources.json
I0204 13:05:46.397458   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/storage.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.397515   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/admissionregistration.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.397572   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/apiextensions.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.397625   15326 cached_discovery.go:70] returning cached discovery info from /Users/luno/.kube/cache/discovery/13.125.59.233_6443/scheduling.k8s.io/v1beta1/serverresources.json
I0204 13:05:46.398497   15326 loader.go:359] Config loaded from file /Users/luno/Workspace/GitHub/ziwon/quickstart-heptio/kubeconfig
I0204 13:05:46.398751   15326 round_trippers.go:386] curl -k -v -XGET  -H "Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io, application/json" -H "User-Agent: kubectl/v1.12.3 (darwin/amd64) kubernetes/435f92c" 'https://13.125.59.233:6443/api/v1/nodes?limit=500'
I0204 13:05:46.435293   15326 round_trippers.go:405] GET https://13.125.59.233:6443/api/v1/nodes?limit=500 200 OK in 36 milliseconds
I0204 13:05:46.435313   15326 round_trippers.go:411] Response Headers:
I0204 13:05:46.435318   15326 round_trippers.go:414]     Content-Type: application/json
I0204 13:05:46.435324   15326 round_trippers.go:414]     Date: Mon, 04 Feb 2019 04:05:46 GMT
I0204 13:05:46.437396   15326 request.go:942] Response Body: {"kind":"Table","apiVersion":"meta.k8s.io/v1beta1","metadata":{"selfLink":"/api/v1/nodes","resourceVersion":"7633"},"columnDefinitions":[{"name":"Name","type":"string","format":"name","description":"Name must be unique within a namespace. Is required when creating resources, although some resources may allow a client to request the generation of an appropriate name automatically. Name is primarily intended for creation idempotence and configuration definition. Cannot be updated. More info: http://kubernetes.io/docs/user-guide/identifiers#names","priority":0},{"name":"Status","type":"string","format":"","description":"The status of the node","priority":0},{"name":"Roles","type":"string","format":"","description":"The roles of the node","priority":0},{"name":"Age","type":"string","format":"","description":"CreationTimestamp is a timestamp representing the server time when this object was created. It is not guaranteed to be set in happens-before order across separate operations. Clients may not set this value. It is represented in RFC3339 form and is in UTC.\n\nPopulated by the system. Read-only. Null for lists. More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata","priority":0},{"name":"Version","type":"string","format":"","description":"Kubelet Version reported by the node.","priority":0},{"name":"Internal-IP","type":"string","format":"","description":"List of addresses reachable to the node. Queried from cloud provider, if available. More info: https://kubernetes.io/docs/concepts/nodes/node/#addresses","priority":1},{"name":"External-IP","type":"string","format":"","description":"List of addresses reachable to the node. Queried from cloud provider, if available. More info: https://kubernetes.io/docs/concepts/nodes/node/#addresses","priority":1},{"name":"OS-Image","type":"string","format":"","description":"OS Image reported by the node from /etc/os-release (e.g. Debian GNU/Linux 7 (wheezy)).","priority":1},{"name":"Kernel-Version","type":"string","format":"","description":"Kernel Version reported by the node from 'uname -r' (e.g. 3.16.0-0.bpo.4-amd64).","priority":1},{"name":"Container-Runtime","type":"string","format":"","description":"ContainerRuntime Version reported by the node through runtime remote API (e.g. docker://1.5.0).","priority":1}],"rows":[{"cells":["ip-10-0-11-90.ap-northeast-2.compute.internal","Ready","\u003cnone\u003e","1h","v1.11.5","10.0.11.90","\u003cnone\u003e","Ubuntu 16.04.5 LTS","4.4.0-1072-aws","docker://17.3.2"],"object":{"kind":"PartialObjectMetadata","apiVersion":"meta.k8s.io/v1beta1","metadata":{"name":"ip-10-0-11-90.ap-northeast-2.compute.internal","selfLink":"/api/v1/nodes/ip-10-0-11-90.ap-northeast-2.compute.internal","uid":"208bde9f-2829-11e9-a2c8-02015048da6c","resourceVersion":"7622","creationTimestamp":"2019-02-04T03:01:11Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"t2.small","beta.kubernetes.io/os":"linux","failure-domain.beta.kubernetes.io/region":"ap-northeast-2","failure-domain.beta.kubernetes.io/zone":"ap-northeast-2a","kubernetes.io/hostname":"ip-10-0-11-90.ap-northeast-2.compute.internal"},"annotations":{"kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock","node.alpha.kubernetes.io/ttl":"0","volumes.kubernetes.io/controller-managed-attach-detach":"true"}}}},{"cells":["ip-10-0-15-49.ap-northeast-2.compute.internal","Ready","\u003cnone\u003e","1h","v1.11.5","10.0.15.49","\u003cnone\u003e","Ubuntu 16.04.5 LTS","4.4.0-1072-aws","docker://17.3.2"],"object":{"kind":"PartialObjectMetadata","apiVersion":"meta.k8s.io/v1beta1","metadata":{"name":"ip-10-0-15-49.ap-northeast-2.compute.internal","selfLink":"/api/v1/nodes/ip-10-0-15-49.ap-northeast-2.compute.internal","uid":"208d8cb1-2829-11e9-a2c8-02015048da6c","resourceVersion":"7619","creationTimestamp":"2019-02-04T03:01:11Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"t2.small","beta.kubernetes.io/os":"linux","failure-domain.beta.kubernetes.io/region":"ap-northeast-2","failure-domain.beta.kubernetes.io/zone":"ap-northeast-2a","kubernetes.io/hostname":"ip-10-0-15-49.ap-northeast-2.compute.internal"},"annotations":{"kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock","node.alpha.kubernetes.io/ttl":"0","volumes.kubernetes.io/controller-managed-attach-detach":"true"}}}},{"cells":["ip-10-0-21-21.ap-northeast-2.compute.internal","Ready","master","1h","v1.11.5","10.0.21.21","\u003cnone\u003e","Ubuntu 16.04.5 LTS","4.4.0-1072-aws","docker://17.3.2"],"object":{"kind":"PartialObjectMetadata","apiVersion":"meta.k8s.io/v1beta1","metadata":{"name":"ip-10-0-21-21.ap-northeast-2.compute.internal","selfLink":"/api/v1/nodes/ip-10-0-21-21.ap-northeast-2.compute.internal","uid":"af849c71-2828-11e9-a2c8-02015048da6c","resourceVersion":"7626","creationTimestamp":"2019-02-04T02:58:02Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"t2.small","beta.kubernetes.io/os":"linux","failure-domain.beta.kubernetes.io/region":"ap-northeast-2","failure-domain.beta.kubernetes.io/zone":"ap-northeast-2a","kubernetes.io/hostname":"ip-10-0-21-21.ap-northeast-2.compute.internal","node-role.kubernetes.io/master":""},"annotations":{"kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock","node.alpha.kubernetes.io/ttl":"0","volumes.kubernetes.io/controller-managed-attach-detach":"true"}}}},{"cells":["ip-10-0-23-205.ap-northeast-2.compute.internal","Ready","\u003cnone\u003e","1h","v1.11.5","10.0.23.205","\u003cnone\u003e","Ubuntu 16.04.5 LTS","4.4.0-1072-aws","docker://17.3.2"],"object":{"kind":"PartialObjectMetadata","apiVersion":"meta.k8s.io/v1beta1","metadata":{"name":"ip-10-0-23-205.ap-northeast-2.compute.internal","selfLink":"/api/v1/nodes/ip-10-0-23-205.ap-northeast-2.compute.internal","uid":"22442c26-2829-11e9-a2c8-02015048da6c","resourceVersion":"7628","creationTimestamp":"2019-02-04T03:01:14Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"t2.small","beta.kubernetes.io/os":"linux","failure-domain.beta.kubernetes.io/region":"ap-northeast-2","failure-domain.beta.kubernetes.io/zone":"ap-northeast-2a","kubernetes.io/hostname":"ip-10-0-23-205.ap-northeast-2.compute.internal"},"annotations":{"kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock","node.alpha.kubernetes.io/ttl":"0","volumes.kubernetes.io/controller-managed-attach-detach":"true"}}}},{"cells":["ip-10-0-30-16.ap-northeast-2.compute.internal","Ready","\u003cnone\u003e","1h","v1.11.5","10.0.30.16","\u003cnone\u003e","Ubuntu 16.04.5 LTS","4.4.0-1072-aws","docker://17.3.2"],"object":{"kind":"PartialObjectMetadata","apiVersion":"meta.k8s.io/v1beta1","metadata":{"name":"ip-10-0-30-16.ap-northeast-2.compute.internal","selfLink":"/api/v1/nodes/ip-10-0-30-16.ap-northeast-2.compute.internal","uid":"1c7ce276-2829-11e9-a2c8-02015048da6c","resourceVersion":"7627","creationTimestamp":"2019-02-04T03:01:04Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"t2.small","beta.kubernetes.io/os":"linux","failure-domain.beta.kubernetes.io/region":"ap-northeast-2","failure-domain.beta.kubernetes.io/zone":"ap-northeast-2a","kubernetes.io/hostname":"ip-10-0-30-16.ap-northeast-2.compute.internal"},"annotations":{"kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock","node.alpha.kubernetes.io/ttl":"0","volumes.kubernetes.io/controller-managed-attach-detach":"true"}}}},{"cells":["ip-10-0-7-175.ap-northeast-2.compute.internal","Ready","\u003cnone\u003e","1h","v1.11.5","10.0.7.175","\u003cnone\u003e","Ubuntu 16.04.5 LTS","4.4.0-1072-aws","docker://17.3.2"],"object":{"kind":"PartialObjectMetadata","apiVersion":"meta.k8s.io/v1beta1","metadata":{"name":"ip-10-0-7-175.ap-northeast-2.compute.internal","selfLink":"/api/v1/nodes/ip-10-0-7-175.ap-northeast-2.compute.internal","uid":"3f3c5d27-2829-11e9-a2c8-02015048da6c","resourceVersion":"7623","creationTimestamp":"2019-02-04T03:02:03Z","labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"t2.small","beta.kubernetes.io/os":"linux","failure-domain.beta.kubernetes.io/region":"ap-northeast-2","failure-domain.beta.kubernetes.io/zone":"ap-northeast-2a","kubernetes.io/hostname":"ip-10-0-7-175.ap-northeast-2.compute.internal"},"annotations":{"kubeadm.alpha.kubernetes.io/cri-socket":"/var/run/dockershim.sock","node.alpha.kubernetes.io/ttl":"0","volumes.kubernetes.io/controller-managed-attach-detach":"true"}}}}]}
I0204 13:05:46.438816   15326 get.go:558] no kind is registered for the type v1beta1.Table in scheme "k8s.io/kubernetes/pkg/api/legacyscheme/scheme.go:29"
NAME                                             STATUS   ROLES    AGE   VERSION
ip-10-0-11-90.ap-northeast-2.compute.internal    Ready    <none>   1h    v1.11.5
ip-10-0-15-49.ap-northeast-2.compute.internal    Ready    <none>   1h    v1.11.5
ip-10-0-21-21.ap-northeast-2.compute.internal    Ready    master   1h    v1.11.5
ip-10-0-23-205.ap-northeast-2.compute.internal   Ready    <none>   1h    v1.11.5
ip-10-0-30-16.ap-northeast-2.compute.internal    Ready    <none>   1h    v1.11.5
ip-10-0-7-175.ap-northeast-2.compute.internal    Ready    <none>   1h    v1.11.5
```
</details>

그러면 위 상세 로그에서 다음과 같은 `kubectl get node`에 대응하는 `curl` 명령어를 확인할 수 있다.

```shell
$ curl -k -v -XGET  -H "Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io, application/json" -H "User-Agent: kubectl/v1.12.3 (darwin/amd64) kubernetes/435f92c" 'https://13.125.59.233:6443/api/v1/nodes?limit=500'
```

위 `kubectl`을 통해서 호출되는 API 서버의 응답은 다음과 같다.

<details>
<summary>`kubectl get node` json 응답</summary>
```json
{
  "kind": "Table",
  "apiVersion": "meta.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "7633"
  },
  "columnDefinitions": [
    {
      "name": "Name",
      "type": "string",
      "format": "name",
      "description": "Name must be unique within a namespace. Is required when creating resources, although some resources may allow a client to request the generation of an appropriate name automatically. Name is primarily intended for creation idempotence and configuration definition. Cannot be updated. More info: http://kubernetes.io/docs/user-guide/identifiers#names",
      "priority": 0
    },
    {
      "name": "Status",
      "type": "string",
      "format": "",
      "description": "The status of the node",
      "priority": 0
    },
    {
      "name": "Roles",
      "type": "string",
      "format": "",
      "description": "The roles of the node",
      "priority": 0
    },
    {
      "name": "Age",
      "type": "string",
      "format": "",
      "description": "CreationTimestamp is a timestamp representing the server time when this object was created. It is not guaranteed to be set in happens-before order across separate operations. Clients may not set this value. It is represented in RFC3339 form and is in UTC.\n\nPopulated by the system. Read-only. Null for lists. More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata",
      "priority": 0
    },
    {
      "name": "Version",
      "type": "string",
      "format": "",
      "description": "Kubelet Version reported by the node.",
      "priority": 0
    },
    {
      "name": "Internal-IP",
      "type": "string",
      "format": "",
      "description": "List of addresses reachable to the node. Queried from cloud provider, if available. More info: https://kubernetes.io/docs/concepts/nodes/node/#addresses",
      "priority": 1
    },
    {
      "name": "External-IP",
      "type": "string",
      "format": "",
      "description": "List of addresses reachable to the node. Queried from cloud provider, if available. More info: https://kubernetes.io/docs/concepts/nodes/node/#addresses",
      "priority": 1
    },
    {
      "name": "OS-Image",
      "type": "string",
      "format": "",
      "description": "OS Image reported by the node from /etc/os-release (e.g. Debian GNU/Linux 7 (wheezy)).",
      "priority": 1
    },
    {
      "name": "Kernel-Version",
      "type": "string",
      "format": "",
      "description": "Kernel Version reported by the node from 'uname -r' (e.g. 3.16.0-0.bpo.4-amd64).",
      "priority": 1
    },
    {
      "name": "Container-Runtime",
      "type": "string",
      "format": "",
      "description": "ContainerRuntime Version reported by the node through runtime remote API (e.g. docker://1.5.0).",
      "priority": 1
    }
  ],
  "rows": [
    {
      "cells": [
        "ip-10-0-11-90.ap-northeast-2.compute.internal",
        "Ready",
        "<none>",
        "1h",
        "v1.11.5",
        "10.0.11.90",
        "<none>",
        "Ubuntu 16.04.5 LTS",
        "4.4.0-1072-aws",
        "docker://17.3.2"
      ],
      "object": {
        "kind": "PartialObjectMetadata",
        "apiVersion": "meta.k8s.io/v1beta1",
        "metadata": {
          "name": "ip-10-0-11-90.ap-northeast-2.compute.internal",
          "selfLink": "/api/v1/nodes/ip-10-0-11-90.ap-northeast-2.compute.internal",
          "uid": "208bde9f-2829-11e9-a2c8-02015048da6c",
          "resourceVersion": "7622",
          "creationTimestamp": "2019-02-04T03:01:11Z",
          "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "t2.small",
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
            "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2a",
            "kubernetes.io/hostname": "ip-10-0-11-90.ap-northeast-2.compute.internal"
          },
          "annotations": {
            "kubeadm.alpha.kubernetes.io/cri-socket": "/var/run/dockershim.sock",
            "node.alpha.kubernetes.io/ttl": "0",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
          }
        }
      }
    },
    {
      "cells": [
        "ip-10-0-15-49.ap-northeast-2.compute.internal",
        "Ready",
        "<none>",
        "1h",
        "v1.11.5",
        "10.0.15.49",
        "<none>",
        "Ubuntu 16.04.5 LTS",
        "4.4.0-1072-aws",
        "docker://17.3.2"
      ],
      "object": {
        "kind": "PartialObjectMetadata",
        "apiVersion": "meta.k8s.io/v1beta1",
        "metadata": {
          "name": "ip-10-0-15-49.ap-northeast-2.compute.internal",
          "selfLink": "/api/v1/nodes/ip-10-0-15-49.ap-northeast-2.compute.internal",
          "uid": "208d8cb1-2829-11e9-a2c8-02015048da6c",
          "resourceVersion": "7619",
          "creationTimestamp": "2019-02-04T03:01:11Z",
          "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "t2.small",
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
            "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2a",
            "kubernetes.io/hostname": "ip-10-0-15-49.ap-northeast-2.compute.internal"
          },
          "annotations": {
            "kubeadm.alpha.kubernetes.io/cri-socket": "/var/run/dockershim.sock",
            "node.alpha.kubernetes.io/ttl": "0",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
          }
        }
      }
    },
    {
      "cells": [
        "ip-10-0-21-21.ap-northeast-2.compute.internal",
        "Ready",
        "master",
        "1h",
        "v1.11.5",
        "10.0.21.21",
        "<none>",
        "Ubuntu 16.04.5 LTS",
        "4.4.0-1072-aws",
        "docker://17.3.2"
      ],
      "object": {
        "kind": "PartialObjectMetadata",
        "apiVersion": "meta.k8s.io/v1beta1",
        "metadata": {
          "name": "ip-10-0-21-21.ap-northeast-2.compute.internal",
          "selfLink": "/api/v1/nodes/ip-10-0-21-21.ap-northeast-2.compute.internal",
          "uid": "af849c71-2828-11e9-a2c8-02015048da6c",
          "resourceVersion": "7626",
          "creationTimestamp": "2019-02-04T02:58:02Z",
          "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "t2.small",
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
            "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2a",
            "kubernetes.io/hostname": "ip-10-0-21-21.ap-northeast-2.compute.internal",
            "node-role.kubernetes.io/master": ""
          },
          "annotations": {
            "kubeadm.alpha.kubernetes.io/cri-socket": "/var/run/dockershim.sock",
            "node.alpha.kubernetes.io/ttl": "0",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
          }
        }
      }
    },
    {
      "cells": [
        "ip-10-0-23-205.ap-northeast-2.compute.internal",
        "Ready",
        "<none>",
        "1h",
        "v1.11.5",
        "10.0.23.205",
        "<none>",
        "Ubuntu 16.04.5 LTS",
        "4.4.0-1072-aws",
        "docker://17.3.2"
      ],
      "object": {
        "kind": "PartialObjectMetadata",
        "apiVersion": "meta.k8s.io/v1beta1",
        "metadata": {
          "name": "ip-10-0-23-205.ap-northeast-2.compute.internal",
          "selfLink": "/api/v1/nodes/ip-10-0-23-205.ap-northeast-2.compute.internal",
          "uid": "22442c26-2829-11e9-a2c8-02015048da6c",
          "resourceVersion": "7628",
          "creationTimestamp": "2019-02-04T03:01:14Z",
          "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "t2.small",
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
            "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2a",
            "kubernetes.io/hostname": "ip-10-0-23-205.ap-northeast-2.compute.internal"
          },
          "annotations": {
            "kubeadm.alpha.kubernetes.io/cri-socket": "/var/run/dockershim.sock",
            "node.alpha.kubernetes.io/ttl": "0",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
          }
        }
      }
    },
    {
      "cells": [
        "ip-10-0-30-16.ap-northeast-2.compute.internal",
        "Ready",
        "<none>",
        "1h",
        "v1.11.5",
        "10.0.30.16",
        "<none>",
        "Ubuntu 16.04.5 LTS",
        "4.4.0-1072-aws",
        "docker://17.3.2"
      ],
      "object": {
        "kind": "PartialObjectMetadata",
        "apiVersion": "meta.k8s.io/v1beta1",
        "metadata": {
          "name": "ip-10-0-30-16.ap-northeast-2.compute.internal",
          "selfLink": "/api/v1/nodes/ip-10-0-30-16.ap-northeast-2.compute.internal",
          "uid": "1c7ce276-2829-11e9-a2c8-02015048da6c",
          "resourceVersion": "7627",
          "creationTimestamp": "2019-02-04T03:01:04Z",
          "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "t2.small",
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
            "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2a",
            "kubernetes.io/hostname": "ip-10-0-30-16.ap-northeast-2.compute.internal"
          },
          "annotations": {
            "kubeadm.alpha.kubernetes.io/cri-socket": "/var/run/dockershim.sock",
            "node.alpha.kubernetes.io/ttl": "0",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
          }
        }
      }
    },
    {
      "cells": [
        "ip-10-0-7-175.ap-northeast-2.compute.internal",
        "Ready",
        "<none>",
        "1h",
        "v1.11.5",
        "10.0.7.175",
        "<none>",
        "Ubuntu 16.04.5 LTS",
        "4.4.0-1072-aws",
        "docker://17.3.2"
      ],
      "object": {
        "kind": "PartialObjectMetadata",
        "apiVersion": "meta.k8s.io/v1beta1",
        "metadata": {
          "name": "ip-10-0-7-175.ap-northeast-2.compute.internal",
          "selfLink": "/api/v1/nodes/ip-10-0-7-175.ap-northeast-2.compute.internal",
          "uid": "3f3c5d27-2829-11e9-a2c8-02015048da6c",
          "resourceVersion": "7623",
          "creationTimestamp": "2019-02-04T03:02:03Z",
          "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/instance-type": "t2.small",
            "beta.kubernetes.io/os": "linux",
            "failure-domain.beta.kubernetes.io/region": "ap-northeast-2",
            "failure-domain.beta.kubernetes.io/zone": "ap-northeast-2a",
            "kubernetes.io/hostname": "ip-10-0-7-175.ap-northeast-2.compute.internal"
          },
          "annotations": {
            "kubeadm.alpha.kubernetes.io/cri-socket": "/var/run/dockershim.sock",
            "node.alpha.kubernetes.io/ttl": "0",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
          }
        }
      }
    }
  ]
}
```
</details>

위 `curl` 명령을 `kubeconfig`에 인증서없이 그대로 호출하게 되면, 다음과 같이 API 서버로부터 `403` 인증 오류가 반환된다. 

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "nodes is forbidden: User \"system:anonymous\" cannot list nodes at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "nodes"
  },
  "code": 403
}
```

### client-certificate-data, client-key-data

쿠버네티스는 서버와 클라이언트가 각각 상호 인증(mutual authentication)한다.

만약, `kubeconfig` 파일의 `certificate-authority-data`에 데이터가 잘못되거나 값이 삭제될 경우에는 다음과 같이 서버 연결이 사이닝되지 않았다는 오류를 볼 수 있다.

```shell
$ kubectl get node
Unable to connect to the server: x509: certificate signed by unknown authority
```

API 서버에 대해 보안 연결이 맺어져도 클라이언트 인증이 되어야 제한된 리소스에 접근이 가능하다. 이 때 클라이언트 인증시에 필요한 CSR과 프라이빗 키의 데이터가 각각 `client-certificate-data`, `client-key-data`에 기록된다. 

## 인증 (Authentication)

쿠버네티스 클라이언트 컴포넌트가 API 서버에 연결을 맺을 때, 쿠버네티스가 사용자 ID를 할당하는 여러 가지 방법이 있을 수 있는데, 쿠버네티스에는 다음과 같은 인증 방법이 있다. (참고 - [Authentication strategies](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authentication-strategies))

- X509 Client Certs: 괜찮은 인증 방법이지만, 정기적으로 클라이언트 인증서 갱신 및 재배포를 처리해야 한다.
- Static Token File: 일시적이지 않은 특성 때문에 추천하지 않는다.
- Bootstrap Tokens: 위의 정적 토큰과 동일하다.
- Static Password File: API 서버를 재시작하지 않으면 패스워드를 변경할 수 없기 때문에 추천하지 않는다.
- Service Account Tokens: 최종 사용자가 쿠버네티스 클러스터와 상호 작용하려는 경우 사용해서는 안되나, 쿠버네티스에서 실행되는 응용 프로그램 및 작업 부하에 대해서는 추천할 수 있는 인증 방법이다.
- OpenID Connect Tokens: OIDC가 AWS IAM 등 ID 제공 업체와 통합되므로 최종 사용자를 위한 최상의 인증 방법이다.
- Webhook Token Authentication: OIDC와 마찬가지로 Webhook으로 GitHub 개인 토큰을 통합하여 최종 사용자가 사용할 수 있는 인증 방법이다. 
- Authentication Proxy: `X-Remote-User`와 같은 요청 헤더의 값을 설정하는 인증 프락시와 통합시에 사용하는 인증 방법이다.

클라이언트 인증서를 생성하는 방법은 다음과 같다. (참고 - [Manage TLS Certificates in a Cluster](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/))

영상에서는 클라이언트 인증을 위한 임의의 사용자 계정(예, `users:ziwon`)을 추가하는 예제를 보이고 있다.

먼저 개인키를 생성한다.

```shell
$ openssl genrsa -out ziwon.pem 2048
```

다음으로 인증서 발급에 필요한 CSR(Certificate Signing Request, 인증서 서명 요청)을 생성한다. 

```shell
$ openssl req -new -key ziwon.pem -out ziwon.csr -subj "/CN=users:ziwon/O=cloud-native"
```

그리고 생성된 CSR을 쿠버테스트에 등록하기 위해 base64로 인코딩한 값을 추출한다.

```shell
$ cat ziwon.csr | base64 | tr -d '\n'
```

생성된 쿠버네티스 csr을 등록한다.

```
$ cat << EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: user-request-ziwon
spec:
  groups:
  - system:authenticated
  request: ...
  usages:
  - digital signature
  - key encipherment
  - client auth
EOF
```

등록 후, `pending` 상태에 있는 `user-request-ziwon`의 값을 승인한다.

```shell
$ kubectl certificate approve user-request-ziwon
```

승인된 `user-request-ziwon` CSR로부터 공개키를 생성한다.

```shell
$ kubectl get csr user-request-ziwon -o jsonpath='{.status.certificate}' | base64 -d > ziwon.crt
```

쿠버네티스 API 서버의 Root CA, 그리고 클라이언트의 공개키와 개인키를 이용해, 클라이언트 `kubeconfig` 파일을 생성한다.

```shell
cat ziwon.crt | base64 | tr -d '\n'
```

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <base64-encoded-ca-cert>
    server: https://localhost:53079
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: ziwon
  name: ziwon@kubernetes
current-context: ziwon@kubernetes
kind: Config
preferences: {}
users:
- name: ziwon
  user:
    client-certificate-data: <base64-encoded-crt>
    client-key-data: <base64-encoded-key-data>
```

그러면 다음과 같이 API 서버에 대해 클라이언트 ID `users:ziwon` 로 접근이 되는 것을 볼 수 있다. 그러나 접근이 되더라도 권한(Authoriztion)을 부여받은 Role이 지정되지 않았다. 따라서, 클러스터 리소스 및 네임스페이스 리소스 접근시 각각 오류 응답을 받게 된다.

- 클러스터 객체 접근시

```shell
$ kubectl get node
Error from server (Forbidden): nodes is forbidden: User "users:ziwon" cannot list resource "nodes" in API group "" at the cluster scope
```

- 네임스페이스 객체 접근시 

```shell
$ kubectl get pod
Error from server (Forbidden): pods is forbidden: User "users:ziwon" cannot list resource "pods" in API group "" in the namespace "default"
```

## Authorization

모든 쿠버네티스트 리소스는 다음과 같이 클러스터 객체와 네임스페이스 객체로 구분할 수 있다.

### 클러스터 객체
```shell
$ kubectl api-resources --namespaced=false
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
componentstatuses                 cs                                          false        ComponentStatus
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumes                 pv                                          false        PersistentVolume
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
tokenreviews                                   authentication.k8s.io          false        TokenReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
podsecuritypolicies               psp          extensions                     false        PodSecurityPolicy
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```

### 네임스페이스 객체

```shell
$ kubectl api-resources --namespaced
NAME                        SHORTNAMES   APIGROUP                    NAMESPACED   KIND
bindings                                                             true         Binding
configmaps                  cm                                       true         ConfigMap
endpoints                   ep                                       true         Endpoints
events                      ev                                       true         Event
limitranges                 limits                                   true         LimitRange
persistentvolumeclaims      pvc                                      true         PersistentVolumeClaim
pods                        po                                       true         Pod
podtemplates                                                         true         PodTemplate
replicationcontrollers      rc                                       true         ReplicationController
resourcequotas              quota                                    true         ResourceQuota
secrets                                                              true         Secret
serviceaccounts             sa                                       true         ServiceAccount
services                    svc                                      true         Service
controllerrevisions                      apps                        true         ControllerRevision
daemonsets                  ds           apps                        true         DaemonSet
deployments                 deploy       apps                        true         Deployment
replicasets                 rs           apps                        true         ReplicaSet
statefulsets                sts          apps                        true         StatefulSet
localsubjectaccessreviews                authorization.k8s.io        true         LocalSubjectAccessReview
horizontalpodautoscalers    hpa          autoscaling                 true         HorizontalPodAutoscaler
cronjobs                    cj           batch                       true         CronJob
jobs                                     batch                       true         Job
leases                                   coordination.k8s.io         true         Lease
events                      ev           events.k8s.io               true         Event
daemonsets                  ds           extensions                  true         DaemonSet
deployments                 deploy       extensions                  true         Deployment
ingresses                   ing          extensions                  true         Ingress
networkpolicies             netpol       extensions                  true         NetworkPolicy
replicasets                 rs           extensions                  true         ReplicaSet
networkpolicies             netpol       networking.k8s.io           true         NetworkPolicy
poddisruptionbudgets        pdb          policy                      true         PodDisruptionBudget
rolebindings                             rbac.authorization.k8s.io   true         RoleBinding
roles                                    rbac.authorization.k8s.io   true         Role
```

다음과 같이 RBAC에 의한 리소스 접근시에 권한과 관련된 네임스페이스 객체와 클러스터 객체가 있다.

### 네임스페이스
- Role
- RoleBinding

쿠버네티스의 네임스페이스에 정의된 기본적인 `role` 목록은 다음과 같다.

```shell
$ kubectl get role --all-namespaces
NAMESPACE     NAME                                             AGE
kube-public   kubeadm:bootstrap-signer-clusterinfo             8h
kube-public   system:controller:bootstrap-signer               8h
kube-system   extension-apiserver-authentication-reader        8h
kube-system   kube-proxy                                       8h
kube-system   kubeadm:kubelet-config-1.13                      8h
kube-system   kubeadm:nodes-kubeadm-config                     8h
kube-system   system::leader-locking-kube-controller-manager   8h
kube-system   system::leader-locking-kube-scheduler            8h
kube-system   system:controller:bootstrap-signer               8h
kube-system   system:controller:cloud-provider                 8h
kube-system   system:controller:token-cleaner                  8h
kube-system   weave-net                                        8h
```

### 클러스터 
- ClusterRole
- ClusterRoleBinding

쿠버네티스의 클러스터에 정의된 기본적인 `clusterrole` 목록은 다음과 같다.

```shell
$ kubectl get clusterrole
NAME                                                                   AGE
admin                                                                  8h
cluster-admin                                                          8h
edit                                                                   8h
system:aggregate-to-admin                                              8h
system:aggregate-to-edit                                               8h
system:aggregate-to-view                                               8h
system:auth-delegator                                                  8h
system:aws-cloud-provider                                              8h
system:basic-user                                                      8h
system:certificates.k8s.io:certificatesigningrequests:nodeclient       8h
system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   8h
system:controller:attachdetach-controller                              8h
system:controller:certificate-controller                               8h
system:controller:clusterrole-aggregation-controller                   8h
system:controller:cronjob-controller                                   8h
system:controller:daemon-set-controller                                8h
system:controller:deployment-controller                                8h
system:controller:disruption-controller                                8h
system:controller:endpoint-controller                                  8h
system:controller:expand-controller                                    8h
system:controller:generic-garbage-collector                            8h
system:controller:horizontal-pod-autoscaler                            8h
system:controller:job-controller                                       8h
system:controller:namespace-controller                                 8h
system:controller:node-controller                                      8h
system:controller:persistent-volume-binder                             8h
system:controller:pod-garbage-collector                                8h
system:controller:pv-protection-controller                             8h
system:controller:pvc-protection-controller                            8h
system:controller:replicaset-controller                                8h
system:controller:replication-controller                               8h
system:controller:resourcequota-controller                             8h
system:controller:route-controller                                     8h
system:controller:service-account-controller                           8h
system:controller:service-controller                                   8h
system:controller:statefulset-controller                               8h
system:controller:ttl-controller                                       8h
system:coredns                                                         8h
system:csi-external-attacher                                           8h
system:csi-external-provisioner                                        8h
system:discovery                                                       8h
system:heapster                                                        8h
system:kube-aggregator                                                 8h
system:kube-controller-manager                                         8h
system:kube-dns                                                        8h
system:kube-scheduler                                                  8h
system:kubelet-api-admin                                               8h
system:node                                                            8h
system:node-bootstrapper                                               8h
system:node-problem-detector                                           8h
system:node-proxier                                                    8h
system:persistent-volume-provisioner                                   8h
system:volume-scheduler                                                8h
view                                                                   8h
weave-net                                                              8h
```

먼저, 클러스터 전체에 대해 정의된 기본 사용자 `system:basic-user`의 RBAC을 확인해보자.

<details>
<summary>$ kubectl get clusterrole system:basic-user -o yaml</summary>
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2019-02-04T05:11:12Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:basic-user
  resourceVersion: "43"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/system%3Abasic-user
  uid: 49fa84df-283b-11e9-9ac2-0242c86ffe48
rules:
- apiGroups:
  - authorization.k8s.io
  resources:
  - selfsubjectaccessreviews
  - selfsubjectrulesreviews
  verbs:
  - create
```
</details>

다음으로 클러스터 전체를 관리하는 관리자 `admin`의 RBAC을 확인해보자.

<details>
<summary>$ kubectl get clusterrole admin -o yaml</summary>
```yaml
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-admin: "true"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2019-02-04T05:11:12Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: admin
  resourceVersion: "350"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/admin
  uid: 49faedb8-283b-11e9-9ac2-0242c86ffe48
rules:
- apiGroups:
  - ""
  resources:
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  - secrets
  - services/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - serviceaccounts
  verbs:
  - impersonate
- apiGroups:
  - ""
  resources:
  - pods
  - pods/attach
  - pods/exec
  - pods/portforward
  - pods/proxy
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - replicationcontrollers
  - replicationcontrollers/scale
  - secrets
  - serviceaccounts
  - services
  - services/proxy
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - deployments/rollback
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  - statefulsets/scale
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/rollback
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - create
  - delete
  - deletecollection
  - patch
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - controllerrevisions
  - daemonsets
  - deployments
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  - statefulsets/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - authorization.k8s.io
  resources:
  - localsubjectaccessreviews
  verbs:
  - create
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - rolebindings
  - roles
  verbs:
  - create
  - delete
  - deletecollection
  - get
  - list
  - patch
  - update
  - watch
```

```shell
$ kubectl create rolebinding ziwon --clusterrole=admin --user=users:ziwon --dry-run -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: ziwon
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: users:ziwon
```
</details>

먼저, `foo`라는 네임스페이스를 만든다. 이때 기본 `~/.kube/config`를 사용한다.

```shell
$ kubectl run --restart=Never --image=gcr.io/kuar-demo/kuard-amd64:blue kuard
```

그리고 새로운 쉘에서 새로 추가된 사용자 `users:ziwon`의 `kubeconfig-ziwon`를 이용해 Pod을 하나 배포해본다.

```shell
$ export KUBECONFIG=/path/to/kubeconfig-ziwon
$ kubectl config set-context $(kubectl config current-context) --namespace=foo
$ kubectl run --restart=Never --image=gcr.io/kuar-demo/kuard-amd64:blue kuard

Error from server (Forbidden): pods is forbidden: User "users:ziwon" cannot create resource "pods" in API group "" in the namespace "foo"
```

배포가 되지 않는 것을 확인할 수 있다. 이에, 다시 어드민 계정으로 사용자 `users:ziwon`에 대한 `Pod` 등의 네임스페이스 객체 접근을 위한 `rolebinding`과 클러스터 객체 접근을 위한 `clusterrolebinding`을 만든다.

```shell
$ export KUBECONFIG=~/.kube/config
$ kubectl create rolebinding ziwon-role --clusterrole=admin --user=users:ziwon
$ kubectl create clusterrolebinding ziwon-clusterrole --clusterrole=view --user=users:ziwon
```

그러면 다음과 같이 정상적으로 배포가 되고, 네임스페이스 Pod 객체 목록을 정상적으로 확인할 수 있다.

```shell
$ kubectl run --restart=Never --image=gcr.io/kuar-demo/kuard-amd64:blue kuard
$ kubectl get pod -n foo
NAME    READY   STATUS    RESTARTS   AGE
kuard   1/1     Running   0          17s
```

또한 다음과 같이, `clusterrolebinding`이 부여되었기 때문에 클러스터 전체 Pod 목록도 확인할 수 있다.

```shell
$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                            READY   STATUS    RESTARTS   AGE
default       kuard                                           1/1     Running   0          10m
foo           kuard                                           1/1     Running   0          2m14s
kube-system   coredns-86c58d9df4-98rkf                        1/1     Running   0          9h
kube-system   coredns-86c58d9df4-zt4l8                        1/1     Running   0          9h
kube-system   etcd-kind-1-control-plane1                      1/1     Running   0          9h
kube-system   etcd-kind-1-control-plane2                      1/1     Running   0          9h
kube-system   kube-apiserver-kind-1-control-plane1            1/1     Running   0          9h
kube-system   kube-apiserver-kind-1-control-plane2            1/1     Running   0          9h
kube-system   kube-controller-manager-kind-1-control-plane1   1/1     Running   1          9h
kube-system   kube-controller-manager-kind-1-control-plane2   1/1     Running   0          9h
kube-system   kube-proxy-6cmkr                                1/1     Running   0          9h
kube-system   kube-proxy-bzbwk                                1/1     Running   0          9h
kube-system   kube-proxy-hlq4l                                1/1     Running   0          9h
kube-system   kube-proxy-jl9sg                                1/1     Running   0          9h
kube-system   kube-proxy-mw2rt                                1/1     Running   0          9h
kube-system   kube-proxy-p9pzv                                1/1     Running   0          9h
kube-system   kube-proxy-tjs8r                                1/1     Running   0          9h
kube-system   kube-proxy-tvgz7                                1/1     Running   0          9h
kube-system   kube-scheduler-kind-1-control-plane1            1/1     Running   1          9h
kube-system   kube-scheduler-kind-1-control-plane2            1/1     Running   0          9h
kube-system   weave-net-52l8x                                 2/2     Running   0          9h
kube-system   weave-net-5f4lb                                 2/2     Running   0          9h
kube-system   weave-net-6d9dx                                 2/2     Running   0          9h
kube-system   weave-net-6nhns                                 2/2     Running   1          9h
kube-system   weave-net-6t7tw                                 2/2     Running   0          9h
kube-system   weave-net-hqlw6                                 2/2     Running   0          9h
kube-system   weave-net-m2dn4                                 2/2     Running   0          9h
kube-system   weave-net-nrsvs                                 2/2     Running   0          9h
```

## 관련 링크
- [RSA 인증서 (Certification) 와 전자서명 (Digital Sign)의 원리](https://rsec.kr/?p=426)
- [Adding users on "Quick Start for Kubernetes on AWS"](https://www.linkedin.com/pulse/adding-users-quick-start-kubernetes-aws-jakub-scholz)
- [How to: RBAC best practices and workarounds](http://docs.heptio.com/content/tutorials/rbac.html)
- [How Kubernetes certificate authorities work](https://jvns.ca/blog/2017/08/05/how-kubernetes-certificates-work/)
- [Kubernetes Security Best-Practices](https://dev.to/petermbenjamin/kubernetes-security-best-practices-hlk#authentication)