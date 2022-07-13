---
title: "[The Fuzzing Book - Introduction to Software Testing"
comments: true
categories:
- tech
tags:
- tech
- python
- testing
- fuzzing
last_modified_at: 2022-07-05T16:48+09:00
toc: true
---

책의 주요 내용에 들어가기 앞서 Software Testing의 필수 개념에 대해서 알아보자.

# 소프트웨어 테스팅 소개
## Simple Testing

해당 소스코드는 Newton-Rapson Method 를 사용하여 제곱근을 구하는 함수의 파이썬 코드이다.

```python
def my_sqrt(x):
    """Computes the square root of x, using the Newton-Raphson method"""
    approx = None
    guess = x / 2
    while approx != guess:
        approx = guess
        guess = (approx + x / approx) / 2
    return approx
```

위 함수가 실제로 제곱근을 구할 수 있는지 테스트해보자

## 파이썬 프로그램의 이해

위 파이썬 코드를 이해하기 위한 가장 중요한 세가지는 아래와 같다.

1. 파이썬은 들여쓰기를 통해 프로그램을 구성하기 때문에 함수와 **while** 문의 몸체는 들여쓰기로 정의된다.
2. 파이썬은 동적 타입 언어이기 때문에 **x**, **approx**, **guess**와 같은 변수의 타입은 런타임 시 결정된다.
3. 할당 연산자(=)나 비교 연산자(==, !=, <), 제어문같은(while, if) 대부분의 파이썬 구문적인 특징은 다른 흔한 프로그래밍 언어에서 파생되었다.

위 세가지 내용을 통해 위 함수의 동작을 이해할 수 있다.

## 함수 실행

함수가 제대로 실행되는지 확인해보자. 몇가지 값을 사용하여 **my_sqrt** 함수가 제대로 실행되는지 확인할 수 있다.

예를 들어 매개변수 **x**에 4와 2를 넣었을 때 다음과 같이 올바른 값이 나오는 것을 확인할 수 있다.

```python
>>> my_sqrt(4)
2.0
>>> my_sqrt(2)
1.414213562373095
```

## 함수 디버깅

**my_sqrt**함수가 동작하는 방식을 볼 수 있는 간단한 방법은 함수의 중요(critical) 포인트에 **print()** 문을 넣는것이다.
예를 들어 아래와 같이 **approx**변수의 값을 출력함으로써 각각의 루프에서 값이 어떻게 가까워지는지 확인할 수 있다.

```python
>>> def my_sqrt_with_log(x):
    """Computes the square root of x, using the Newton–Raphson method"""
        approx = None
        guess = x / 2
        while approx != guess:
            print("approx =", approx)  # <-- New
            approx = guess
            guess = (approx + x / approx) / 2
        return approx

>>> my_sqrt_with_log(9)
approx = None
approx = 4.5
approx = 3.25
approx = 3.0096153846153846
approx = 3.000015360039322
approx = 3.0000000000393214
3.0
```

## 함수 검사

다시 테스팅으로 돌아와보자. 과연 위에서 실행한 **my_sqrt(2)** 의 결과값이 실제로 정확하다고 할 수 있을까? 우리는 **√x x √x = x** 식을 사용하여 위 의문을 쉽게 증명할 수 있다.

```python
>>> my_sqrt(2) * my_sqrt(2)
1.9999999999999996
```
반올림 오류가 발생하긴 했지만 값에 근접해보인다.

지금부터 우리는 프로그램을 실행시킨 후 입력값을 입력하고, 결과값이 올바른지 검사하면서 위의 프로그램을 테스트할 것이다. 이러한 테스트는 프로그램이 제품으로 생산되기 전에 최소한의 품질 보증이다.

# 테스트 실행 자동화

지금까지의 테스트를 봤을 때는 실행부터 결과 확인까지 일일이 하나하나 진행했다. 이런 방법은 사람이 직접하는만큼 유연한 방법이지만 길게봤을 때는 비효율적인 방법이다.

