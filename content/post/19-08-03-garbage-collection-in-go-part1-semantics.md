
+++
date          = "2019-08-03T23:46:00+09:00"
draft         = true
title         = "[번역] Go 가비지 콜렉션: 파트 1 - 세만틱스"
tags          = ["Go", "GC"]
categories    = ["Programming"]
slug          = "garbage-collection-in-go-part1-semantics"
toc           = true
socialsharing = true
nocomment     = false
+++

원문의 출처는 [Garbage Collection In Go : Part I - Semantics](https://www.ardanlabs.com/blog/2018/12/garbage-collection-in-go-part1-semantics.html)입니다. 
> 
> 원본 동영상 강의 - [Ultimate Go Programming, Second Edition](https://learning.oreilly.com/videos/ultimate-go-programming/9780135261651), ([GitHub](https://github.com/ardanlabs/gotraining/tree/master/topics/go/language/pointers) 자료)

다른 분이 번역하신 Go 스케쥴링에 대한 부분을 먼저 읽으셔도 좋습니다.
 
> - [Go 스케쥴링 파트1 - OS 스케쥴러](https://marsettler.com/2018/10/02/scheduling-in-go-part1.html)
> - [Go 스케쥴링 파트2 - Go 스케쥴](https://marsettler.com/2018/10/03/scheduling-in-go-part2.html)
> - [Go 스케쥴링 파트3 - 동시성](https://marsettler.com/2018/12/08/scheduling-in-go-part3.html)


# 서두

이것은 Go 가비지 콜렉터에 숨겨진 역학과 의미론에 대해 이해를 돕기 위한 3 개의 시리즈 중 첫 번째 포스트이다.
이 포스트는 콜렉터의 의미론에 관한 기본 자료에 대해 중점을 둔다.

3개의 파트로 구성된 시리즈의 순서는 다음과 같다:

- 1) [Go 가비지 콜렉션: 파트 1 - Semantics](garbage-collection-in-go-part1-semantics)
- 2) Go 가비지 콜렉션: 파트 2 - GC Traces
- 3) Go 가비지 콜렉션: 파트 3 - GC Pacing

# 입문

가비지 콜렉터는 힙(heap) 메모리 할당은 추적하고, 더 이상 필요하지 않은 할당은 해제하고, 아직 사용 중인 할당은 유지해야 합니다. 프로그래밍 언어가 이러한 행위들을 시행하기로 하는 방법들은 복잡하지만, 응용 애플리케이션 개발자가 소프트웨어를 제작하기 위해 세부 사항을 알아야할 필요는 없습니다. 또한 이러한 시스템의 구현은 언어마다 다른 VM 또는 런타임 릴리스로 항상 변화하고 발전하고 있습니다. 응용 애플리케이션 개발자들에게 중요한 것은 언어의 가비지 콜렉터가 어떻게 동작하는지, 구현에 대해 염려하지 않고 그들이 어떻게 그 동작들에 동감할 수 있는지에 대한 훌륭한 작업 모델을 갖는 것이다. 

버전 1.12을 기준으로, Go 프로그래밍 언어는 비세대 동시 트리컬러 마크앤 스위프 콜렉터(non-generational concurrent tri-color mark and sweep collector)를 사용한다. 마크 앤 스위프 콜렉터가 어떻게 동작하는지를 시각적으로 알고 싶다면, 켄 폭스가 쓴 훌륭한 기사와 제공하는 애니메이션을 보라. Go의 콜렉터 구현은 Go가 릴리즈될 때마다 변경되고 진화해왔다. 그래서, 구현 상세에 대해 이야기하는 어떤 포스트도 언어의 다음 버전이 릴리즈되면 더 이상 정확하지 않을 것이다.

말한 것처럼, 이 포스트에서 할 모델링은 실제 구현된 세부사항에 초첨을 맞추지 않을 것이다. 모델링은 여러분이 경험하게 될 동작과 앞으로 보게 될 동작들에 초점을 맞출 것이다. 이 포스트에서 나는 콜렉터의 동작을 공유하고 현재의 구현이나 향후의 변경과는 무관하게 그 동작에 공감하는 방법을 설명하겠다. 이로 인해서 더 좋은 Go 개발자가 될 수 있을 것이다.

참고: [여기](https://github.com/ardanlabs/gotraining/tree/master/reading#garbage-collection)와 Go의 실제 콜렉터에 대해 할 수 있는 더 많은 읽을 거리가 있다.  