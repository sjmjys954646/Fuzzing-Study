# Fuzzing: Breaking Things with Random Inputs

#

## 목차

* Synopsis
* A Testing Assignment
* A Simple Fuzzer
* Fuzzing External Programs
	- Creating Input Files
	- Invoking External Programs
	- Long-Running Fuzzing
* Bugs Fuzzers Find
	- Buffer Overflows
	- Missing Error Checks
	- Rogue Numbers
* Catching Errors
	- Generic Checkers
	- Program-Specific Checkers
	- Static Code Checkers
* A Fuzzing Architecture
	- Runner Classes
	- Fuzzer Classes
* Lessons Learned
* Exercises
	- Exercise 1: Simulate Troff
	- Exercise 2: Run Simulated Troff
	- Exercise 3: Run Real Troff

#
---
#

## Synopsis

이 챕터에서 제공하는 코드를 사용하기 위해서 추가하세요.

```python
>>> from fuzzingbook.Fuzzer import <identifier>
```

이 챕터는 A Fuzzing Architecture에서 소개된 두가지 중요한 클래스를 제공합니다.

* **fuzzer**의 기본 클래인 Fuzzer
* **programs**의 기본 클래스인 Runner

#

## Fuzzers

**Fuzzer**는 fuzzers의 기본 클래스이며, 간단화 인스턴스로 **RandomFuzzer**를 사용합니다.
**Fuzzer** 객체의 **fuzz()** 메서드는 생성된 입력이 포함된 문자열을 반환합니다.

```python
>>> random_fuzzer = RandomFuzzer()
>>> random_fuzzer.fuzz()
'%$<1&<%+=!"83?+)9:++9138 42/ "7;0-,)06 "1(2;6>?99$%7!!*#96=>2&-/(5*)=$;0$$+;<12"?30&'
```

**RandomFuzzer()**는 생성자를 사용하면 다음과 같은 키워드 인수를 지정할 수 있습니다.

```python
>>> print(RandomFuzzer.__init__.__doc__)
Produce strings of `min_length` to `max_length` characters
           in the range [`char_start`, `char_start` + `char_range`)

>>> random_fuzzer = RandomFuzzer(min_length=10, max_length=20, char_start=65, char_range=26)
>>> random_fuzzer.fuzz()
'XGZVDDPZOOW'
```