1. 일일이 검사시, 한정된 숫자에 대해서만 실행과 결과를 검사할 수 있다. 
2. 프로그램에 수정이 발생했을 때는 해당 과정을 다시 일일이 진행해야 된다.

위 두가지 이유로 인해 테스트 자동화가 매우 유용하다. 테스트를 자동화하는 가장 간단한 방법은 일단 컴퓨터로 테스트의 결과를 미리 계산하는 것이다. 그 후 해당 결과를 프로그램의 결과와 비교하면 된다.

예를 들어 아래 코드는 √4 = 2 식이 맞는지 자동으로 테스트하는 코드이다.

```python
>>> result = my_sqrt(4)
>>> expected_result = 2.0
>>> if result == expected_result:
        print("Test passed")
    else:
        print("Test failed")

Test passed
```

위 테스트의 좋은 점은 테스트를 계속 반복할 수 있고, 적어도 4의 제곱근을 정확하게 계산할 수 있다는 것을 보장할 수 있다. 하지만 위 방법도 아직 아래와 같은 이슈가 있다. 

1. 한번의 테스트에 대해 5개 줄의 코드가 필요하다.
2. 위 코드는 반올림 오류를 고려하고 있지 않다.
3. 한개의 입력과 출력에 대해서만 테스트를 진행한다.

위 문제들을 하나씩 해결해보자. 일단 테스트를 좀더 완성도있게 만들자. 거의 모든 프로그래밍 언어에는 조건이 유지되는지 자동으로 확인하고 그렇지 않으면 실행을 중지할 수 있는 수단이 있다. 이것을 **assertion** 이라고 부르며, 테스팅에 굉장히 유용하다.

파이썬에서는 **assert** 문이 조건을 가지고 있고, 만약 조건이 참이라면 아무일도 일어나지 않는다. 만약 조건 평가가 실패라면, **assert** 문에서 실패 시점을 가리키면서 에러를 발생시킨다.

아래 코드는 우리는 **assert**를 사용하여 **my_sqrt()**가 위와 같이 예상 결과를 산출하는지 쉽게 확인할 수 있다. 

아래 코드를 실행시키면 아무일도 일어나지 않는것을 볼 수 있다.

```python
>>> assert my_sqrt(4) == 2
```

부도소수점 계산에서는 반올림 오류가 발생할 수 있다. 그래서 간단하게 두개의 부동 소수점 값의 같음을 비교할 수 없다. 오히려, 우리는 두 숫자 사이의 절대적인 차이가 일반적으로 **ε**나 **앱실론**로 표시된 특정 임계값 이하로 유지되도록 할 것이다. 아래가 그 방법이다.

```python
>>> EPSILON = 1e-8
>>> assert abs(my_sqrt(4) - 2) < EPSILON
>>>
```

우리는 또한 이 목적을 위한 특수 함수를 도입할 수 있으며, 이제 구체적인 값에 대한 더 많은 테스트를 수행할 수 있다.

```python
>>> def assertEquals(x, y, epsilon=1e-8):
        assert abs(x - y) < epsilon

>>> assertEquals(my_sqrt(4), 2)
>>> assertEquals(my_sqrt(9), 3)
>>> assertEquals(my_sqrt(100), 10)
>>>
```

위 함수는 잘 작동되는 것처럼 보인다. 만약 특정 계산의 결과값을 알고 있으면, 프로그램이 정확하게 동작하는지 보장하기 위해 위와같은 **assertion**을 반복할 수 있다. 
(파이썬 프로그래머라면 math.isclose() 함수를 대신 사용할 수 있다.)

# 테스트 생성

√x × √x = x 공식을 보편적으로 유지되므로 몇가지 값을 사용하여 명시적인 테스트를 할 수 있다.

```python
>>> assertEquals(my_sqrt(2) * my_sqrt(2), 2)
>>> assertEquals(my_sqrt(3) * my_sqrt(3), 3)
>>> assertEquals(my_sqrt(42.11) * my_sqrt(42.11), 42.11)
>>>
```

