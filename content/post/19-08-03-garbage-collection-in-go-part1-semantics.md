
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

원문 - [Garbage Collection In Go : Part I - Semantics](https://www.ardanlabs.com/blog/2018/12/garbage-collection-in-go-part1-semantics.html)
> 
> 원본 동영상 강의 - [Ultimate Go Programming, Second Edition](https://learning.oreilly.com/videos/ultimate-go-programming/9780135261651), ([GitHub](https://github.com/ardanlabs/gotraining/tree/master/topics/go/language/pointers) 자료)

참고 - Go 스케쥴링 (한글 번역)
 
> - [Go 스케쥴링 파트1 - OS 스케쥴러](https://marsettler.com/2018/10/02/scheduling-in-go-part1.html)
> - [Go 스케쥴링 파트2 - Go 스케쥴](https://marsettler.com/2018/10/03/scheduling-in-go-part2.html)
> - [Go 스케쥴링 파트3 - 동시성](https://marsettler.com/2018/12/08/scheduling-in-go-part3.html)


## 서두

이 포스트는 Go 가비지 콜렉터에 숨겨진 메커니즘과 세만틱스에 대해 이해를 돕는 3부작 시리즈 중 첫 번째 포스트이다. 이 포스트는 콜렉터 세만틱스에 관한 기본 내용에 대해 중점을 둔다.

3부작 시리즈의 인덱스는 다음과 같다:

- 1) [Go 가비지 콜렉션: 파트 1 - Semantics](garbage-collection-in-go-part1-semantics)
- 2) Go 가비지 콜렉션: 파트 2 - GC Traces
- 3) Go 가비지 콜렉션: 파트 3 - GC Pacing

## 입문

가비지 콜렉터는 힙(heap) 메모리 할당을 추적하고, 더 이상 필요하지 않은 메모리 할당을 해제하고, 아직 사용 중인 메모리 할당을 유지해야 한다. 프로그래밍 언어가 이러한 동작들을 시행하기로 결정하는 방법은 복잡하지만, 소프트웨어를 제작하려는 응용 애플리케이션 개발자가 세부 사항을 알아야 할 필요는 없다. 또한 이러한 시스템의 구현은 언어의 다른 VM 또는 런타임 릴리스마다 항상 변경되며 진화해가고 있다. 응용 애플리케이션 개발자들에게 중요한 것은 언어의 가비지 콜렉터가 어떻게 동작하는지 그리고 그들이 구현에 대해 염려하지 않고 어떻게 그 동작들에 공감할 수 있는지에 대해 훌륭한 워킹 모델을 가져가는 것이다.

버전 1.12을 기준으로, Go 프로그래밍 언어는 비세대 동시 트리컬러 마크앤 스위프 콜렉터(non-generational concurrent tri-color mark and sweep collector)를 사용한다. 마크 앤 스위프 콜렉터가 어떻게 동작하는지를 시각적으로 알고 싶다면, 켄 폭스가 쓴 훌륭한 아티클과 제공하는 애니메이션을 보라. Go의 콜렉터 구현은 Go가 릴리즈될 때마다 변경되고 진화해왔다. 그래서, 다음 버전이 릴리즈되면 구현 상세에 대해 말하는 모든 포스트들이 더 이상 정확하지 않게 된다.

말한 것처럼, 이 포스트에서 가지려는 모델링은 실제 구현된 세부사항에 초첨을 맞추지 않을 것이다. 여러분이 경험하게 될 동작과 앞으로 보게 될 동작들에 초점을 맞추며 모델링해갈 것이다. 이 포스트에서. 나는 콜렉터의 동작을 공유하고 현재의 구현이나 향후의 변경과는 무관하게 그 동작과 공감해가는 방법을 설명하겠다. 이로 인해서 더 좋은 Go 개발자가 될 수 있을 것이다.

참고: [여기](https://github.com/ardanlabs/gotraining/tree/master/reading#garbage-collection)에 Go의 실제 콜렉터에 대해 할 수 있는 더 많은 읽을 거리가 있다.

## 힙은 컨테이너가 아니다.

나는 절대로 힙을 값을 저장하거나 해제할 수 있는 컨테이너라고 언급하지 않을 것이다. "힙(Heap)"을 정의하는 선형적인 컨테인먼트가 없다는 것을 이해하는 것이 중요하다. 프로세스 공간에서 애플리케이션용으로 예약된 모든 메모리는 힙 메모리 할당에 사용할 수 있다고 생각하라. 주어진 힙 메모리 할당이 가상으로 또는 물리적으로 저장되는 것은 우리의 모델과 관련이 없다. 이 깨달음이 가비지 콜렉터가 어떻게 동작하는지 더 잘 이해하는 데에 도움이 될 것이다.

## 콜렉터 동작

수집이 시작될 때, 콜렉터는 세 단계의 작업을 거친다. 이 중 두 단계는 Stop The World(STW) 지연 시간을 생성하고 다른 단계는 애플리케이션의 처리 속도를 늦추는 지연 시간을 생성한다. 세 가지 단계는 다음과 같다.

- 마크 설정 - STW
- 마킹 - 동시적
- 마크 종료 - STW

각 단계의 자세한 내용은 다음과 같다.

### 마크 설정 - STW

수집이 시작되면, 가장 먼저 수행되어야 하는 활동은 Write Barrier를 켜는 것이다. Write Barrier의 목적은 콜렉터와 애플리케이션 고루틴이 동시에 실행되기 때문에 수집 중에 콜렉터가 힙의 데이터 무결성을 유지할 수 있도록 하는 것이다.

Write Barrier를 켜려면 실행 중인 모든 애플리케이션 프로그램 고루틴을 중지해야 한다. 이 활동은 평균 10~30마이크로초 이내로 보통 매우 빠르다. 애플리케이션 고루틴이 제대로 동작하는 한 그렇다는 것이다. 

참고: 스케줄러 다이어그램을 좀 더 잘 알고 싶으면, 이 Go 스케줄러 시리즈 포스트를 꼭 읽어보라.

#### 그림 1

![](https://www.ardanlabs.com/images/goinggo/100_figure1.png)

그림 1은 수집이 시작되기 전에 실행되는 4개의 애플리케이션 고루틴을 나타낸다. 네 개의 고루틴은 각각 멈춰야 한다. 이를 수행하는 유일한 방법은 콜렉터가 각각의 고루틴이 함수 호출을 할 때까지 지켜보고 기다리는 것이다. 함수 호출은 고루틴들이 안전한 지점에서 멈추도록 보장한다. 만약 그 고루틴들 중 하나가 함수 호출을 하지 않고 다른 것들을 수행하면 어떻게 될까?

#### 그림 2

![](https://www.ardanlabs.com/images/goinggo/100_figure2.png)

그림 2는 진짜 문제를 나타낸다. P4에서 실행되는 고루틴이 중지될 때까지 수집을 시작할 수 없으며, 어떤 계산을 수행하는 타이트한 루프로 인해 발생할 수 없다.