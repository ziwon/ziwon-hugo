+++
date = "2015-09-04T00:27:35+09:00"
draft = false
title = "Falcon Rest API Template"
tags        = [ "rest", "python", "falcon", "api"]
categories  = [ "Programming"]
slug        = "falcon-rest-template"
notoc       = true
+++

[Falcon](http://falconframework.org)를 이용해 간단한 REST API 템플릿을 만들어 보았다. Falcon은 Cloud API와 백엔드를 개발을 위해 만든 경량화된 고성능 파이썬 웹프레임워크이다.

https://github.com/patriz/falcon-rest-template

문서에도 나와있지만, Cython을 이용하면 약 20% 정도의 성능효과를 볼 수 있다.

```
pip install --upgrade cython falcon
```