수천개의 값에 대해서 매우 쉽게 테스트할 수도 있다.

```python
>>> for n in range(1, 1000):
        assertEquals(my_sqrt(n) * my_sqrt(n), n)
```

100가지 값에 대해서 **my_sqrt()** 함수를 테스트하기 위해서는 시간이 얼마나 필요할까? 한번 알아보자.

경과한 시간을 알아보기 위해 책에서 사용하는 자체 **Timer** 모듈을 사용한다.

```python
>>> import bookutils
>>> from Timer import Timer
>>> with Timer() as t:
        for n in range(1, 10000):
            assertEquals(my_sqrt(n) * my_sqrt(n), n)
        print(t.elapsed_time())

0.017505749999999987
```

10,000개의 값은 약 100분의 1초가 걸리므로, my_sqrt()의 단일 실행은 1/1000000초 또는 약 1마이크로초가 걸린다.

무작위로 선택된 10,000개의 값으로 이것을 반복하자. 파이썬 random.random() 함수는 0.0에서 1.0 사이의 무작위 값을 반환한다.

```python
>>> import random
>>> with Timer() as t:
        for i in range(10000):
            x = 1 + random.random() * 1000000
            assertEquals(my_sqrt(x) * my_sqrt(x), x)
        print(t.elapsed_time())

0.020891541999999985
```

1초 이내에, 10,000개의 무작위 값을 테스트했고, 매번 제곱근은 실제로 올바르게 계산되었다. 이제 **my_sqrt()** 함수를 변경할 떄마다 함수가 제대로 작동한다는 보장을 가지고 테스트를 반복할 수 있다. 하지만 무작위 함수는 프로그램 동작을 크게 변경하는 특수 값을 생성할 것 같지는 않다. 우리는 이것을 나중에 아래에서 논의할 것이다.

# 런타임 확인

**my_sqrt** 함수에 대한 테스트를 작성하고 실행하는 대신, 검사를 구현에 바로 통합할 수도 있다.

아래와 같은 *자동 런타임 검사*는 구현하기 쉽다.

```python
>>> def my_sqrt_checked(x):
        root = my_sqrt(x)
        assertEquals(root * root, x)
        return root

>>> my_sqrt_checked(2.0)
1.414213562373095
```

이제 **my_sqrt_checked**함수를 사용하여 제곱근을 계산할 때마다, 이미 결과가 정확하다는 것을 알고 있으며, 모든 새로운 성공적인 계산에 대해 그렇게 할 것이다.

**자동화된 런타임 검사**는 아라와 같은 두가지를 가정하고 있다.

* 런타임 검사를 공식화할 수 있어야 한다. 확인할 구체적인 값을 갖는 것은 항상 가능해야 하지만, 추상적인 방식으로 원하는 속성을 공식화하는 것은 매우 복잡할 수 있다. 실제로, 어떤 속성이 가장 중요한지 결정하고, 적절한 검사를 설계해야 합니다. 게다가, 런타임 검사는 로컬 속성뿐만 아니라 모두 식별되어야 하는 프로그램 상태의 여러 속성에 따라 달라질 수 있습니다.
* 그러한 런타임 수표를 감당할 수 있어야 한다. **my_sqrt** 함수의 경우, 검사는 그리 비싸지 않지만, 간단한 작업 후에도 대용량 데이터 구조를 확인해야 한다면, 검사의 비용은 금지될 수 있습니다. 실제로, 런타임 검사는 일반적으로 생산 중에 비활성화되며, 효율성을 위해 신뢰성을 거래한다. 반면에, 포괄적인 런타임 검사 제품군은 오류를 찾고 빠르게 디버깅할 수 있는 좋은 방법입니다. 생산 중에 여전히 얼마나 많은 기능을 원하는지 결정해야 합니다.

