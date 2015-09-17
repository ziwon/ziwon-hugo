+++
date          = "2015-09-04T00:27:35+09:00"
draft         = false
title         = "Falcon REST API"
tags          = [ "rest", "python", "falcon", "api"]
categories    = [ "Programming"]
slug          = "falcon-rest-template"
notoc         = true
socialsharing = true
nocomment     = false
+++

Cloud API와 백엔드 개발에 경량화된 파이썬 프레임워크인 [Falcon](http://falconframework.org)를 이용해 간단한 REST API 템플릿을 만들어 보았다. 개발하는 느낌은 Flask와 비슷한데 (음.. Flask와 뭐가 다르지?) 미들웨어, 후킹 데코레이션 등을 이용해 HTTP 요청과 응답의 입출력에 대해 체이닝 처리를 좀 더 직관적으로 할 수 있다. 성능이 Flask보다 좀 더 빠른 것으로 알려져 있다. Rackspace에서 밀고있는 Cloud API 전용 프레임워크인데, 최신 릴리즈 버전이 0.3이다. 심플하고 미니멀한 걸 좋아한다면 해볼만하다.

문서에도 나와있지만, Cython을 이용하면 약 20% 정도의 성능효과를 볼 수 있다.

```
pip install --upgrade cython falcon
```

다음은 Go와 성능비교이다.

**falcon rest api with gunicornt (-w 9 -k gevent)**
```
wrk -t10 -c100 -d30s http://localhost:5000
Running 30s test @ http://localhost:5000
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   221.79ms  414.03ms   2.00s    85.43%
    Req/Sec   755.94    486.41     2.77k    64.26%
  139452 requests in 30.08s, 32.98MB read
  Socket errors: connect 0, read 0, write 0, timeout 389
Requests/sec:   4636.13
Transfer/sec:      1.10MB
```

**sample http server with gogin only**
```
wrk -t10 -c100 -d30s http://localhost:8080
Running 30s test @ http://localhost:8080
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    11.85ms   10.12ms 250.01ms   74.03%
    Req/Sec     0.88k   124.97     2.56k    75.99%
  262302 requests in 30.10s, 35.77MB read
Requests/sec:   8713.78
Transfer/sec:      1.19MB
```

참고로 테스트 하드웨어 장비는 다음과 같다. (구린 내 아이맥ㅠ)
```
  Model Name:	iMac
  Model Identifier:	iMac11,3
  Processor Name:	Intel Core i5
  Processor Speed:	2.8 GHz
  Number of Processors:	1
  Total Number of Cores:	4
  L2 Cache (per Core):	256 KB
  L3 Cache:	8 MB
  Memory:	12 GB
  Processor Interconnect Speed:	4.8 GT/s
  Boot ROM Version:	IM112.0057.B01
  SMC Version (system):	1.59f2
```