![diagram](https://velog.velcdn.com/images/silvergun8291/post/a5c4d2d3-5e5d-429c-907b-afd62601b289/image.JPG)


## Runners

퍼저는 PASS, FAIL, UNRESOLVED로 Runner와 페어링 될 수 있습니다. **Print Runner**는 주어진 입력을 출력하고 PASS 결과를 반환합니다.

```python
>>> print_runner = PrintRunner()
>>> random_fuzzer.run(print_runner)
EQYGAXPTVPJGTYHXFJ

('EQYGAXPTVPJGTYHXFJ', 'UNRESOLVED')
```

**ProgramRunner**는 생성된 입력을 외부 프로그램에 공급합니다. 결과는 프로그램의 상태(완료된 프로레스 인스턴스)와 결과(PASS, FAIL, UNRESOLVED)의 쌍입니다.

```python
>>> cat = ProgramRunner('cat')
>>> random_fuzzer.run(cat)
(CompletedProcess(args='cat', returncode=0, stdout='BZOQTXFBTEOVYX', stderr=''),
 'PASS')
 ```

![diagram2](https://velog.velcdn.com/images/silvergun8291/post/480dc1c6-2f25-4dda-a6e9-4cc7e2680a88/image.JPG)

#

## A Testing Assignment

퍼징은 "1988년 가을의 어둡고 폭풍우가 몰아치는 밤"에 발명되었습니다. [Takanen et al, 2008]. 매디슨 위스콘신에 있는 아파트에 있었던, 바튼 밀러 교수는 1200보의 전화선을 통해 그의 대학 컴퓨터에 접속했습니다. 천둥번개는 회선에 노이즈를 발생시켰고, 이 노이즈는 UNIX 명령어들이 잘못된 입력을 받아 충돌하는 원인이 되었습니다. 잦은 충돌로 인해 그는 놀랐고, 프로그램이 저 노이즈보다 더 견고해야 한다고 생각했습니다. 과학자로서, 그는 문제의 범위와 원인을 조사하기를 원했습니다.그래서 위스콘신-매디슨 대학에서 그의 학생들에게 프로그래밍 과제를 냈습니다. 그 과제는 학생들이 최초의 퍼저를 만들게 하는 것이었습니다.

과제는 다음과 같습니다.

> 이 프로젝트의 목표는 예측할 수 없는 입력 스트림이 주어졌을 때 다양한 UNIX 유틸리티 프로그램의 견고성을 평가하는 것입니다. [...] 첫 번째로, fuzz generator를 만드세요. 이것은 랜덤 문자 스트림을 출력하는 프로그램입니다. 둘째, fuzz generator를 사용하여 가능한 한 많은 UNIX 유틸리티를 공격하세요.

이 과제는 퍼징의 본질을 다룹니다: 무작위 입력을 만들고 그 입력이 프로그램에 오류를 일으키는지 확인하세요.

#

## A Simple Fuzzer

이 과제를 완수하고 fuzz generator를 만들어 봅시다. 아이디어는 임의의 문장을 생성하여 버퍼 문자열 변수에 추가한 후 문자열을 반환하는 것입니다.

이 구현에서는 다음과 같은 파이썬 기능을 사용합니다.

* **random.randrange(start, end)** – return a random number  [  **start, end**  )
* **range(start, end)** – create a list with integers in the range  [  **start, end**  ) . Typically used in iterations.
* **for elem in list: body** – execute **body** in a loop with **elem** taking each value from **list**.
* **for i in range(start, end): body** – execute **body** in a loop with **i** from **start** to **end**  −  1.
* **chr(n)** – return a character with ASCII code n

우리는 랜덤한 숫자를 사용하기 위해서, 각각의 모듈을 import 해야합니다.

```python
import random
```

여기에 실제 **fuzzer()** 함수가 있습니다.

```python
def fuzzer(max_length: int = 100, char_start: int = 32, char_range: int = 32) -> str:
    """A string of up to `max_length` characters
       in the range [`char_start`, `char_start` + `char_range`)"""
    string_length = random.randrange(0, max_length + 1)
    out = ""
    for i in range(0, string_length):
        out += chr(random.randrange(char_start, char_start + char_range))
    return out
```

**fuzzer()** 함수는 기본 인수를 사용하여 다음과 같은 임의의 문자열을 반환합니다.

```python
print(fuzzer())

>>> !7#%"*#0=)$;%6*;>638:*>80"=</>(/*:-(2<4 !:5*6856&?""11<7+%<%7,4.8,*+&,,$,."
```

바트 밀러는 이러한 무작위 비정형 데이터의 이름으로 "fuzz"라는 용어를 만들었습니다. 이제 이 "fuzz" 문자열이 특정 입력 형식(예를 들어 쉼표로 구분된 값 목록 또는 이메일 주소)의 데이터가 들어올 것이라고 예상하는 프로그램의 입력이라고 상상해 보십시오. 프로그램이 이러한 입력을 문제없이 처리할 수 있을까요?


Fuzzing은 쉽게 다른 종류의 입력값도 만들도록 설정할 수 있습니다. 예를 들어 우리는 fuzzer()가 일련의 소문자를 생성하도록 할 수 있습니다.

```python
# max length: 1000, start: 'a', range: 26 ('a' ~ 'z')
fuzz = fuzzer(1000, ord('a'), 26)
print(fuzz)
```

```python
alzbcutzsbsxwuftbioecunclfnpvljmnfpmivwboohvtpezhqfngeqjrzovkywaankfvxultmtzrkldpbybiixrzolfyzcrxoclvgkhmgfxfdggsvqcygqhbzzskscocrxllosagkvaszlngpysurezehvcqcghygphnhonehczraznkibltfmocxddoxcmrvatcleysksodzlwmzdndoxrjfqigjhqjxkblyrtoaydlwwisrvxtxsejhfbnforvlfisojqaktcxpmjqsfsycisoexjctydzxzzutukdztxvdpqbjuqmsectwjvylvbixzfmqiabdnihqagsvlyxwxxconminadcaqjdzcnzfjlwccyudmdfceiepwvyggepjxoeqaqbjzvmjdlebxqvehkmlevoofjlilegieeihmetjappbisqgrjhglzgffqrdqcwfmmwqecxlqfpvgtvcddvmwkplmwadgiyckrfjddxnegvmxravaunzwhpfpyzuyyavwwtgykwfszasvlbwojetvcygectelwkputfczgsfsbclnkzzcjfywitooygjwqujseflqyvqgyzpvknddz
```

프로그램이 식별자를 입력으로 예상하면, 이 정도로 긴 식별자가 입력으로 들어올 것이라고 예상할 수 있을까요?

#

**_Quiz_**
.  다음중 임의의 긴 10진수 문자열을 생성할 수 있는 코드는?

1. **fuzzer(100, 1, 100)**
2. **fuzzer(100, 100, 0)**
3. **fuzzer(100, 10, ord('0'))**
4. **fuzzer(100, ord('0'), 10)**

#

**정답**

```python
fuzz = fuzzer(100, ord('0'), 10)
print(fuzz)

>>> 050199092904721615267546627640773972382632848750065259698551700448752187153
```

#

## Fuzzing External programs

외부 프로그램에 실제로 fuzzed 입력을 넣으면 어떻게 되는지 봅시다. 이를 위해 두 단계로 진행하겠습니다. 첫 번째로 우리는 fuzzed 테스트 데이터로 입력 파일을 만든 다음, 이 입력 파일을 선택한 프로그램에 넣습니다.

#

### Creating Input Files

파일 시스템을 흐트러뜨리지 않도록 임시 파일 이름을 얻읍시다.

```python
import os
import tempfile
```

```python
basename = "input.txt"
tempdir = tempfile.mkdtemp()
FILE = os.path.join(tempdir, basename)
print(FILE)

>>> C:\Users\swlab\AppData\Local\Temp\tmp1p7w1ho4\input.txt
```

이제 쓰기 작업을 위해 이 파일을 열 수 있습니다. 파이썬 open() 함수는 임의의 내용을 쓸 수 있는 파일을 엽니다. 일반적으로 with 문과 함께 사용되어 파일이 더 이상 필요하지 않으면 즉시 닫을 수 있습니다.

```python
data = fuzzer()
with open(FILE, "w") as f:
    f.write(data)
```

파일 내용을 읽으면 파일이 실제로 생성되었는지 확인할 수 있습니다.

```python
contents = open(FILE).read()
print(contents)
assert(contents == data)

>>> !.:),08>84-$3<-02-6/ 5<6>1'+;<;06,2>5&$4560-+"
```

#

### Invoking External Programs

[bc download](https://ftp.jaist.ac.jp/pub/GNU/bc/)

이제 입력파일도 있으니, **bc** 계산기 프로그램을 테스트 해봅시다.

**bc**를 호출하려면 파이썬 **subprocess** 모듈을 사용하십시오. 작동 방식은 다음과 같습니다.

```python
import os
import subprocess
```

```python
program = "bc"
with open(FILE, "w") as f:
    f.write("2 + 2\n")
result = subprocess.run([program, FILE],
                        stdin=subprocess.DEVNULL,
                        stdout=subprocess.PIPE,
                        stderr=subprocess.PIPE,
                        universal_newlines=True)  # Will be "text" in Python 3.7
```

우리는 **결과**로 부터 프로그램 출력을 확인할 수 있습니다. **bc**의 경우, 출력은 산술식을 계산한 결과입니다.

```python
print(result.stdout)

>>> '4\n'
```

우리는 또한 상태를 확인할 수 있습니다. 값이 0이면 프로그램이 올바르게 종료되었음을 나타냅니다.

```python
print(result.returncode)

>>> 0
```

모든 오류 메시지는 **result.stderr**를 통해 볼 수 있습니다.

```python
print(result.stderr)

>>> ''
```

**bc** 대신, 당신이 좋아하는 어떤 프로그램도 넣을 수 있습니다. 그러나 프로그램이 시스템을 변경하거나 손상시킬 수 있는 경우 fuzzed 입력에 이러한 작업을 수행하는 데이터 또는 명령어가 포함될 수 있으니 주의해야 합니다.

#

**_Quiz_**
.  재미삼아 파일 제거 프로그램(예: **rm -rf FILE**)을 테스트한다고 상상해 보십시오. 여기서 **FILE**은 fuzzer()에 의해 생성된 문자열입니다. **fuzzer()** (기본 인수 포함)가 **FILE** 인수를 생성하여 모든 파일을 삭제할 가능성은 얼마나 됩니까?

1. About one in a billion
2. About one in a million
3. About one in a thousand
4. About one in ten

#

사실 그 가능성은 당신이 생각하는 것보다 높습니. 예를 들어 **/** (루트 디렉터리)를 제거하면 전체 파일 시스템이 사라집니다. **~** (홈 디렉터리)를 제거하면 모든 파일이 사라집니다. **.** (현재 디렉터리)를 제거하면 현재 디렉터리의 모든 파일이 사라집니다. 이 중 하나를 만들기 위해서는 문자열 길이가 1 (100개 중 1개)과 이 세글자 중 하나 (32개 중 3개)가 필요합니다. 실제로 1000분의 1확률입니다.

```python
1/100 * 3/32

>>> 0.0009375
```

그러나 두 번째 문자가 공백인 한 어떤 문자열도 실제로 처리할 수 있습니다. 즉 **rm -rf / WHATEVER**는 먼저 /를 처리하고 그 다음 이어지는 문자열을 처리할 수 있습니다. 첫 번째 문자는 32개 중 3개, 공백은 32개 중 1개입니다. 그래서 우리는 300개 중 1개입니다.

```python
3/32 * 1/32

>>> 0.0029296875
```

퍼징 테스트가 일반적으로 수백만 번 실행된다는 점을 고려할 때 이 위험을 감수하고 싶지 않을 것입니다. 도커 컨테이너와 같이 원하는 대로 재설정할 수 있는 안전한 환경에서 퍼저를 실행하세요.

#

### Long-Running Fuzzing

이제 테스트한 프로그램에 많은 수의 입력을 제공하여 프로그램이 충돌하는지 여부를 확인합시다. runs 변수는 모든 결과를 입력 데이터와 실제 결과의 쌍으로 저장합니다. (참고: 이 작업을 실행하는 데 시간이 걸릴 수 있습니다.)

```python
trials = 100
program = "bc"

runs = []

for i in range(trials):
    data = fuzzer()
    with open(FILE, "w") as f:
        f.write(data)
    result = subprocess.run([program, FILE],
                            stdin=subprocess.DEVNULL,
                            stdout=subprocess.PIPE,
                            stderr=subprocess.PIPE,
                            universal_newlines=True)
    runs.append((data, result))
```

이제 우리는 실행 결과에 대한 통계를 구할 수 있습니다. 예를 들어 입력값이 유효한지 검사하고 유효하면 리스트에 추가를 합니다. 그리고 이러한 유효한 식을 1로 표현합니다. 여기서 이 1의 개수가 몇개인지 세면 통과한 케이스가 몇개인지 알 수 있습니다.

```python
sum(1 for (data, result) in runs if result.stderr == "")

>>> 6
```

대부분의 입력은 유효하지 않으며, 무작위 입력에 유효한 산술 식이 포함될 가능성이 낮기 때문에 크게 놀랄 일은 아닙니다.

첫 번째 에러 메시지를 봐봅시다.

```python
errors = [(data, result) for (data, result) in runs if result.stderr != ""]
(first_data, first_result) = errors[0]

print(repr(first_data))
print(first_result.stderr)
```

```python
'47&&,/8/\'(9-(!#%.7%7=+%0((<!2=>*=,7(>?67$1" ?668>&97(>\'0! 1=01 $;\'050 !.#;;\'2&= ?\'8'
/tmp/tmpfw0cxb3p/input.txt 1: syntax error
/tmp/tmpfw0cxb3p/input.txt 1: illegal character: '
(standard_in) 1: syntax error
```

잘못된 문자, 구문 분석 오류 또는 구문 오류 이외의 메시지가 포함된 실행이 있습니까? (예를 들어, 충돌이나 치명적인 버그를 발견했는가?) 많지 않음:

```
errors = [result.stderr for (data, result) in runs if
 result.stderr != ""
 and "illegal character" not in result.stderr
 and "parse error" not in result.stderr
 and "syntax error" not in result.stderr]

 print(errors)


 >>> []
 ```

아마도 충돌은 bc가 그냥 충돌하는 것으로 나타날 것입니다.안타깝게도 반환 코드는 0이 아닙니다.

```python
result = sum(1 for (data, result) in runs if result.returncode != 0)
print(result)

>>> 0
```

위의 bc 테스트를 좀 더 진행하면 어떨까요? 테스트가 진행되는 동안, 1989년의 최첨단 기술이 어땠는지 살펴봅시다.

#

## Bugs Fuzzers Find

1989년 Miller와 그의 학생들이 첫 퍼저를 실행했을 때 놀라운 결과를 얻었습니다. 퍼징한 UNIX 유틸리티의 약 3분의 1이 문제가 있었습니다. 퍼징 입력이 들어갔을 때 충돌, 중단 또는 실패가 발생했다는 것입니다. [Miller et al, 여기에는 위의 bc 프로그램도 포함되어 있습니다. (분명히, 이제 버그가 수정되었습니다!)

이러한 UNIX 유틸리티 중 많은 부분이 네트워크 입력을 처리하는 스크립트에 사용되었음을 고려하면, 이는 놀라운 결과였습니다. 프로그래머들은 신속하게 자체 퍼저를 구축하고 실행했으며, 보고된 오류를 수정하기 위해 서둘렀고, 더 이상 외부 입력을 신뢰하지 않는 것을 배웠습니다.

밀러의 실험 결과 어떤 문제가 발견되었나요? 프로그래머들이 1990년에 저지른 실수는 오늘날에도 여전히 같은 실수라는 것이 밝혀졌습니다.

#

### Buffer Overflows

많은 프로그램에는 입력 및 입력 요소에 대한 최대 길이가 내장되어 있습니다. C와 같은 언어에서는 프로그램(또는 프로그래머)이 눈치채지 못하게 이 길이를 초과하기 쉬우며, 이른바 버퍼 오버플로(buffer overflow)를 유발합니다. 예를 들어, 다음 코드는 입력이 8자를 초과하더라도 입력 문자열을 weekday 문자열로 복사합니다.

```c
char weekday[9]; // 8 characters + trailing '\0' terminator
strcpy (weekday, input);
```

아이러니하게도, 입력이 "Wednesday"(9자)이면 이 작업은 실패합니다. 초과 문자('y'와 '\0' 문자열 종결자)는 weekday 이후 메모리 공간으로 복사되어 임의 동작을 트리거합니다. <span style="background-color: purple">아마도 'n'에서 'y'로 설정된 부울 문자 변수일 수 있습니다.</span>퍼징을 사용하면 임의의 긴 입력 및 입력 요소를 매우 쉽게 생성할 수 있습니다.

우리는 파이썬 함수로 버퍼 오버플로우를 쉽게 시뮬레이션 할 수 있습니다.

```python
def crash_if_too_long(s):
    buffer = "Thursday"
    if len(s) > len(buffer):
        raise ValueError
```

```python
from fuzzingbook.ExpectError import ExpectError
```

```python
trials = 100
with ExpectError():
    for i in range(trials):
        s = fuzzer()
        crash_if_too_long(s)
```

```python
Traceback (most recent call last):
  File "c:\Users\Python\hello.py", line 24, in <module>
    crash_if_too_long(s)
  File "c:\Users\Python\hello.py", line 17, in crash_if_too_long
    raise ValueError
ValueError (expected)
```

위의 코드에 있는 **ExpectError()** 행은 오류 메시지가 출력되지만 실행은 계속된다는 것을 보장합니다.

#

### Missing Error Checks

많은 프로그래밍 언어들은 예외가 없지만, 대신 함수가 예외적인 상황에서 특별한 **오류 코드**를 반환합니다.예를 들어, C언어 함수 getchar()는 일반적으로 표준 입력으로부터 문자를 반환합니다. 입력이 없다면 **EOF**를 반환합니다. 이제 프로그래머가 공백 문자를 읽을 때까지 getchar()로 문자를 읽는다고 가정해봅시다.

```c
while (getchar() != ' ');
```

입력이 조기에 종료되면 어떻게 될까요? **getchar()는 EOF**를 반환하고, 다시 호출할 때 **EOF**를 계속 반환하므로 위의 코드는 단순히 무한 루프에 들어갑니다.

다시, 우리는 그러한 누락된 오류 검사를 시뮬레이션할 수 있습니다. 입력에 공백이 없을 경우 효과적으로 정지되는 함수는 다음과 같습니다.

```python
def hang_if_no_space(s):
    i = 0
    while True:
        if i < len(s):
            if s[i] == ' ':
                break
        i += 1
```

Introduction to Testing의 timeout 메커니즘을 사용하면 시간이 지난후 이 함수를 중단시킬 수 있습니다. 몇 번의 퍼징 입력을 넣으면 TimeoutError 메시지를 출력하고 함수를 중단시키게 됩니다.

```python
from fuzzingbook.ExpectError import ExpectTimeout
```

```python
trials = 100
with ExpectTimeout(2):
    for i in range(trials):
        s = fuzzer()
        hang_if_no_space(s)
```

```python
Traceback (most recent call last):
  File "c:\Users\Python\Python\hello.py", line 34, in <module>
    hang_if_no_space(s)
  File "c:\Users\Python\Python\hello.py", line 23, in hang_if_no_space
    while True:
  File "c:\Users\Python\Python\hello.py", line 23, in hang_if_no_space
    while True:
  File "C:\Users\AppData\Local\Programs\Python\Python310\lib\site-packages\fuzzingbook\Timeout.py", line 191, in check_time
    raise TimeoutError
TimeoutError (expected)
```

위 코드의 **with ExpectTimeout()** 행은 코드 실행을 2초 후에 중단하고 오류 메시지가 출력되도록 합니다.


#


### Rogue Numbers

퍼징을 사용하면 입력에서 **일반적이지 않은 값**을 생성하여 모든 종류의 흥미로운 동작을 유발하기 쉽습니다. 다음 코드를 C 언어로 다시 생각해 보십시오. 이 코드는 먼저 입력에서 버퍼 크기를 읽은 다음 지정된 크기의 버퍼를 할당합니다.

```c
char *read_input() {
    size_t size = read_buffer_size();
    char *buffer = (char *)malloc(size);
    // fill buffer
    return (buffer);
}
```

**크기**가 프로그램 메모리를 초과하여 매우 크면 어떻게 될까요? **크기**가 입력으로 들어오는 문자 수보다 적으면 어떻게 될까요? **크기**가 음수이면 어떻게 될까요? 여기에 난수를 제공함으로써, 퍼징은 모든 종류의 오류를 발생시킬 수 있습니다.

다시 말하지만, 우리는 파이썬에서 저러한 악의적인 숫자를 쉽게 시뮬레이션할 수 있습니다. 정수로 변환된 후 전달된 값(문자열)이 너무 크면 **collapse_if_too_large()** 함수는 실행에 실패할 것입니다.

```python
def collapse_if_too_large(s):
    if int(s) > 1000:
        raise ValueError
```

우리는 **fuzzer()**가 숫자 문자열을 생성하도록 할 수 있습니다.

```python
long_number = fuzzer(100, ord('0'), 10)
print(long_number)

>>> 75796082745267671405786577704870880086215805248523282954251610864940357086645564
```

만약 우리가 저런 숫자들을 **collapse_if_too_large()** 함수의 인자로 넣으면, 함수는 실행에 실패하여 오류 메시지를 출력할 것입니다.

```python
with ExpectError():
    collapse_if_too_large(long_number)
```

```python
Traceback (most recent call last):
  File "c:\Users\Python\hello.py", line 45, in <module>
    collapse_if_too_large(long_number)
  File "c:\Users\Python\hello.py", line 39, in collapse_if_too_large
    raise ValueError
ValueError (expected)
```

만약 우리가 정말로 시스템에 그렇게 많은 메모리를 할당하고 싶다면, 실제로 위와 같이 빠르게 실패하도록 하는 것이 더 나은 선택일 것입니다. 실제로 메모리 부족으로 인해 시스템이 완전히 반응하지 않을 정도로 속도가 급격히 느려질 수 있으며, 재시작만이 유일한 옵션이 될 수도 있습니다.

누군가는 이것이 모두 나쁜 프로그래밍 또는 나쁜 프로그래밍 언어의 문제라고 주장할 수 있습니다. 하지만, 매일 수천명의 사람들이 프로그램을 짜기 시작하고, 그들은 모두 같은 실수를 반복하고, 심지어 오늘날에도 이러한 오류들은 계속 발생합니다.


#


## Catching Errors

Miller와 그의 학생들이 첫 번째 퍼저를 만들었을 때, 그들은 단순히 프로그램이 중단되거나 중단된다는 이유만으로 오류를 식별할 수 있었습니다. 위 두 가지 조건은 쉽게 식별할 수 있었습니다. 하지만 실패가 더 미묘하다면, 우리는 추가적인 점검을 해야 합니다.

#


### Generic Checkers

위에서 설명한 바와 같이 버퍼 오버플로우는 보다 일반적인 문제의 특정 예입니다. C 및 C++와 같은 언어에서 프로그램은 메모리의 임의 부분(초기화되지 않았거나 이미 해제되었거나 단순히 접근하려는 데이터 구조의 일부가 아닌 부분)에 접근할 수 있습니다. 운영 체제를 작성하려는 경우 이 작업이 필요하며, 최대 성능이나 제어 능력을 원하는 경우 매우 유용합니다. 하지만 실수를 방지하기에는 매우 좋지 않습니다. 다행히 런타임에 이러한 문제를 해결하는 데 도움이 되는 도구가 있으며, 퍼징과 결합하면 매우 좋습니다.

#


**Checking Memory Accesses**

테스트하는 동안 문제가 있는 메모리 접근을 포착하기 위해 특별한 메모리 검사 환경에서 C 프로그램을 실행할 수 있습니다. 실행 시 이러한 프로그램은 유효하고 초기화된 메모리에 액세스하는지 여부를 검사합니다. 가장 일반적인 예는 잠재적으로 위험한 메모리 안전 위반을 탐지하는 LLVM Address Santizer입니다. 다음 예제에서는 이 도구를 사용하여 다소 간단한 C 프로그램을 컴파일하고 할당된 메모리 부분을 읽음으로써 범위를 벗어난 읽기를 유발합니다.

```python
with open("program.c", "w") as f:
    f.write("""
#include <stdlib.h>
#include <string.h>

int main(int argc, char** argv) {
    /* Create an array with 100 bytes, initialized with 42 */
    char *buf = malloc(100);
    memset(buf, 42, 100);

    /* Read the N-th element, with N being the first command-line argument */
    int index = atoi(argv[1]);
    char val = buf[index];

    /* Clean up memory so we don't leak */
    free(buf);
    return val;
}
    """)
```

```python
from bookutils import print_file
```

```python
print_file("program.c")
```

```c
#include <stdlib.h>
#include <string.h>

int main(int argc, char** argv) {
    /* Create an array with 100 bytes, initialized with 42 */
    char *buf = malloc(100);
    memset(buf, 42, 100);

    /* Read the N-th element, with N being the first command-line argument */
    int index = atoi(argv[1]);
    char val = buf[index];

    /* Clean up memory so we don't leak */
    free(buf);
    return val;
}
```

우리는 C 프로그램을 address sanitization가 활성화된 상태에서 컴파일 합니다.

```
!clang -fsanitize=address -g -o program program.c
```

인수가 **99**인 프로그램을 실행하면 buff[99]가 반환되는데, 이는 42입니다.

```bash
./program 99; echo $?

>>> 42
```

그러나 buf[110]에 접근하면 AddressSanitizer에서 Out-of-bounds 오류가 발생합니다.

```bash
./program 110
```

```bash
=================================================================
==653==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60b00000015e at pc 0x00000051219e bp 0x7ffe8606eab0 sp 0x7ffe8606eaa8
READ of size 1 at 0x60b00000015e thread T0
    #0 0x51219d  (/home/ion/fuzzingbook/program+0x51219d)
    #1 0x7f499695dc86  (/lib/x86_64-linux-gnu/libc.so.6+0x21c86)
    #2 0x419d19  (/home/ion/fuzzingbook/program+0x419d19)

0x60b00000015e is located 10 bytes to the right of 100-byte region [0x60b0000000f0,0x60b000000154)
allocated by thread T0 here:
    #0 0x4d9bd0  (/home/ion/fuzzingbook/program+0x4d9bd0)
    #1 0x512104  (/home/ion/fuzzingbook/program+0x512104)
    #2 0x7f499695dc86  (/lib/x86_64-linux-gnu/libc.so.6+0x21c86)

SUMMARY: AddressSanitizer: heap-buffer-overflow (/home/ion/fuzzingbook/program+0x51219d)
Shadow bytes around the buggy address:
  0x0c167fff7fd0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c167fff7fe0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c167fff7ff0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0c167fff8000: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c167fff8010: fd fd fd fd fd fa fa fa fa fa fa fa fa fa 00 00
=>0x0c167fff8020: 00 00 00 00 00 00 00 00 00 00 04[fa]fa fa fa fa
  0x0c167fff8030: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c167fff8040: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c167fff8050: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c167fff8060: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c167fff8070: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==653==ABORTING
```

만약 당신이 C 프로그램에서 오류를 찾고 싶다면, 퍼징에 대한 검사를 켜는 것은 꽤 쉽습니다. 툴(AddressSanitizer의 경우 일반적으로 2X)에 따라 실행 속도가 일정하게 느려지고 메모리도 더 많이 소모되지만 이러한 버그를 찾는 데 필요한 인간의 노력에 비해 CPU 사이클은 매우 저렴합니다.

메모리에 대한 범위를 벗어나는 접근은 공격자가 의도하지 않은 정보에 접근하거나 수정할 수 있기 때문에 보안 위험이 큽니다. 유명한 예로, HeartBleed 버그는 OpenSSL 라이브러리의 보안 버그입니다.
(**OpenSSL**: 컴퓨터 네트워크를 통한 통신 보안을 제공하는 암호화 프로토콜)

HeartBleed 버그는 특수하게 조작된 명령을 SSL 하트비트 서비스에 전송하여 익스플로잇을 합니다. 하트비트 서비스는 다른 쪽 끝에 있는 서버가 여전히 활성 상태인지 확인하는 데 사용됩니다. 클라이언트는 서비스를 다음과 같은 문자열로 보냅니다.

```bash
문자열 (문자열 길이)
BIRD (4 letters)
```

서버가 BIRD로 응답하면 클라이언트는 서버가 활성 상태임을 알 수 있습니다.

안타깝게도 서버가 요청한 문자열보다 많은 문자로 회신하도록 요청하여 이 서비스를 익스플로잇 할 수 있습니다. 이것은 이 XKCD 만화에 잘 설명되어 있다.

![comic1](https://velog.velcdn.com/images/silvergun8291/post/cb548b0d-91f8-44f8-8d7d-1ae56947e2a8/image.png)


![comic2](https://velog.velcdn.com/images/silvergun8291/post/36dd5142-bda3-44f2-991e-661709095593/image.png)


![comic3](https://velog.velcdn.com/images/silvergun8291/post/aed9f17f-84dc-4cf2-8598-aed1c579804a/image.png)

OpenSSL 구현에서는 이러한 메모리 컨텐츠에 암호화 인증서, 개인 키 등이 포함될 수 있으며, 더 나쁜 것은 이 메모리가 방금 액세스되었다는 사실을 아무도 눈치채지 못할 것이라는 점입니다. HeartBleed가 발견되었을 때, 그것은 수년 동안 존재해 왔고, 아무도 이미 어떤 비밀이 유출되었는지와 어떤 것이 이미 유출되었는지 알 수 없을 것입니다; HeartBleed 발표 페이지에서 모든 것을 말해줍니다.

하지만 HeartBleed는 어떻게 발견되었을까요? 아주 간단합니다. 구글과 코데노미콘 회사의 연구원들은 memory sanitizer로 OpenSSL 라이브러리를 컴파일하고 퍼저로 생성한 명령어들로 테스트 했습니다. 그러자 memory sanitizer는 out-of-bounds 에러가 발생했다고 알려줬습니다.

메모리 검사기는 퍼징 테스트를 하는 동안 런타임 오류를 감지하기 위해 실행할 수 있는 많은 검사기 중 하나입니다. mining function specifications 챕터에서 generic checker를 정의하는 방법에 대해 자세히 알아보겠습니다.

프로그램이 종료되었으므로 **program** 파일을 정리합시다.

```bash
rm -fr program program.*
```


#

**Information Leaks**

정보 누수는 불법 메모리 액세스를 통해 발생할 수 있을 뿐만 아니라 "유효한" 메모리 내에서 발생할 수 있습니다. 이 "유효한" 메모리에 누출되지 않아야 하는 중요한 정보가 포함되어 있는 경우입니다. 이 문제를 Python 프로그램에서 설명하겠습니다. 먼저 실제 데이터와 랜덤 데이터로 채워진 프로그램 메모리를 생성해 보겠습니다.

```python
secrets = ("<space for reply>" + fuzzer(100) +
           "<secret-certificate>" + fuzzer(100) +
           "<secret-key>" + fuzzer(100) + "<other-secrets>")
```

uninitialized_memory_marker에 "deadbeef"를 대입하고 secrets 문자열에 이어 붙입니다.

```python
uninitialized_memory_marker = "deadbeef"
while len(secrets) < 2048:
    secrets += uninitialized_memory_marker
```

길이뿐만 아니라 응답을 다시 보낼 수 있는 서비스(위에서 설명한 하트비트 서비스와 유사)를 정의합니다. 이 서비스는 전송할 응답을 메모리에 저장한 다음 지정된 길이로 다시 전송합니다.

```python
def heartbeat(reply: str, length: int, memory: str) -> str:
    # Store reply in memory
    memory = reply + memory[len(reply):]

    # Send back heartbeat
    s = ""
    for i in range(length):
        s += memory[i]
    return s
```

이것은 표준 문자열에 완벽하게 적용됩니다.

```python
reply = heartbeat("potato", 6, memory=secrets)
print(reply)

>>> potato
```

```python
reply = heartbeat("bird", 4, memory=secrets)
print(reply)

>>> bird
```

그러나 길이가 응답 문자열의 길이보다 크면 메모리의 추가 내용이 유출됩니다. 이 모든 것은 여전히 정규 배열 범위 내에서 발생하므로 address sanitizer가 작동하지 않습니다.

```python
reply = heartbeat("hat", 500, memory=secrets)
print(reply)
```

```python
hatace for reply>!7#%"*#0=)$;%6*;>638:*>80"=</>(/*:-(2<4 !:5*6856&?""11<7+%<%7,4.8,*+&,,$,."<secret-certificate>5%<%76< -5 7:,>((/$$-/->.;.=;(.%!:50#7*8=$&&=$9!%6(4=&69':'<3+0-3.24#7=!&60)2/+";+<7+1<2!4$>92+$1<(3<secret-key>&5''>#28($<other-secrets>deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdea
```

어떻게 하면 그런 문제들을 감지할 수 있을까요? 아이디어는 secrets 처럼 유출 되면 안될 정보 뿐만 아니라 초기화 되지 않은 메모리도 식별하자는 것입니다. Python 예제에서 이러한 검사를 시뮬레이션할 수 있습니다.

```python
from fuzzingbook.ExpectError import ExpectError
```

```python
with ExpectError():
    for i in range(10):
        s = heartbeat(fuzzer(), random.randint(1, 500), memory=secrets)
        assert not s.find(uninitialized_memory_marker)
        assert not s.find("secret")
```

```python
Traceback (most recent call last):
  File "c:\Users\The Fuzzing Book\Code\fuzzingbook\hello.py", line 40, in <module>
    assert not s.find(uninitialized_memory_marker)
AssertionError (expected)
```

이러한 검사를 통해 우리는 secrets 및/또는 uninitialized memory가 실제로 유출된다는 것을 알게 되었습니다. information flow 챕터에서, 이를 자동으로 수행하는 방법과 민감한 정보와 그로부터 도출된 값이 새어나가지 않도록 하는 방법에 대해 배웁니다.

경험에 비추어 볼 때, 퍼징 테스트를 하는 동안에는 가능한 많은 자동 검사기를 활성화 해야 합니다. CPU 사이클은 저렴하고 오류는 비쌉니다. 실제로 오류를 감지하는 옵션 없이 프로그램만 실행하면 몇 가지 기회를 놓치게 됩니다.


#


### Program-Specific Checkers

지정된 플랫폼 또는 언어의 모든 프로그램에 적용되는 일반 검사기 외에도 프로그램 또는 하위 시스템에 적용되는 특정 검사기를 고안할 수 있습니다. testing 챕터에서 이미 런타임에 함수 결과가 정확한지 확인하는 런타임 검증 기술을 암시했습니다.

오류를 조기에 발견하기 위한 핵심 아이디어는 중요한 함수의 입력(전제 조건)과 결과(후제 조건)를 확인하는 assert 문입니다. 특히 퍼징 테스트를 하는 동안에 프로그램에 assert 문이 많을수록 일반 검사기에 의해 탐지되지 않는 오류를 실행 중에 탐지할 가능성이 높아집니다. 성능에 대한 assert 문의 영향을 우려하는 경우 프로덕션 코드에서 assert 문을 제거할 수 있습니다 (가장 중요한 검사를 활성 상태로 두는 것이 도움이 될 수 있음).

오류를 찾기 위한 assert 문의 가장 중요한 사용 방법 중 하나는 복잡한 데이터 구조의 무결성을 검증하는 것입니다. 간단한 예를 들어 개념을 설명하겠습니다. airport_codes를 공항과 매핑한다고 가정해 보겠습니다.

```python
from typing import Dict
```

```python
airport_codes: Dict[str, str] = {
    "YVR": "Vancouver",
    "JFK": "New York-JFK",
    "CDG": "Paris-Charles de Gaulle",
    "CAI": "Cairo",
    "LED": "St. Petersburg",
    "PEK": "Beijing",
    "HND": "Tokyo-Haneda",
    "AKL": "Auckland"
}  # plus many more
```

```python
airport = airport_codes["YVR"]
print(airport)

>>> Vancouver
```

```
result = "AKL" in airport_codes
print(result)

>>> True
```

이 airport code 리스트는 매우 중요할 수 있습니다. 공항 코드 중 하나에서 맞춤법이 틀린 경우 어떤 응용 프로그램이든 영향을 미칠 수 있습니다. 따라서 리스트의 일관성을 확인하는 기능을 도입합니다. 일관성 조건을 representation invariant이라고 하며, 이를 확인하는 함수(또는 메소드)는 일반적으로 the representation is ok이라는 뜻의 repOK()로 명명됩니다.

우선 개별 공항 코드를 검사해 봅시다. 코드가 일치하지 않으면 검사기가 실패합니다.

```python
def code_repOK(code: str) -> bool:
    assert len(code) == 3, "Airport code must have three characters: " + repr(code)
    for c in code:
        assert c.isalpha(), "Non-letter in airport code: " + repr(code)
        assert c.isupper(), "Lowercase letter in airport code: " + repr(code)
    return True
```

```python
assert code_repOK("SEA")
```

이제 code_repOK()를 사용하여 리스트의 모든 요소를 확인할 수 있습니다.

```python
def airport_codes_repOK():
    for code in airport_codes:
        assert code_repOK(code)
    return True
```

```python
with ExpectError():
    assert airport_codes_repOK()
```

목록에 잘못된 요소를 추가하면 검사가 실패합니다.

```python
airport_codes["YMML"] = "Melbourne"
```

```python
with ExpectError():
    assert airport_codes_repOK()
```

```python
Traceback (most recent call last):
  File "c:\Users/The Fuzzing Book\Code\fuzzingbook\hello.py", line 45, in <module>
    assert airport_codes_repOK()
  File "c:\Users\The Fuzzing Book\Code\fuzzingbook\hello.py", line 38, in airport_codes_repOK
    assert code_repOK(code)
  File "c:\Users\The Fuzzing Book\Code\fuzzingbook\hello.py", line 29, in code_repOK
    assert len(code) == 3, "Airport code must have three characters: " + repr(code)
AssertionError: Airport code must have three characters: 'YMML' (expected)
```

물론 리스트를 직접 조작하는 대신 요소를 추가하는 특별한 기능이 있습니다. 그러면 코드가 유효한지도 확인할 수 있습니다:

```python
def add_new_airport(code: str, city: str) -> None:
    assert code_repOK(code)
    airport_codes[code] = city
```

```python
with ExpectError():  # For BER, ExpectTimeout would be more appropriate
    add_new_airport("BER", "Berlin")
```

또한 이 검사를 통해 인수 리스트에서 오류를 찾을 수 있습니다.

```python
with ExpectError():
    add_new_airport("London-Heathrow", "LHR")
```

```python
Traceback (most recent call last):
  File "c:\Users\The Fuzzing Book\Code\fuzzingbook\hello.py", line 48, in <module>
    add_new_airport("London-Heathrow", "LHR")
  File "c:\Users\The Fuzzing Book\Code\fuzzingbook\hello.py", line 43, in add_new_airport
    assert code_repOK(code)
  File "c:\Users\The Fuzzing Book\Code\fuzzingbook\hello.py", line 29, in code_repOK
    assert len(code) == 3, "Airport code must have three characters: " + repr(code)
AssertionError: Airport code must have three characters: 'London-Heathrow' (expected)
```

그러나 최대 점검을 위해 **add_new_airport()** 기능을 사용하면 공항 코드 리스트를 변경 전후에 정확하게 표시할 수 있습니다.

```python
def add_new_airport_2(code: str, city: str) -> None:
    assert code_repOK(code)
    assert airport_codes_repOK()
    airport_codes[code] = city
    assert airport_codes_repOK()
```

<span style="background-color: purple">
이는 앞에서 설명한 불일치를 포착합니다.
</span>

```python
with ExpectError():
    add_new_airport_2("IST", "Istanbul Yeni Havalimanı")
```

코드에 **repOK()** assert 문이 많을수록 더 많은 오류를 발견할 수 있으며, 너의 영역과 문제에만 국한된 것들도 마찬가지 입니다. 또한 이러한 assert 문은 프로그래밍 중에 사용자가 만든 assert 문을 문서화하여 다른 프로그래머가 코드를 이해하고 오류를 방지하는 데 도움이 됩니다.

마지막 예로, 레드-블랙 트리, 셀프 밸런싱 이진 검색 트리 등 다소 복잡한 데이터 구조를 고려해 보겠습니다. 레드-블랙 트리를 구현하는 것은 그리 어렵지 않지만, 그것을 정확하게 하는 것은 숙련된 프로그래머에게도 몇 시간의 작업이 될 수 있습니다. 그러나 **rePOK()** 방법은 모든 가정을 문서화하고 검사합니다.

```python
class RedBlackTree:
    def repOK(self):
        assert self.rootHasNoParent()
        assert self.rootIsBlack()
        assert self.rootNodesHaveOnlyBlackChildren()
        assert self.treeIsAcyclic()
        assert self.parentsAreConsistent()
        return True

    def rootIsBlack(self):
        if self.parent is None:
            assert self.color == BLACK
        return True

    def add_element(self, elem):
        assert self.repOK()
        ...  # Add the element
        assert self.repOK()

    def delete_element(self, elem):
        assert self.repOK()
        ...  # Delete the element
        assert self.repOK()
```

여기서 **repOK()**는 **RedBlackTree** 클래스의 개체에서 실행되는 메서드입니다. 그것은 다섯 가지 다른 검사를 실행하는데, 모두 자신만의 assert 문을 가지고 있습니다. 요소를 추가하거나 삭제할 때마다 이러한 모든 일관성 검사가 자동으로 실행됩니다. 물론 충분히 많은 퍼저에 의해 생성된 입력을 통해 트리를 실행하면 이러한 오류가 있는 경우 검사기가 오류를 찾습니다.

#


### Static Code Checkers

**repOK()** assert 문의 많은 이점은 코드에서 정적 형식 체커를 사용하여 얻을 수도 있습니다. 예를 들어 파이썬에서 MyPy 정적 검사기는 인수의 유형이 적절하게 선언되는 즉시 유형 오류를 찾을 수 있습니다.

```bash
# MyPy 설치
python3 -m pip install -U mypy
```

```python
typed_airport_codes: Dict[str, str] = {
    "YVR": "Vancouver",  # etc
}
```

이제 문자열이 아닌 유형의 키를 추가하는 경우

```python
typed_airport_codes[1] = "First"
```

이 오류는 MyPy에서 즉시 확인할 수 있습니다.

```bash
$ mypy airports.py
airports.py:45: error: Invalid index type "int" for "Dict[str, str]"; expected type "str"
Found 1 error in 1 file (checked 1 source file)
```

그러나 정확히 세 개의 대문자로 구성된 공항 코드나 트리가 비순환적인 것과 같은 더 고급 속성을 정적으로 확인하는 것은 정적인 체킹의 한계에 빠르게 도달합니다. 하지만 **repOK()** assert 문은 여전히 필요합니다. 좋은 테스트 생성기와 함께 사용하는 것이 가장 좋습니다.

#


## A Fuzzing Architecture

다음 장에서 이 장의 일부를 재사용하고자 하므로, 재사용하기 쉽고, 특히 확장하기 쉬운 방식으로 정의해 보겠습니다. 이를 위해 위의 기능을 재사용 가능한 방식으로 캡슐화하는 여러 클래스를 소개합니다.

#

### Runner Classes

첫 번째로 소개하는 것은 **Runner**의 개념, 즉 주어진 입력으로 어떤 객체를 실행하는 역할을 하는 객체이다. runner는 일반적으로 테스트 중인 프로그램이나 함수지만, 우리는 더 간단한 러너를 가질 수도 있습니다.

runners를 기본 클래스로 시작해봅시다. runner는 기본적으로 입력(문자열)을 runner에게 전달하기 위해 사용되는 메서드 **run**(입력)을 제공합니다. run()은 pair(result, outcome)을 반환합니다. 여기서 result는 run에 대한 세부 정보를 제공하는 runner의 특별한 값이며, outcome은 result를 세 가지 범주로 분류하는 값입니다.

* Runner.PASS – 테스트 통과. 실행 결과 올바른 결과가 나왔습니다.
* Runner.FAIL – 테스트 실패. 실행 결과 잘못된 결과가 나왔습니다.
* Runner.UNRESOLVED – 테스트 통과, 실패 둘다 아님. 입력이 잘못되어 실행할 수 없을 때 발생합니다.

```python
from typing import Tuple
from typing import List
from typing import Union
import subprocess
```

```python
Outcome = str
```

```python
class Runner:
    """Base class for testing inputs."""

    # Test outcomes
    PASS = "PASS"
    FAIL = "FAIL"
    UNRESOLVED = "UNRESOLVED"

    def __init__(self) -> None:
        """Initialize"""
        pass

    def run(self, inp: str):
        """Run the runner with the given input"""
        return (inp, Runner.UNRESOLVED)
```

기본 클래스로서, **Runner**는 단지 그것을 기반으로 하는 더 복잡한 runners들을 위한 인터페이스를 제공합니다. 보다 구체적으로, 우리는 메서드를 추가하기 위해서 또는 메서드를 오버라이드 하기 위해서 부모 클래스에서 메서드를 상속받는 자식 클래스를 소개합니다.

다음은 이러한 자식 클래스의 한 가지 예입니다. **PrintRunner**는 상속된 run() 메서드를 오버라이드하여 주어진 모든 것을 출력합니다.

```python
class PrintRunner(Runner):
    """Simple runner, printing the input."""

    def run(self, inp):
        """Print the given input"""
        print(inp)
        return (inp, Runner.UNRESOLVED)
```

```python
p = PrintRunner()
(result, outcome) = p.run("Some input")

>>> Some input
```

result는 입력으로 전달된 문자열입니다.

```python
print(result)

>>> some input
```

그러나 이 시점에서는 프로그램 동작을 분류할 방법이 없습니다.

```python
print(outcome)

>>> UNRESOLVED
```

**ProgramRunner** 클래스는 대신 프로그램의 표준 입력으로 입력을 보냅니다. **ProgramRunner** 객체를 만들 때 프로그램이 지정됩니다.

```python
class ProgramRunner(Runner):
    """Test a program with inputs."""

    def __init__(self, program: Union[str, List[str]]) -> None:
        """Initialize.
           `program` is a program spec as passed to `subprocess.run()`"""
        self.program = program

    def run_process(self, inp: str = "") -> subprocess.CompletedProcess:
        """Run the program with `inp` as input.
           Return result of `subprocess.run()`."""
        return subprocess.run(self.program,
                              input=inp,
                              stdout=subprocess.PIPE,
                              stderr=subprocess.PIPE,
                              universal_newlines=True)

    def run(self, inp: str = "") -> Tuple[subprocess.CompletedProcess, Outcome]:
        """Run the program with `inp` as input.  
           Return test outcome based on result of `subprocess.run()`."""
        result = self.run_process(inp)

        if result.returncode == 0:
            outcome = self.PASS
        elif result.returncode < 0:
            outcome = self.FAIL
        else:
            outcome = self.UNRESOLVED

        return (result, outcome)
```

다음은 binary(즉, non-textual) 입력 및 출력의 변형입니다.

```python
class BinaryProgramRunner(ProgramRunner):
    def run_process(self, inp: str = "") -> subprocess.CompletedProcess:
        """Run the program with `inp` as input.  
           Return result of `subprocess.run()`."""
        return subprocess.run(self.program,
                              input=inp.encode(),
                              stdout=subprocess.PIPE,
                              stderr=subprocess.PIPE)
```

입력을 출력으로 복사하는 프로그램인 **cat**을 사용하여 **ProgramRunner**를 시연해 보겠습니다. 우리는 **cat**의 표준 호출이 단순히 그 작업을 하고, **cat**의 출력은 입력과 같다는 것을 알 수 있습니다.

```python
cat = ProgramRunner(program="cat")
print(cat.run("hello"))

>>> (CompletedProcess(args='cat', returncode=0, stdout='hello', stderr=''), 'PASS')
```

#


### Fuzzer Classes

이제 소비자에게 실제로 데이터를 제공하는 퍼저를 정의해 보겠습니다. 퍼저의 기본 클래스는 일부 입력을 생성하는 하나의 중앙 메서드 **fuzz()를 제공합니다. 그런 다음 run()** 함수는 fuzz() 입력을 runner에게 보내 결과를 반환합니다; runs()는 지정된 횟수 동안 이 작업을 수행합니다.

```python
import random
```

```python
class Fuzzer:
    """Base class for fuzzers."""

    def __init__(self) -> None:
        """Constructor"""
        pass

    def fuzz(self) -> str:
        """Return fuzz input"""
        return ""

    def run(self, runner: Runner = Runner()) \
            -> Tuple[subprocess.CompletedProcess, Outcome]:
        """Run `runner` with fuzz input"""
        return runner.run(self.fuzz())

    def runs(self, runner: Runner = PrintRunner(), trials: int = 10) \
            -> List[Tuple[subprocess.CompletedProcess, Outcome]]:
        """Run `runner` with fuzz input, `trials` times"""
        return [self.run(runner) for i in range(trials)]
```

기본적으로 **Fuzzer** 객체는 **fuzz()** 함수가 추상 자리 표시자에 불과하기 때문에 많은 작업을 수행하지 않습니다. 그러나 하위 클래스 **RandomFuzer**는 위의 **fuzzer()** 함수의 기능을 구현하며 최소 길이를 지정하기 위해 추가 매개 변수 **min_length**를 추가합니다.

```python
class RandomFuzzer(Fuzzer):
    """Produce random inputs."""

    def __init__(self, min_length: int = 10, max_length: int = 100,
                 char_start: int = 32, char_range: int = 32) -> None:
        """Produce strings of `min_length` to `max_length` characters
           in the range [`char_start`, `char_start` + `char_range`)"""
        self.min_length = min_length
        self.max_length = max_length
        self.char_start = char_start
        self.char_range = char_range

    def fuzz(self) -> str:
        string_length = random.randrange(self.min_length, self.max_length + 1)
        out = ""
        for i in range(0, string_length):
            out += chr(random.randrange(self.char_start,
                                        self.char_start + self.char_range))
        return out
```

RandomFuzzer를 사용하면 퍼저를 만들 때 기본 구성을 한 번만 지정하면 되는 퍼저를 만들 수 있습니다.

```python
random_fuzzer = RandomFuzzer(min_length=20, max_length=20)
for i in range(10):
    print(random_fuzzer.fuzz())
```

```python
1-86816)'5=0*8,*/#;*
65"0+;'0-" 5%/%&7=5,
$ ?!?2>>,9>?%:#8%?0>
:,31*%0>>3:%??'70$(&
7< 4!2-!" )486- 2(($
.#(;&(#572/6%/2=)0"%
#!%0=025$17/#-/4/*28
.*9)#? 9:,3.)7+(#3-4
&)<,>3;-"='0,#*':+ !
 3> )9"-6"4%#?,<8$;4
```

우리는 이제 이전에 정의된 cat runner에게 생성된 입력을 전송하여 cat이 실제로 (퍼즈된) 입력을 출력으로 복사하는지 확인할 수 있습니다.

```python
for i in range(10):
    inp = random_fuzzer.fuzz()
    result, outcome = cat.run(inp)
    assert result.stdout == inp
    assert outcome == Runner.PASS
```

그러나 **Fuzzer**를 **Runner**와 결합하는 것은 매우 일반적이므로 이를 위해 **Fuzzer** 클래스에서 제공하는 **run()** 메서드를 사용할 수 있습니다.

```python
print(random_fuzzer.run(cat))

>>> (CompletedProcess(args='cat', returncode=0, stdout='=!0(".0/4*\'!<75&;:5.', stderr=''), 'PASS')
```

**runs()**를 사용하면 fuzzing run을 여러 번 반복하여 결과 리스트를 얻을 수 있습니다.

```python
print(random_fuzzer.runs(cat, 10))

```

```python
[(CompletedProcess(args='cat', returncode=0, stdout='(1\'=6<-(14&/,!++"+5>', stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout="'8$>!)/#'952(+=67:;1", stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout='96#:?/>"+%*:,+-)\',&/', stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout='"&31,%,2??$*1>41.?.8', stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout="+,6; %%.-1, ':2&#!,:", stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout='<*013\'!.;=7+932&" 4$', stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout="40'?*4&8;!;251 .7(92", stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout=')37)4588(%\'7)$30"?*#', stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout="'*<35=):?18;6'4>6)6#", stderr=''), 'PASS'), (CompletedProcess(args='cat', returncode=0, stdout='+)""2.+<=1&! +!<+?%7', stderr=''), 'PASS')]
```

이 장에서 소개한 간단한 random fuzzers부터 시작해서 훨씬 더 발전된 fuzzers들을 만들기 위해 우리는 모든 준비가 됬습니다. 채널을 고정하세요!


#


## Lessons Learned

* 무작위 입력을 생성하는 ("fuzzing")은 임의 프로그램의 견고성을 신속하게 테스트할 수 있는 간단하고 비용 효율적인 방법입니다.
* 버그 fuzzer가 발견한 버그는 주로 입력 처리의 오류와 결함 때문이다.
* 에러를 잡기 위해서 가능한 많은 일관성 검사기를 사용하십시오.


다 마쳤으니 사용한 파일을 제거합시다.

```python
os.remove(FILE)
os.removedirs(tempdir)
```

---

#

## Exercises

[Miller et al, 1990]에 트로프 조판 시스템에서 Miller가 발견한 오류중 하나 입니다. Troff는 라인으로 구성된 텍스트를 입력으로 사용합니다. 점(.)으로 시작하는 라인에는 타이핑 명령이 포함됩니다.

```python
.NH
Some Heading
.LP
Some paragraph
```

(nroff-ms를 사용하여) 텍스트를 생성합니다.

```python
1.  Some Heading

Some paragraph
```

Miller의 시점에서, troff의 입력이 포함되면 실패합니다.

1. 입력 시퀀스 **\D**(백슬래시 + D) 뒤에 인쇄할 수 없는 문자가 있습니다.
2. ASCII 128~255 범위의 문자(예: 8번째 비트 설정)와 줄 바꿈 문자
3. 점(**.**) 하나 뒤에 줄 바꿈 문자가 표시됩니다.

#


### Exercise 1: Simulate Troff

위의 각 항목에 대해 실패 기준을 충족하면 실패하는 Python 함수 f(s)를 작성하십시오.


#


### Exercise 2: Run Simulated Troff

위의 술어를 확인하는 **Runner**의 하위 클래스로 **TroffRunner** 클래스를 만드세요. Fuzzer를 실행시키세요. **Fuzzer** 객체가 전체 문자 범위를 생성하도록 하세요. 테스트 실패 빈도를 측정하세요.

#


### Exercise 3: Run Real Troff

**BinaryProgramRunner**를 사용하여 실제 **troff** 프로그램에 구성한 fuzzer를 적용하세요. 실패 또는 충돌을 나타내는 출력 코드가 0이 아닌 run을 생성할 수 있는지 확인하세요.