런타임 검사의 중요한 한계는 확인해야 할 결과가 있는 경우에만 정확성을 보장한다는 것이다. 즉, 결과가 항상 있을 것이라고 보장하지는 않습니다. 이것은 상징적인 검증 기술과 프로그램 증명에 비해 중요한 한계이며, 훨씬 더 높은 (종종 수동적인) 노력으로 결과가 있다는 것을 보장할 수 있다.

# System 입력 vs 함수 입력

이번에는 **my_sqrt()** 함수를 다른 프로그래머들이 다른 코드애 심을 수 있도록 만들어볼 것이다. 어떤 시점에서 함수는 제 3자에서 오는 입력을 처리해야 하고, 해당 입력은 프로그래머에 의해 통제될 수 없다.

```python
>>> def sqrt_program(arg: str) -> None:
        x = int(arg)
        print('The root of', x, 'is', my_sqrt(x))

>>> sqrt_program(4)
The root of 4 is 2.0
```

우리는 위와같이 시스템 입력으로 쉽게 **sqrt_program** 함수를 호출할 수 있다.

하지만 문제가 있다. 프로그램은 정당한 외부 입력에 대해 검사하고 있지 않다. 예를 들어 **sqrt_program(-1)** 을 호출하면 무슨일이 있어날까?

만약 음수를 사용해서 **my_sqrt** 함수를 호출하면 함수는 무한 루프에 빠지게 된다. 기술적인 이유로, 우리는 이 장에서 무한 루프를 가질 수 없습니다 (코드가 영원히 실행되기를 원하지 않는 한) 그래서 우리는 특별한 **with ExpectTimeOut(1)** 구문을 사용하여 1초 후에 실행을 중단한다.

```python
>>> from ExpectError import ExpectTimeout
>>> with ExpectTimeout(1):
        sqrt_program("-1")

Traceback (most recent call last):
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/1288144681.py", line 2, in <module>
    sqrt_program("-1")
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/449782637.py", line 3, in sqrt_program
    print('The root of', x, 'is', my_sqrt(x))
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/2661069967.py", line 7, in my_sqrt
    guess = (approx + x / approx) / 2
  File "/Users/zeller/Projects/fuzzingbook/notebooks/Timeout.ipynb", line 43, in timeout_handler
    raise TimeoutError()
TimeoutError (expected)
```

위 내용은 무언가 잘못됐다는 에러 메시지이다. 해당 메시지는 함수의 콜 스택과 에러 발생 시점에 실행된 줄을 나열하고 있다. 

코드가 에러로 인해 중단되는 것을 원치 않기 때문에, 외부 입력을 받았을 때 해당 값이 제대로 검증되었는지 확인해야 한다. 

```python
>>> def sqrt_program(arg: str) -> None:
        x = int(arg)
        if x < 0:
            print("Illegal Input")
        else:
            print('The root of', x, 'is', my_sqrt(x))

>>> sqrt_program(-1)
Illegal Input
```

위 코드에서 **my_sqrt()** 함수가 입력값에 따라 호출되는지 확인할 수 있다.

하지만 만약 함수가 숫자가 아닌 값을 입력받았을 때는 어떤 값이 출력될까?

숫자가 아닌 문자열을 변환하려고 하면 런타임 오류가 발생한다.

```python
>>> from ExpectError import ExpectError
>>> with ExpectError():
        sqrt_program("xyzzy")

Traceback (most recent call last):
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/1336991207.py", line 2, in <module>
    sqrt_program("xyzzy")
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/3211514011.py", line 2, in sqrt_program
    x = int(arg)
ValueError: invalid literal for int() with base 10: 'xyzzy' (expected)
>>>
```

아래 코드가 문자열 입력을 고려한 수정된 버전이다.

```python
>>> def sqrt_program(arg: str) -> None:
      try:
          x = float(arg)
      except ValueError:
          print("Illegal Input")
      else:
          if x < 0:
            print("Illegal Number")
          else:
              print('The root of', x, 'is', my_sqrt(x))

>>> sqrt_program("4")
The root of 4.0 is 2.0
>>> sqrt_program("-1")
Illegal Number
>>> sqrt_program("xyzzy")
Illegal Input
>>>
```

