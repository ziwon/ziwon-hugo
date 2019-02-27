
+++
date          = "2019-02-27T00:17:00+09:00"
draft         = true
title         = "[번역] 쿠버네티스 네트워킹 모델의 이해"
tags          = ["Networking", "Kubernetes", "Networking"]
categories    = ["DevOps"]
slug          = "understanding-kubernetes-networking-model"
toc           = true
socialsharing = true
nocomment     = false
+++

> 이 글은 원저자 **Kenvin Sookocheff** 님의 동의를 받아 한국어로 번역하였습니다. 원문의 출저는 [A Guide to the Kubernetes Networking Model](https://sookocheff.com/post/kubernetes/understanding-kubernetes-networking-model/)입니다.


Kubernetes was built to run distributed systems over a cluster of machines. The very nature of distributed systems makes networking a central and necessary component of Kubernetes deployment, and understanding the Kubernetes networking model will allow you to correctly run, monitor and troubleshoot your applications running on Kubernetes.

쿠버네티스는 머신 클러스터에서 분산된 시스템을 실행하기 위해 만들어졌습니다. 분산 시스템의 그 특성으로 인해 네트워킹은 쿠버네티스 배포의 핵심적이고 필요한 컴포넌트이며, 쿠버네티스 네트워킹 모델을 이해하면 쿠버네티스에서 실행 중인 애플리케이션을 올바르게 실행하고, 모니터링하고, 문제를 해결할 수 있을 것입니다.

Networking is a vast space with a lot of mature technologies. For people unfamiliar with the landscape, this can be uncomfortable because most people have existing preconceived notions about networking, and there are a lot of both new and old concepts to understand and fit together into a coherent whole. A non-exhaustive list might include technologies like network namespaces, virtual interfaces, IP forwarding, and network address translation. This guide intends to demystify Kubernetes networking by discussing each of Kubernetes dependent technologies along with descriptions on how those technologies are used to enable the Kubernetes networking model.

네트워킹은 성숙한 기술을 많이 지닌 거대한 공간입니다. 이 풍경에 익숙하지 않은 사람들에게는 네트워킹은 불편할 수 있습니다. 왜냐하면 대부분의 사람들이 네트워킹에 대한 기존의 선입견을 가지고 있고, 일관성이 있는 전체로 이해하고 맞추어야 하는 새로운 개념들과 오래된 개념들이 많이 있기 때문입니다. 대략적인 목록으로 네트워크 네임스페이스, 가상 인터페이스, IP 전달 및 네트워크 주소 변환과 같은 것들입니다. 본 가이드는 쿠버네티스의 네트워킹 기술을 사용하기 위해 이러한 기술이 어떻게 사용되지는지에 대한 설명과 함께 쿠버네티스가 의존하는 기술 각각에 대해 논의함으로써 쿠버네티스 네트워킹을 설명하고자 합니다. 

This guide is fairly long and divided in several sections. We start by discussing some basic Kubernetes terminology to ensure terms are being used correctly throughout the guide, then discuss the Kubernetes networking model and the design and implementation decisions that it imposes. This is followed by the longest and most interesting part of this guide: an in-depth discussion on how traffic is routed within Kubernetes using several different use cases.

본 가이드는 상당히 길며 여러 섹션으로 나뉩니다. 먼저 쿠버네티스의 기본 용어 몇 가지를 논의하여 가이드 전체에서 용어가 올바르게 사용되는지 확인한 다음 쿠버네티스 네트워킹 모델과 이 모델이 내세우는 설계 및 구현 결정에 대해 논의합니다. 이는 본 가이드의 가장 길고 흥미로운 부분으로 서로 다른 사용 사례를 사용해서 쿠버네티스 내에서 트래픽을 라우팅하는 방법에 대한 심도있는 논의가 될 것입니다.