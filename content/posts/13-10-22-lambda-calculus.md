+++
date        = "2013-10-22T13:03:05+09:00"
draft       = false
title       = "Lambda Calculus"
tags        = [ "Programming", "Calculus"]
categories  = [ "Programming", "Calculus"]
slug        = "lambda-calculus"
notoc       = true
+++

람다 대수는 결정문제(decision problem)를 풀기 위해, 계산가능성(computability)의 개념을 정의한 수학적 모델로 1936년 [알로존 처치](http://en.wikipedia.org/wiki/Alonzo_Church)(Alonzo Church)가 고안하였다. 

좀 더 구체적으로는 소프트웨어에서 '알고리즘이란 무엇인가?'를 정의한 클래스(class)나 객체(object)가 아닌 값(function) 중심 언어, 즉 함수형 언어의 계산 모델이다. (같은 해에 처치의 제자였던, [앨런 튜닝](http://en.wikipedia.org/wiki/Alan_Turing)은 튜링 머신의 개념으로 계산 모델을  정의했다. Lisp를 만든 [존 매카시](http://en.wikipedia.org/wiki/John_McCarthy_(computer_scientist\))도 처치의 제자였다고 한다.)

!['알론조 처치'](http://upload.wikimedia.org/wikipedia/en/thumb/a/a6/Alonzo_Church.jpg/220px-Alonzo_Church.jpg)

람다 대수는 그 자체가 이미 하나의 언어로 많은 함수형 프로그래밍 언어가 람다대수에 기초하고 있다.

>	functional language = lambda calcus + sugars (ex. Lisp, ML, Haskell, Scala.. ) [[1]]

즉, 우리가 함수를 정의하고 전달하고 반환하는 모든 것들이 람다 대수의 원리를 따르는 계산인 것이다.

이상 아래의 내용은 "A Tutorial Introduction to the Lambda Calculus[[2]]"의 내용 중 일부를 중심으로. 필요한 경우 살을 붙여가며, 간략히 요약한 것이다.

### 람다표현식 (expression)

람다 대수에서는 모든 계산 가능한 함수를 람다로 표현할 수 있는데, 람다 표현식(λ expression)은 다음과 같이 재귀적으로 정의된다.

    <expression>  := <name> | <function> | <application>
    <function>	  := λ<name>.<expression>
    <application> := <expression> <expression>


위에서 보다시피, 표현식(expression)은 사상점을 식별하는 변수나 혹은 함수 본문을 정의하고 있는 사상(abstraction)이거나 사상을 특화하는 적용(application)으로 구성된다.

람다 대수에서 키워드는 \\(λ\\) 와 \\(.\\)이다. 표현식을 분명하게 표기하기 위해 괄호를 사용한다. 예를 들어, 표현식 \\(E\\)는 \\((E)\\)와 동일하다. 혼란을 피하기 위해 함수의 적용(application)은 왼쪽으로 결합한다. 따라서, 표현식 $$E_1E_2E_3...E_n$$ 은 아래와 같이 표현식을 적용하며 평가된다. $$(...((E_1E_2)E_3)...E_n)$$

예를 들면, 함수 \\(f(x) = x\\)은 람다식으로 다음과 같다.

$$λx.x$$

위 표현식은 항등함수(identity function)를 정의한다. `λ` 표기 다음의 이름은 이 함수 인자의 식별자라 하고, `.` 표기 다음의 표현식을 함수 정의의 본문(body)이라고 한다.

함수는 표현식에 적용될 수 있다. 적용(application)의 예는 다음과 같다.

$$(λx.x)y$$

이는 `y`에 적용된 항등함수이다. 함수 적용은 함수를 정의하는 본문 안에서 인자의 값 `x`를 치환함으로써 (여기서는 `y`로) 평가된다. 예를 들면, 다음과 같다.

$$(λx.x)y = [y/x]x = y$$

위 변환에서 `[y/x]` 표기는 표현식에서 모든 경우의 `x`가 `y`에 의해 오른쪽으로 치환되는 것을 나타낸다. 그리고, 람다식에서 람다 식별자인 이름은 단순한 플레이스홀더에 지나지 않는다. 따라서, 다음은 동등하다.

$$(λz.z) ≡ (λy.y) ≡ (λt.t) ≡ (λu.u)$$


### 자유 변수와 종속 변수 (free variable and bound variable)

람다 대수에서 사상(abstraction)의 정의에 해당하는 모든 이름은 지역(local)이다. 함수 $λx.x$에서 `x`는 `x`로 시작하는 정의의 본문 안에 나타나므로 "종속된다"라고 말한다. `λ`로 시작하지 않는 이름을 자유 변수(free variable), 또는 비지역(non-local)이라고 일컫는다.

예를 들면, 아래의 표현식에서 $$(λx.xy)$$ 에서 변수 `x`는 종속되나 `y`는 자유롭다. 아래의 표현식에서, $$(λx.x)(λy.yx)$$ `x`는 왼쪽에서 첫째 표현식의 본문에서는 첫번째 `λ`에 종속된다. 두 번째 표현식의 본문에서 `y`는 두 번째 `y`에 종속되나 `x`는 자유롭다. 두 번째 표현식의 `x`가 첫 번째 표현식의 `x`에 대해서 완전히 독립적이라는 것은 매우 중요하다.

표현식에서 변수 `<name>`이 다음의 3가지 경우에 해당할 때, 자유 변수라고 한다.

* `<name>`은 `<name>`의 자유 변수이다.
* `<name>`은 `λ<name1>`의 자유 변수이다. (단, 식별자 `<name>`이 `<name1>`과 다른 `<exp>`이고, `<name>`이 `<exp>`의 자유 변수일 때)
* `<name>`은 `E1E2`의 자유 변수이다. (단, `E1`의 자유 변수이거나 혹은 `E2`의 자유 변수일때)

그리고 다음의 두 가지 경우에 한해서 `<name>` 변수를 종속 변수라고 한다.

* `<name>`은 `λ<name1>`의 종속 변수이다. (단, 식별자 `<name>`이 `<name1>`과 같고 `<exp>`이고, `<name>`이 `<exp>`의 종속 변수일 때)
* `<name>`은 `E1E2`의 종속 변수이다. (단, `E1`의 종속 변수이거나 혹은 `E2`의 종속 변수일때)

이를 수식화하면, 람다 표현식 `E`의 자유 변수 집합 `M`은 `FV(M)`으로 다음과 같다.

$$FV(x) = {x}$$
$$FV (λx.E) = FV (E)\space/\space{x}$$
$$FV(E1 E2) = FV(E1)∪FV(E2)$$


한 표현식에서 동일한 식별자가 자유 변수와 종속 변수가 될 수 있음을 주의해야 한다. 아래의 표현식에서,


$$ (λx.xy)(λy.y) $$


첫 번째 `y`는 왼쪽 방향의 괄호로 묶인 하위 표현식에서는 자유 변수이다. 그러나 오른쪽 방향의 하위 표현식에서는 종속 변수이다.

자유 변수가 없는 표현식을 닫혀 있다(closed)라고 한다. 닫힌 람다 표현식(Closed lambda expression)을 컴비네이터(combinator)라고도 한다.


### 치환 (Substitutions)

함수를 적용할 때마다, 전체 함수 정의를 쓰고 다음에 이를 평가한다. 그러나 단순하게 표기하기 위해 대문자, 숫자 그리고 어떤 함수 정의와 동치로 기호를 사용한다. 예를 들면, 항등함수는 {% m %} (λx.x) {% em %}의 동치로 {% m %} I {% em %} 기호로 표기한다.

자신에 적용된 항등 함수의 적용은 다음과 같다.


$$ II ≡ (λx.x)(λx.x) $$


이 표현식에서 괄호로 묶인 첫 번째 표현식의 본문에 있는 첫 번째 `x`는 두 번째 표현식 본문의 `x`와 별개의 것이다. 사실, 우리는 위 표현식을 아래와 같이 쓸 수 있다.


$$ II ≡ (λx.x)(λz.z) $$


그러므로, 자신에 적용된 항등 함수


$$ II ≡ (λx.x)(λz.z) $$


는 다음의 결과를 따르게 된다.


$$ [λz.z/x]x = λz.z ≡ I $$


즉, 또 항등함수이다.

치환을 수행할 때는 식별자의 자유 변수 사건이 종속 변수 사건과 뒤섞이지 않도록 주의해야 한다. 다음 표현식에서


$$ (λx.(λy.xy))y $$


(괄호 밖의) 오른쪽 `y`는 자유 변수인 반면, 왼쪽에 있는 함수의 `y`는 종속 변수이다. 부정확한 치환으로 두 식별자를 합치면 잘못된 결과를 얻는다.


$$ (λy.yy) $$


단순히 종속 변수 `y`를 `t`로 변경하면, 정확한 치환에 의해 완전히 다른 결과를 얻는다.


$$ (λx.(λt.xt))y = (λt.yt) $$


`E`를 함수 `λx.<exp>`에 적용할 경우, `<exp>`의 모든 자유 변수 `x`는 `E`로 치환한다. 표현식에서 종속변수가 `E`의 자유변수로 치환될 경우, 치환하기 전에 종속 변수를 새이름으로 변경한다. 예를 들면, 아래의 표현식에서


$$ (λx.(λy.(x(λx.xy))))y $$


인자 `x`는 `y`로 결합시키는데, 함수 본문에서는 첫 번째 `x`만이 자유 변수이고 치환될 수 있다. 그래도 치환을 하기 전에 변수 `y`의 종속 변수가 자유 변수의 경우와 섞이지 않도록 이름을 변경하자.


$$ [y/x]λt.(x(λx.xt))) = (λt.(y(λx.xt))) $$


### call by value, call by name, call by need

표현식을 평가할 때, 안쪽에서 바깥쪽으로 평가하는 전략을 **Application order** 라고 하며, **call by value** 이라고 알려져 있다. 예를 들면, 다음과 같다.


$$ (λx.x^2(λx.(x+2)\space2))) → (λx.x^2(2+2)) → (λx.x^2(4)) → 4^2 → 16 $$


반면에, 바깥쪽에서 안쪽으로 평가하는 전략을 **Normal order** 라고 하며 이는 **call by name** 이라고 알려져 있다. 예를 들면, 다음과 같다.


$$ (λx.x^2(λx.(x+2)\space2)) → (λx.(x+2)\space2)^2 → (2+2)^2 → 4^2 → 16 $$


위에서 보다시피, Normal order는 인자의 평가를 필요할 때까지 지연시키기 때문에 Normal order를 **지연 평가(lazy evaluation)** 라고 부른다. (※. SICP 참고[[3]])

위키 정의를 따르면, ** call by need ** 은 아래와 같이 call by name의 최적화된 형태이다. 매번 call by name이 발생할 경우, 함수 본문은 동일하기 때문에 매번 평가할 필요가 없다. 따라서, call by need 에서는 중복적인 연산을 피하기 위해 cache 형태로 값을 저장하여 최초에 한번만 평가한다.

> Call-by-need is a memoized version of call-by-name where, if the function argument is evaluated, that value is stored for subsequent uses [[4]]

#### Lazy evaluation

어떨 때는 *lazy*를 call by name과 call by need를 모두 지칭하지만, 어떨 때는 call by need만을 지칭하기도 한다. 특히, Haskell에서 Lazy evaluation은 기술적으로 non-strict와 sharing, 즉 call by need를 뜻한다.[[5]]

이러한 충돌은 논문에서도 살펴볼 수 있다. 어떤 이는 call by name을 `lazy`라고, 어떤 이는 call by need를 `lazy`라고 지칭한다.

> 예를 들면, Abramsky의 논문 'The Lazy Lambda Calculus (1990)'에서는 'call by name'을 'lazy'라고 칭하고 있다. 반면에, Odersky 교수가 참여한 논문 'A Call-By-Need Lambda Calculus (1995)'에서는 Abramsky의 논문에서의 'lazy'라는 표현과 충돌을 피하기 위해 'call by need'라고 표현하고 있다. [[6]]

Odersky 논문에서 보듯, 람다 대수를 이용한 프로그래밍 언어에 있어 'call by need'는 컴파일러 성능과 직결된 중요한 기능이기 때문에 lazy evaluation을 sharing, caching, memoziation 등의 최적화 기법과 연관짓게 된다. 그러나, 일반적으로 표현식이 그 값이 필요할 때까지 평가되지 않는 것을 'lazy evaluation' 이라고 말하므로 call by name과 call by need 모두 이에 해당한다.

중요한 것은 call by value, call by name, call by need의 차이를 아는 것이다.

### 산술 (Arithmetic)
우리는 프로그래밍 언어에서 산술적인 계산을 할 수 있어야 한다고 기대한다. 람다 대수에서 숫자는 `zero`에서 시작해서 `suc(zero)`라 쓰고 1을, `suc(suc(zeo))`라 쓰고 2를, 그리고 기타등등 이런 식으로 표현한다.

`zero`는 다음과 같이 정의될 수 있다.


$$ λs.(λz.z) $$


이는 `s`와 `z` 두 개의 인자를 지닌 함수이다. 하나 이상의 인자를 지닌 표현식을 다음과 같이 축약한다.


$$ λsz.z $$


즉, `s`는 평가시 치환되는 첫 번째 인자이고, `z`는 두 번째 인자이다. 이 표기를 사용해 자연수는 다음과 같이 정의된다.

$$ 1 ≡ λsz.s(z) $$
$$ 2 ≡ λsz.s(s(z)) $$
$$ 3 ≡ λsz.s(s(s(z))) $$

따라서, 처치의 자연수는 `n`에 대응되는 함수를 `n`회 반복해서 실행한다.


$$ n ≡ λsz.s^n(z) $$


흥미로운 것은 계승자 함수이다. `y`의 추가적인 적용으로 자연수 `y`를 입력받아 `y + 1`을 반환하는 계승자 함수(successor)를 다음과 같이 정의할 수 있다.


$$ S ≡ λwyx.y(wyx) $$


계승자 함수에 `zero`를 적용하면, 다음을 도출한다.


$$ S0 ≡ (λwyx.y(wyx))(λsz.z) $$


첫 번째 표현식 본문에 `w`를 `(λsz.z)`로 치환하면 자연수 1에 대응된다.


$$ λyx.y((λsz.z)yx) = λyx.y((λz.z)x) = λyx.y(x) ≡ 1 $$


계승자 함수에 `1`를 적용하면, 자연수 2에 대응된다.


$$ S1 ≡ (λwyx.y(wyx))(λsz.s(z)) = λyx.y((λsz.s(z))yx) = λyx.y(y(x)) ≡ 2 $$


(이하 중략)

이상, 람다 대수로 자연수와 그 연산을 표현하는 것을 처치 부호화(Church encoding)라고 한다.[[7]]

### 결론

이상, 람다계산법에 대해 간략히 살펴보았다. 짧게나마 살펴보니, 람다계산법을 공부할수록 쉽게 함수형 프로그래밍의 원리를 이해할 수 있을 것 같다. 

[1]: http://ropas.snu.ac.kr/~dreameye/PL/slide/PL7.pdf "dreameye님의 람다계산법"

[2]: http://www.inf.fu-berlin.de/lehre/WS03/alpi/lambda.pdf "A Tutorial Introduction to the Lambda Calculus"

[3]: http://mitpress.mit.edu/sicp/full-text/sicp/book/node85.html

[4]: http://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_need

[5]: http://www.haskell.org/haskellwiki/Lazy_evaluation

[6]: http://lampwww.epfl.ch/~odersky/papers/popl95.ps.gz "A Call-By-Need Lambda Calculus"

[7]: http://en.wikipedia.org/wiki/Church_encoding "Church encoding"