우리는 이제 시스템 수준에서 프로그램이 통제되지 않은 상태로 들어가지 않고도 모든 종류의 입력을 우아하게 처리할 수 있어야 한다는 것을 보았다. 물론, 이것은 모든 상황에서 프로그램을 강력하게 만들기 위해 고군분투해야 하는 프로그래머들에게 부담이다. 그러나 이러한 부담은 소프트웨어 테스트를 생성할 때 이점이 된다. 프로그램이 모든 종류의 입력을 처리할 수 있다면(아마도 잘 정의된 오류 메시지로), 모든 종류의 입력도 보낼 수 있다. 하지만 생성된 값을 가진 함수를 호출할 때, 우리는 정확한 전제 조건을 알아야 한다.

# 테스팅의 한계

테스트에 최선의 노력에도 불구하고, 항상 제한된 입력 세트에 대한 기능을 확인하고 있다는 것을 알아야한다. 따라서 함수가 여전히 실패할 수 있는, 테스트되지 않은 입력이 항상 있을 수 있다.

my_sqrt()의 경우, 0으로 나눈 결과인 계산된 √0을 예로 들어 보자.

```python
>>> with ExpectError():
      root = my_sqrt(0)

Traceback (most recent call last):
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/820411145.py", line 2, in <module>
    root = my_sqrt(0)
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/2661069967.py", line 7, in my_sqrt
    guess = (approx + x / approx) / 2
ZeroDivisionError: float division by zero (expected)
```

지금까지의 테스트에서 우리는 해당 조건을 확인하지 않았지만 √0 = 0을 작성한 프로그램은 놀랍게도 실패한다. 하지만 우리가 1-1000000이 아닌 0-1000000 범위의 입력을 생성하기 위해 무작위 발전기를 설정했더라도, 우연히 제로 값을 생성할 가능성은 여전히 백만분의 1이었을 것이다. 소수의 개별 값에 대해 함수의 동작이 근본적으로 다르다면, 일반 무작위 테스트는 이것들을 생성할 기회가 거의 없다.

물론, 그에 따라 함수를 수정하여 x에 대해 허용된 값을 문서화하고 특수 케이스인 **x = 0** 을 처리할 수 있다.

```python
>>> def my_sqrt_fixed(x):
      assert 0 <= x
      if x == 0:
          return 0
      return my_sqrt(x)

>>> assert my_sqrt_fixed(0) == 0
>>>
```

이것으로, 우리는 이제 **√0 = 0**을 올바르게 계산할 수 있다.

그 외 허용되지지 않은 값은 이제 예외로 이어진다.

```python
>>> with ExpectError():
      root = my_sqrt_fixed(-1)
Traceback (most recent call last):
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/305965227.py", line 2, in <module>
    root = my_sqrt_fixed(-1)
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_11687/3001478627.py", line 2, in my_sqrt_fixed
    assert 0 <= x
AssertionError (expected)
>>>
```

여전히, 우리는 광범위한 테스트가 프로그램의 정확성에 대한 높은 자신감을 줄 수 있지만, 미래의 모든 실행이 정확할 것이라는 보장은 아니라는 것을 기억해야 한다. 모든 결과를 확인하는 런타임 검증조차도 결과를 생성하면 결과가 정확할 것이라고 보장할 수 있지만, 향후 실행이 실패한 검사로 이어지지 않을 수 있다는 보장은 없습니다. 이 글을 쓰면서, 나는 my_sqrt_fixed(x)가 모든 유한 숫자 x에 대해 √x의 올바른 구현이라고 믿지만 확신할 수는 없다. 

Newton-Raphson 방법을 사용하면 구현이 정확하다는 것을 실제로 증명할 수 있는 좋은 기회가 있을 수 있습니다: 구현은 간단하고 수학은 잘 이해됩니다. 아아, 이것은 소수의 도메인에서만 마찬가지입니다.
