---
title: "[The Fuzzing Book] - Fuzzing: Breaking Things with Random Inputs"
comments: true
categories:
- tech
tags:
- tech
- python
- testing
- fuzzing
last_modified_at: 2022-07-15T15:51+09:00
toc: true
---

# Fuzzing with Grammars
---

## 목차
* Synopsis
* Input Languages
* Grammars
  * Rules and Expansions
  * Arithmetic Expressions
* Repressenting Grammars in Python
* Some Definitions
* A Simple Grammar Fuzzer
* Visualling Grammars as Railroad Diagrams
* Some Grammars
  * A CGI Grammar
  * A URL Grammar
  * A Nature Language Grammar
* Grammars as Mutation Seeds
* A Grammar Toolbox
  * Escapes
  * Extendings Grammars
  * Character Classes
  * Grammar Shortcuts
  * Grammar Extensions
* Checking Grammars
* Lessions Learned
* Next Steps


["Mutation-Based Fuzzing"](https://www.fuzzingbook.org/html/MutationFuzzer.html) 장에서 테스트 생성의 속도 향상을 위해 간단한 입력 파일들을 만드는 등의 방법을 알아보았다. 이번 장에서는 프로그램의 합법적인 입력의 범위를 제공함으로써 해당 아이디어를 한 단계 발전시킬 것이다.

문법을 통한 입력 지정은 조직적이고 효과적인 테스트 생성을 허용해주고 특히,
복잡한 입력 형식에 도움이 된다. 또한, 문법은 퍼징 구성, API 퍼징, GUI 퍼징을 포함한 다양한 곳의 기반역할을 한다.

---

## Synopsis
---
이번 장에서 제공하는 아래 코드를 사용하여, 다음 기능들을 활용해라.

```python
>>> from fuzzingbook.Grammars import <identifier>
```

이 장에서는 입력 언어를 지정하고 구문적으로 유효한 입력이 있는 프로그램을 테스트하는 데 사용할 간단한 수단으로 **Grammar**를 소개하고 있다. 문법은 아래 예시와 같이 대체 확장 목록에 대한 비터미널 기호의 매핑으로 정의된다.

```python
>>> US_PHONE_GRAMMAR: Grammar = {
>>>     "<start>": ["<phone-number>"],
>>>     "<phone-number>": ["(<area>)<exchange>-<line>"],
>>>     "<area>": ["<lead-digit><digit><digit>"],
>>>     "<exchange>": ["<lead-digit><digit><digit>"],
>>>     "<line>": ["<digit><digit><digit><digit>"],
>>>     "<lead-digit>": ["2", "3", "4", "5", "6", "7", "8", "9"],
>>>     "<digit>": ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
>>> }
>>> 
>>> assert is_valid_grammar(US_PHONE_GRAMMAR)
```

비터미널 기호는 **&lt;digit&gt;** 와 같이 꺽쇄 기호로 둘러쌓여 있다. 문법에서 입력 문자열을을 만들어내기 위해 **producer**는 시작 기호인 **&lt;start&gt;**로 시작하게 되고, <strong style="color: red">해당 기호에 대한 확장을 무작위로 선택하게 된다. 이 과정은 비터미널 기호가 확장될 때까지 계속 진행된다. </strong>

**simple_grammar_fuzzer()** 함수는 아래와 같이 작동된다.

```python
>>> [simple_grammar_fuzzer(US_PHONE_GRAMMAR) for i in range(5)]
['(692)449-5179',
 '(519)230-7422',
 '(613)761-0853',
 '(979)881-3858',
 '(810)914-5475']
```

연습에서는 **simple_grammar_fuzzer()** 함수를 사용했지만, 실습에서는 **GrammarFuzzer** 클래스나 커버리지 기반, 확률 기반 또는 발전기 기반 파생 상품 중 하나를를 사용해야 한다. 앞서 말한 툴들이 더 효율적이고, 무한 루트를 막아주고, 몇가지 추가 기능들을 제공해준다.

이 장에서는 또한 문자 클래스와 반복에 단축키 표기법을 사용하거나 문법을 확장하는 것과 같은 문법을 쉽게 쓰는 몇 가지 도우미 기능이 있는 **grammar toolbox**를 소개한다.

------
## Input Language

프로그램의 모든 가능한 행동들은 그것의 입력에 의해 일어난다. 여기서 **"Input"** 은 파일이나 environment, 네트워크에서 얻은 데이터, 사용자 입력 데이터, 다른 자원과의 상호 작용에서 얻은 데이터와 같은 다양한 가능한 출처가 될 수 있다. 이런 모든 입력들은 프로그램의 에러를 비롯해서 프로그램이 어떻게 행동할지 결정한다. 그러므로 테스트를 할 때에는 가능한 입력 자원, 입력을 통제하는 방법, 그리고 체계적으로 테스트하는 방법을 생각하는게 도움이 된다.

단순함을 위해, 프로그램이 하나의 입력 소스만을 가지고 있다고 가정할 것이다. 프로그램에 유효한 입력 집합은 **_language_** 라고 불린다. **language** 에는 유효한 쉼표로 구분된 입력 집합을 나타내는 CSV 언어처럼 단순한 것에서 유효한 파이썬 프로그집합을 나타내는 Python처럼 복잡한 것까지 있다. 우리는 흔하게 데이터 언어와 프로그래언어를 구분하지만, 어떤 프로그램이라도 입력데이터에 의해 위협을 받을 수 있다. 파일 포맷에 대한 위키피디아 페이지는 1,000개 이상의 다른 파일 포맷을 나열하고 있는데, 각각의 확장자가 하나의 언어이다.

공식적으로 언어를 설명하기 위해, 공식 언어 분야는 언어를 설명하는 여러 언어 사양을 고안했다. **정규 표현식**은 문자열 세트를 나타내는 이러한 언어의 가장 간단한 클래스를 나타낸다. 예를 들어, 정규 표현식 [a-z]*는 소문자 시퀀스를 나타낸다. 오토마타 이론은 이러한 언어를 이러한 입력을 받아들이는 오토마타에 연결한다. 예를 들어, 유한 상태 기계를 사용하여 정규 표현식의 언어를 지정할 수 있다.

정규 표현식은 너무 복잡하지 않은 입력 형식에 적합하며, 관련 유한 상태 기계에는 추론에 적합한 많은 속성이 있다. 하지만 더 복잡한 입력을 지정하기 위해, 그들은 빠르게 한계에 직면한다. 튜링 기계는 계산할 수 있는 모든 것을 계산할 수 있다. Python이 Turing-complete이기 때문에 Python 프로그램 P를 사용하여 합법적인 입력을 지정하거나 열거할 수도 있다. 그러나 컴퓨터 과학 이론은 또한 우리가 원하는 자동화 수준이 아닌 프로그램을 테스트하기 위해 각 테스트 프로그램을 특별히 작성해야 한다고 말한다.

---
## Grammar

정규 표현식과 튜링 기계 사이의 중간 지점은 문법으로 덮여 있다. 문법은 입력 언어를 공식적으로 지정하는 가장 인기 있는 (그리고 가장 잘 이해되는) 형식주의 중 하나이다. 문법을 사용하여 입력 언어의 다양한 속성을 표현할 수 있다. 문법은 입력의 구문 구조를 표현하는 데 특히 적합하며, 중첩되거나 재귀 입력을 표현하기 위한 선택의 형식주의이다. 우리가 사용하는 문법은 가장 쉽고 인기 있는 문법 형식 중 하나인 소위 문맥 없는 문법이다.

### Rule And Expansion
---

문법은 시작 기호와 시작 기호(및 기타 기호)를 확장할 수 있는 방법을 나타내는 확장 규칙 집합(또는 단순히 규칙)으로 구성된다. 예를 들어, 두 자리의 시퀀스를 나타내는 다음 문법을 확인해보아라.

```
<start> ::= <digit><digit>
<digit> ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

이러한 문법을 읽으려면, 시작 기호(&lt;start&gt;)에서 부터 시작해야 된다. 확장 규칙 &lt;A&gt; ::= &lt;B&gt;는 왼쪽의 기호(&lt;A&gt;)가 오른쪽의 문자열(&lt;B&gt;)로 대체될 수 있음을 의미한다. 위의 문법에서 &lt;start&gt;는 &lt;digit&gt;&lt;digit&gt;로 대체된다.

위 코드에서 &lt;digit&gt;은 &lt;digit&gt; 규칙의 오른쪽에 있는 문자열로 대체될 것이다. 특수 연산자 **|** 는 확장 대안(또는 단순히 대안)을 나타내며, 이는 확장을 위해 숫자를 선택할 수 있음을 의미한다. 따라서 각 <자리>는 주어진 숫자 중 하나로 확장되어 결국 00에서 99 사이의 문자열을 산출할 것이다. 0에서 9까지의 추가 확장은 없으므로, 우리는 모두 준비되었다.

문법의 흥미로운 점은 그들이 재귀적일 수 있다는 것이다. 즉, 확장은 이전에 확장된 기호를 사용할 수 있으며, 이는 다시 확장될 것이다. 예를 들어, 정수를 설명하는 문법을 생각해보아라.

```
<start>  ::= <integer>
<integer> ::= <digit> | <digit><integer>
<digit>   ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

여기서 <정수>는 한 자리이거나 다른 정수가 뒤따르는 숫자이다. 따라서 숫자 1234는 단일 숫자 1로 표현되고, 정수 234가 뒤따르며, 차례로 숫자 2이고, 정수 34가 뒤따를 것이다.

정수 앞에 기호 (+ 또는 -)가 있을 수 있다는 것을 표현하고 싶다면, 우리는 문법을 다음과 같이 쓸 것이다.

```
<start>   ::= <number>
<number>  ::= <integer> | +<integer> | -<integer>
<integer> ::= <digit> | <digit><integer>
<digit>   ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

이 규칙은 공식적으로 언어를 정의한다. 시작 기호에서 파생될 수 있는 모든 것은 언어의 일부이고, 할 수 없는 것은 언어가 아니다.

### Arithmetic Expressions

<strong style="color: red";>문법의 포스터 자식 예시인</strong> 전체 산술 표현을 다루기 위해 문법을 확장하자. 표현식(&lt;expr&gt;)이 합계 또는 차이 또는 용어이며, 용어는 곱이나 나눗셈 또는 요인이며, 요인은 숫자 또는 괄호 표현식이라는 것을 알 수 있다. 거의 모든 규칙은 재귀를 가질 수 있으므로 (1 + 2) * (3.4 / 5.6 - 789)와 같은 임의의 복잡한 표현을 허용한다.

```
<start>   ::= <expr>
<expr>    ::= <term> + <expr> | <term> - <expr> | <term>
<term>    ::= <term> * <factor> | <term> / <factor> | <factor>
<factor>  ::= +<factor> | -<factor> | (<expr>) | <integer> | <integer>.<integer>
<integer> ::= <digit><integer> | <digit>
<digit>   ::= 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

이러한 문법에서 &lt;start&gt;로 시작한 다음 한 기호를 차례로 확장하여 무작위로 대안을 선택하면 유효한 산술 표현식을 차례로 빠르게 생성할 수 있다. 이러한 문법 퍼징은 복잡한 입력을 생성할 때 매우 효과적이며, 이것이 우리가 이 장에서 구현할 것이다.

## Representing Grammars in Python

문법 퍼저를 만드는 첫 번째 단계는 문법에 적합한 형식을 찾는 것이다. 문법의 쓰기를 가능한 한 간단하게 만들기 위해, 문자열과 목록을 기반으로 하는 형식을 사용한다. 파이썬의 문법은 확장이 대안 목록인 기호 이름과 확장 사이의 매핑 형식을 취하고 있다. 따라서 숫자에 대한 한 규칙 문법이 형태를 취한다.

```python
DIGIT_GRAMMAR = {
    "<start>":
        ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
}
```

> ### A "Grammar" Type
> 문법 유형을 정적으로 확인할 수 있도록 문법 유형을 정의하자.
> 문법 유형에 대한 첫 번째 시도는 각 기호(문자열)가 확장(문자열) 리스트에 매핑되는 것이다.
> ```python
> SimpleGrammar = Dict[str, List[str]]
> ```
> 그러나 이 장의 뒷부분에서 소개할 선택적 속성을 추가하기 위한 opts() 기능은 확장이 문자열과 옵션으로 구성된 쌍이 될 수 있도록 하며, 옵션은 문자열을 값으로 매핑한다.
> ```python
> Option = Dict[str, Any]
> ```
> 따라서, 확장은 문자열이거나 한 쌍의 문자열과 옵션이다.
>```python
>Expansion = Union[str, Tuple[str, Option]]
>```
>이를 통해 이제 문법을 확장 리스트에 대한 문자열 매핑으로 정의할 수 있다.

각 기호(문자열)가 확장(문자열) 리스트에 매핑되는 문법 형식으로 문법 구조를 캡처할 수 있다.

```python
Grammar = Dict[str, List[Expansion]]
```

이 문법 유형으로, 산술 표현의 전체 문법은 다음과 같다.

```python
EXPR_GRAMMAR: Grammar = {
    "<start>":
        ["<expr>"],

    "<expr>":
        ["<term> + <expr>", "<term> - <expr>", "<term>"],

    "<term>":
        ["<factor> * <term>", "<factor> / <term>", "<factor>"],

    "<factor>":
        ["+<factor>",
         "-<factor>",
         "(<expr>)",
         "<integer>.<integer>",
         "<integer>"],

    "<integer>":
        ["<digit><integer>", "<digit>"],

    "<digit>":
        ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]
}
```

문법에서, 모든 기호는 정확히 한 번 정의될 수 있다. 그 기호를 사용해서 어떤 규칙에도 접근할 수 있고, 기호가 문법에 있는지도 확인할 수 있다.

```python
>>> EXPR_GRAMMAR["<digit>"]
['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
>>> "<identifier>" in EXPR_GRAMMAR
False
```

규칙의 왼쪽(즉, 매핑의 키)은 항상 단일 기호라고 가정한다. 이것은 문법에서 문맥 없는 특성을 제공하는 속성이다.

---
## Some Definitions

우리는 표준 시작 기호가 &lt;start&gt;라고 가정한다.
```
START_SYMBOL = "<start>"
```

편리한 **nonterminals()** 함수는 확장에서 비터미널 기호 목록(즉, 공백을 제외한 <와 > 사이의 모든 것)을 추출한다.

```python
>>> import re
>>> RE_NONTERMINAL = re.compile(r'(<[^<> ]*>)')
>>> def nonterminals(expansion):
	    # In later chapters, we allow expansions to be tuples,
	    # with the expansion being the first element
	    if isinstance(expansion, tuple):
	        expansion = expansion[0]
	    return RE_NONTERMINAL.findall(expansion)
	    
>>> assert nonterminals("<term> * <factor>") == ["<term>", "<factor>"]
>>> assert nonterminals("<digit><integer>") == ["<digit>", "<integer>"]
>>> assert nonterminals("1 < 3 > 2") == []
>>> assert nonterminals("1 <3> 2") == ["<3>"]
>>> assert nonterminals("1 + 2") == []
>>> assert nonterminals(("<1>", {'option': 'value'})) == ["<1>"]
>>>
```

마찬가지로, **is_nonterminal()** 함수는 일부 기호가 비터미널인지 확인한다.

```python
>>> def is_nonterminal(s):
    return RE_NONTERMINAL.match(s)
    
>>> assert is_nonterminal("<abc>")
>>> assert is_nonterminal("<symbol-1>")
>>> assert not is_nonterminal("+")
>>>>
```

---
## A Simple Grammar Fuzzer

이제 위의 문법을 사용하자. 처음에시작 기호(&lt;start&gt;)로 시작한 다음 계속 확장하는 매우 간단한 문법 퍼저를 만들 것이다. 무한한 입력으로의 확장을 피하기 위해, 비터미널의 수에 한계(**max_nonterminals**)를 배치한다. 게다가, 기호 수를 더 이상 줄일 수 없는 상황에 갇히지 않기 위해, 또한 확장 단계의 총 수를 제한한다.

```python
import random

class ExpansionError(Exception):
    pass

def simple_grammar_fuzzer(grammar: Grammar, 
                          start_symbol: str = START_SYMBOL,
                          max_nonterminals: int = 10,
                          max_expansion_trials: int = 100,
                          log: bool = False) -> str:
    """Produce a string from `grammar`.
 `start_symbol`: use a start symbol other than `<start>` (default).
 `max_nonterminals`: the maximum number of nonterminals 
 still left for expansion
 `max_expansion_trials`: maximum # of attempts to produce a string
 `log`: print expansion progress if True"""

    term = start_symbol
    expansion_trials = 0

    while len(nonterminals(term)) > 0:
        symbol_to_expand = random.choice(nonterminals(term))
        expansions = grammar[symbol_to_expand]
        expansion = random.choice(expansions)
        # In later chapters, we allow expansions to be tuples,
        # with the expansion being the first element
        if isinstance(expansion, tuple):
            expansion = expansion[0]

        new_term = term.replace(symbol_to_expand, expansion, 1)

        if len(nonterminals(new_term)) < max_nonterminals:
            term = new_term
            if log:
                print("%-40s" % (symbol_to_expand + " -> " + expansion), term)
            expansion_trials = 0
        else:
            expansion_trials += 1
            if expansion_trials >= max_expansion_trials:
                raise ExpansionError("Cannot expand " + repr(term))

    return term
```

이 간단한 문법 퍼저가 시작 기호에서 산술 표현을 어떻게 얻는지 확인해 보자.

```python
>>> simple_grammar_fuzzer(grammar=EXPR_GRAMMAR, max_nonterminals=3, log=True)
<start> -> <expr>                        <expr>
<expr> -> <term> + <expr>                <term> + <expr>
<term> -> <factor>                       <factor> + <expr>
<factor> -> <integer>                    <integer> + <expr>
<integer> -> <digit>                     <digit> + <expr>
<digit> -> 6                             6 + <expr>
<expr> -> <term> - <expr>                6 + <term> - <expr>
<expr> -> <term>                         6 + <term> - <term>
<term> -> <factor>                       6 + <factor> - <term>
<factor> -> -<factor>                    6 + -<factor> - <term>
<term> -> <factor>                       6 + -<factor> - <factor>
<factor> -> (<expr>)                     6 + -(<expr>) - <factor>
<factor> -> (<expr>)                     6 + -(<expr>) - (<expr>)
<expr> -> <term>                         6 + -(<term>) - (<expr>)
<expr> -> <term>                         6 + -(<term>) - (<term>)
<term> -> <factor>                       6 + -(<factor>) - (<term>)
<factor> -> +<factor>                    6 + -(+<factor>) - (<term>)
<factor> -> +<factor>                    6 + -(++<factor>) - (<term>)
<term> -> <factor>                       6 + -(++<factor>) - (<factor>)
<factor> -> (<expr>)                     6 + -(++(<expr>)) - (<factor>)
<factor> -> <integer>                    6 + -(++(<expr>)) - (<integer>)
<expr> -> <term>                         6 + -(++(<term>)) - (<integer>)
<integer> -> <digit>                     6 + -(++(<term>)) - (<digit>)
<digit> -> 9                             6 + -(++(<term>)) - (9)
<term> -> <factor> * <term>              6 + -(++(<factor> * <term>)) - (9)
<term> -> <factor>                       6 + -(++(<factor> * <factor>)) - (9)
<factor> -> <integer>                    6 + -(++(<integer> * <factor>)) - (9)
<integer> -> <digit>                     6 + -(++(<digit> * <factor>)) - (9)
<digit> -> 2                             6 + -(++(2 * <factor>)) - (9)
<factor> -> +<factor>                    6 + -(++(2 * +<factor>)) - (9)
<factor> -> -<factor>                    6 + -(++(2 * +-<factor>)) - (9)
<factor> -> -<factor>                    6 + -(++(2 * +--<factor>)) - (9)
<factor> -> -<factor>                    6 + -(++(2 * +---<factor>)) - (9)
<factor> -> -<factor>                    6 + -(++(2 * +----<factor>)) - (9)
<factor> -> <integer>.<integer>          6 + -(++(2 * +----<integer>.<integer>)) - (9)
<integer> -> <digit>                     6 + -(++(2 * +----<digit>.<integer>)) - (9)
<integer> -> <digit>                     6 + -(++(2 * +----<digit>.<digit>)) - (9)
<digit> -> 1                             6 + -(++(2 * +----1.<digit>)) - (9)
<digit> -> 7                             6 + -(++(2 * +----1.7)) - (9)

'6 + -(++(2 * +----1.7)) - (9)'
```

비터미널의 한계를 늘림으로써, 우리는 훨씬 더 긴 생산을 빠르게 얻을 수 있다.
```python
>>> for i in range(10):
		print(simple_grammar_fuzzer(grammar=EXPR_GRAMMAR,max_nonterminals=5))

7 / +48.5
-5.9 / 9 - 4 * +-(-+++((1 + (+7 - (-1 * (++-+7.7 - -+-4.0))))) * +--4 - -(6) + 64)
8.2 - 27 - -9 / +((+9 * --2 + --+-+-((-1 * +(8 - 5 - 6)) * (-((-+(((+(4))))) - ++4) / +(-+---((5.6 - --(3 * -1.8 * +(6 * +-(((-(-6) * ---+6)) / +--(+-+-7 * (-0 * (+(((((2)) + 8 - 3 - ++9.0 + ---(--+7 / (1 / +++6.37) + (1) / 482) / +++-+0)))) * -+5 + 7.513)))) - (+1 / ++((-84)))))))) * ++5 / +-(--2 - -++-9.0)))) / 5 * --++090
1 - -3 * 7 - 28 / 9
(+9) * +-5 * ++-926.2 - (+9.03 / -+(-(-6) / 2 * +(-+--(8) / -(+1.0) - 5 + 4)) * 3.5)
8 + -(9.6 - 3 - -+-4 * +77)
-(((((++((((+((++++-((+-37))))))))))))) / ++(-(+++(+6)) * -++-(+(++(---6 * (((7)) * (1) / (-7.6 * 535338) + +256) * 0) * 0))) - 4 + +1
5.43
(9 / -405 / -23 - +-((+-(2 * (13))))) + +6 - +8 - 934
-++2 - (--+715769550) / 8 / (1)
```

퍼저가 대부분의 경우에는 잘 작동하지만, 많은 단점이 있다는 점에 유의해라.

실제로, **simple_grammar_fuzzer()**는 많은 수의 검색 및 대체 작업으로 인해 다소 비효율적이며 문자열을 생성하지 못할 수도 있다. 반면에, 구현은 간단하며 대부분의 경우 작업을 수행한다. 이 장을 위해, 우리는 그것을 고수할 것이다. 다음 장에서는 더 효율적인 장을 만드는 방법을 보여줄 것이다.

---
## Visualizing Grammars as Railroad Diagrams

문법을 사용하면 이전에 논의한 몇 가지 예제의 형식을 쉽게 지정할 수 있다. 예를 들어, 위의 산술 표현식은 bc(또는 산술 표현식을 취하는 다른 프로그램)로 직접 보낼 수 있다. 몇 가지 문법을 추가로 소개하기 전에, 이것들을 시각화할 수 있는 수단을 제공하여 그들의 이해를 돕기 위한 대체 관점을 제공하자.

구문 다이어그램이라고도 불리는 철도 다이어그램은 문맥이 없는 문법의 그래픽 표현이다. 그들은 가능한 **"레일"** 트랙을 따라 왼쪽에서 오른쪽으로 읽힌다. 트랙에서 발생하는 일련의 기호는 언어를 정의한다. 철도 다이어그램을 생성하기 위해, **syntax_diagram()** 함수를 구현한다.

> ### Syntax_diagram() 구현
> 우리는 시각화를 위한 외부 라이브러리인 **RailroadDiagrams**를 사용한다.
> ```python
> from RailroadDiagrams import NonTerminal, Terminal, Choice, HorizontalChoice, Sequence
> from RailroadDiagrams import show_diagram
> from IPython.display import SVG
> ```
> 먼저 주어진 기호를 시각화하기 위해 **syntax_diagram_symbol()** 메서드를 정의한다. 터미널 기호는 타원형으로 표시되지만, 비터미널 기호(예: <term>)는 직사각형으로 표시됩니다.
> ```python
> def syntax_diagram_symbol(symbol: str) -> Any:
	> 	if is_nonterminal(symbol):
	> 		return NonTerminal(symbol[1:-1])
    >	else:
    > 		return Terminal(symbol)
> SVG(show_diagram(syntax_diagram_symbol('<term>')))
> ```
> ![fuzzer3-1](../src/fuzzer3_1.png)
> 확장 대안을 시각화하기 위해 **syntax_diagram_expr()** 함수를 정의한다.
> ```python
> def syntax_diagram_expr(expansion: Expansion) -> Any:
	> 	# In later chapters, we allow expansions to be tuples,
	> 	# with the expansion being the first element
    > 	if isinstance(expansion, tuple):
    >		expansion = expansion[0]
    > 	symbols = [sym for sym in re.split(RE_NONTERMINAL, expansion) if sym != ""]
    > 	if len(symbols) == 0:
    >		symbols = [""]  # special case: empty expansion
    >	return Sequence(*[syntax_diagram_symbol(sym) for sym in symbols])
>
> SVG(show_diagram(syntax_diagram_expr(EXPR_GRAMMAR['<term>'][0])))
> ```
> ![fuzzer3-2](../src/fuzzer3_2.png)
> <strong style="color: red";>This is the first alternative of `<term>` – a `<factor>` followed by `*` and a `<term>`.</strong>
> 다음으로, 대체 식을 표시하기 위해 **syntax_diagram_alt()** 를 정의한다.
>
> ```python
> from itertools import zip_longest
> 
> def syntax_diagram_alt(alt: List[Expansion]) -> Any:
	> 	max_len = 5
    >	alt_len = len(alt)
    >	if alt_len > max_len:
	> 		iter_len = alt_len // max_len
    >		alts = list(zip_longest(*[alt[i::iter_len] for i in range(iter_len)]))
    >		exprs = [[syntax_diagram_expr(expr) for expr in alt
    >			if expr is not None] for alt in alts]
    > 		choices = [Choice(len(expr) // 2, *expr) for expr in exprs]
    > 		return HorizontalChoice(*choices)
    >	else:
    >		return Choice(alt_len // 2, *[syntax_diagram_expr(expr) for expr in alt])
>
> SVG(show_diagram(syntax_diagram_alt(EXPR_GRAMMAR['<digit>'])))
> ```
> ![fuzzer3-3](../src/fuzzer3_3.png)
> 
> 우리는 &lt;digit&gt;가 0에서 9까지의 단일 자릿수일 수 있다는 것을 볼 수 있다.
> 마지막으로, 문법이 주어진 **syntax_diagram()** 함수를 정의하고 규칙의 구문 다이어그램을 표시한다.
>
> ```python
> def syntax_diagram(grammar: Grammar) -> None:
	> 	from import SVG, display
>
	> 	for key in grammar:
    > 		print("%s" % key[1:-1])
    >		display(SVG(show_diagram(syntax_diagram_alt(grammar[key]))))
> ```

**syntax_diagram()** 함수를 사용하여 표현 문법의 철도 다이어그램을 생성하자.

```
syntax_diagram(EXPR_GRAMMAR)
```

![fuzzer3-4-1](../src/fuzzer3_4_1.png)
![fuzzer3-4-2](../src/fuzzer3_4_2.png)

이 철도 표현은 특히 더 복잡한 문법의 구조를 시각화하는 데 유용할 것이다.

---
## Some Grammars

더 많은 문법을 만들고 시각화하여 퍼징에 사용하자.


### A CGI Grammar
---

다음은 [coverage](https://www.fuzzingbook.org/html/Coverage.html) 장에 소개된 **cgi_decode()** 의 문법이다.

```python
CGI_GRAMMAR: Grammar = {
    "<start>":
        ["<string>"],

    "<string>":
        ["<letter>", "<letter><string>"],

    "<letter>":
        ["<plus>", "<percent>", "<other>"],

    "<plus>":
        ["+"],

    "<percent>":
        ["%<hexdigit><hexdigit>"],

    "<hexdigit>":
        ["0", "1", "2", "3", "4", "5", "6", "7",
            "8", "9", "a", "b", "c", "d", "e", "f"],

    "<other>":  # Actually, could be _all_ letters
        ["0", "1", "2", "3", "4", "5", "a", "b", "c", "d", "e", "-", "_"],
}

syntax_diagram(CGI_GRAMMAR)
```

![fuzzer3-5-1](../src/fuzzer3_5_1.png)
![fuzzer3-5-2](../src/fuzzer3_5_2.png)

**[basic fuzzing](https://www.fuzzingbook.org/html/Fuzzer.html)** 또는 **[mutation-based fuzzing](https://www.fuzzingbook.org/html/MutationFuzzer.html)** 과 달리, 문법은 모든 종류의 조합을 빠르게 생성한다.

```python
>>> for i in range(10):
	    print(simple_grammar_fuzzer(grammar=CGI_GRAMMAR, max_nonterminals=10))

+%9a
+++%ce+
+_
+%c6c
++
+%cd+5
1%ee
%b9%d5
%96
%57d%42
```

### A URL Grammar
---

우리가 CGI 입력에 대해 본 것과 동일한 속성은 더 복잡한 입력을 보유하고 있다. 문법을 사용하여 많은 수의 유효한 URL을 생성해보자.

```python
URL_GRAMMAR: Grammar = {
    "<start>":
        ["<url>"],
    "<url>":
        ["<scheme>://<authority><path><query>"],
    "<scheme>":
        ["http", "https", "ftp", "ftps"],
    "<authority>":
        ["<host>", "<host>:<port>", "<userinfo>@<host>", "<userinfo>@<host>:<port>"],
    "<host>":  # Just a few
        ["cispa.saarland", "www.google.com", "fuzzingbook.com"],
    "<port>":
        ["80", "8080", "<nat>"],
    "<nat>":
        ["<digit>", "<digit><digit>"],
    "<digit>":
        ["0", "1", "2", "3", "4", "5", "6", "7", "8", "9"],
    "<userinfo>":  # Just one
        ["user:password"],
    "<path>":  # Just a few
        ["", "/", "/<id>"],
    "<id>":  # Just a few
        ["abc", "def", "x<digit><digit>"],
    "<query>":
        ["", "?<params>"],
    "<params>":
        ["<param>", "<param>&<params>"],
    "<param>":  # Just a few
        ["<id>=<id>", "<id>=<nat>"],
}

syntax_diagram(URL_GRAMMAR)
```

![fuzzer3-6-1](../src/fuzzer3_6_1.png)
![fuzzer3-6-2](../src/fuzzer3_6_2.png)
![fuzzer3-6-3](../src/fuzzer3_6_3.png)

다시 말하지만, 밀리초 이내에, 우리는 많은 유효한 입력을 생산할 수 있다.

```python
for i in range(10):
    print(simple_grammar_fuzzer(grammar=URL_GRAMMAR, max_nonterminals=10))

https://user:password@cispa.saarland:80/
http://fuzzingbook.com?def=56&x89=3&x46=48&def=def
ftp://cispa.saarland/?x71=5&x35=90&def=abc
https://cispa.saarland:80/def?def=7&x23=abc
https://fuzzingbook.com:80/
https://fuzzingbook.com:80/abc?def=abc&abc=x14&def=abc&abc=2&def=38
ftps://fuzzingbook.com/x87
https://user:password@fuzzingbook.com:6?def=54&x44=abc
http://fuzzingbook.com:80?x33=25&def=8
http://fuzzingbook.com:8080/def
```

### A Natural Language Grammar
---

마지막으로, 문법은 컴퓨터 입력과 같은 공식 언어에만 국한되지 않고 자연어를 생산하는 데에도 사용될 수 있다. 이것은 우리가 이 책의 제목을 고르는 데 사용한 문법이다.

```python
TITLE_GRAMMAR: Grammar = {
    "<start>": ["<title>"],
    "<title>": ["<topic>: <subtopic>"],
    "<topic>": ["Generating Software Tests", "<fuzzing-prefix>Fuzzing", "The Fuzzing Book"],
    "<fuzzing-prefix>": ["", "The Art of ", "The Joy of "],
    "<subtopic>": ["<subtopic-main>",
                   "<subtopic-prefix><subtopic-main>",
                   "<subtopic-main><subtopic-suffix>"],
    "<subtopic-main>": ["Breaking Software",
                        "Generating Software Tests",
                        "Principles, Techniques and Tools"],
    "<subtopic-prefix>": ["", "Tools and Techniques for "],
    "<subtopic-suffix>": [" for <reader-property> and <reader-property>",
                          " for <software-property> and <software-property>"],
    "<reader-property>": ["Fun", "Profit"],
    "<software-property>": ["Robustness", "Reliability", "Security"],
}

syntax_diagram(TITLE_GRAMMAR)
```

![fuzzer3-7-1](../src/fuzzer3_7_1.png)
![fuzzer3-7-2](../src/fuzzer3_7_2.png)

```python
>>> from [typing](https://docs.python.org/3/library/typing.html) import Set
>>> titles: Set[str] = set()
>>> while len(titles) < 10:
	    titles.add(simple_grammar_fuzzer(
	        grammar=TITLE_GRAMMAR, max_nonterminals=10))
>>> titles
{'Fuzzing: Generating Software Tests',
 'Fuzzing: Principles, Techniques and Tools',
 'Generating Software Tests: Breaking Software',
 'Generating Software Tests: Breaking Software for Robustness and Robustness',
 'Generating Software Tests: Principles, Techniques and Tools',
 'Generating Software Tests: Principles, Techniques and Tools for Profit and Fun',
 'Generating Software Tests: Tools and Techniques for Principles, Techniques and Tools',
 'The Fuzzing Book: Breaking Software',
 'The Fuzzing Book: Generating Software Tests for Profit and Profit',
 'The Fuzzing Book: Generating Software Tests for Robustness and Robustness'}
```

---
## Grammars as Mutation Seeds

문법의 매우 유용한 속성 중 하나는 대부분 유효한 입력을 생성한다는 것이다. 구문적 관점에서, 입력은 주어진 문법의 제약을 충족시키기 때문에 실제로 항상 유효하다. (물론, 애초에 유효한 문법이 필요하다.) 그러나, 문법으로 쉽게 표현할 수 없는 의미론적 특성도 있다. 예를 들어, URL의 경우, 포트 범위가 1024에서 2048 사이여야 한다면, 이것은 문법으로 쓰기 어렵다. 더 복잡한 제약을 충족시켜야 한다면, 문법이 표현할 수 있는 한계에 빠르게 도달한다.

이 문제를 해결할 한 가지 방법은 이 책의 뒷부분에서 논의할 문법에 제약을 붙이는 것이다. 또 다른 가능성은 문법 기반 퍼징과 돌연변이 기반 퍼징의 강점을 결합하는 것이다. 그 아이디어는 문법으로 생성된 입력을 추가 돌연변이 기반 퍼징을 위한 seed로 사용하는 것이다. 이렇게 하면 유효한 입력뿐만 아니라 유효한 입력과 유효하지 않은 입력 사이의 경계를 확인할 수 있다. 이것은 약간 유효하지 않은 입력이 파서 오류를 찾을 수 있기 때문에 특히 흥미롭다. 일반적으로 퍼징과 마찬가지로, 프로그램의 오류를 드러내는 것은 예상치 못한 일이다.

생성된 입력을 seed로 사용하려면, 이전에 도입된 돌연변이 퍼저에 직접 공급할 수 있다.

```python
>>> from MutationFuzzer import MutationFuzzer  # minor dependency
>>> number_of_seeds = 10
>>> seeds = [
	    simple_grammar_fuzzer(
        grammar=URL_GRAMMAR,
        max_nonterminals=10) for i in range(number_of_seeds)]
>>> seeds
['ftps://user:password@www.google.com:80',
 'http://cispa.saarland/',
 'ftp://www.google.com:42/',
 'ftps://user:password@fuzzingbook.com:39?abc=abc',
 'https://www.google.com?x33=1&x06=1',
 'http://www.google.com:02/',
 'https://user:password@www.google.com/',
 'ftp://cispa.saarland:8080/?abc=abc&def=def&abc=5',
 'http://www.google.com:80/def?def=abc',
 'http://user:password@cispa.saarland/']
>>> m = MutationFuzzer(seeds)
>>> [m.fuzz() for i in range(20)]
['ftps://user:password@www.google.com:80',
 'http://cispa.saarland/',
 'ftp://www.google.com:42/',
 'ftps://user:password@fuzzingbook.com:39?abc=abc',
 'https://www.google.com?x33=1&x06=1',
 'http://www.google.com:02/',
 'https://user:password@www.google.com/',
 'ftp://cispa.saarland:8080/?abc=abc&def=def&abc=5',
 'http://www.google.com:80/def?def=abc',
 'http://user:password@cispa.saarland/',
 'Eh4tp:www.coogle.com:80/def?d%f=abc',
 'ftps://}ser:passwod@fuzzingbook.com:9?abc=abc',
 'uftp//cispa.sRaarland:808&0?abc=abc&def=defabc=5',
 'http://user:paswor9d@cispar.saarland/v',
 'ftp://Www.g\x7fogle.cAom:42/',
 'hht://userC:qassMword@cispy.csaarland/',
 'httx://ww.googlecom:80defde`f=ac',
 'htt://cispq.waarlnd/',
 'htFtp\t://cmspa./saarna(md/',
 'ft:/www.google.com:42\x0f']
```

처음 10개의 **fuzz()** 호출은 시드된 입력을 반환하지만(설계된 대로), 나중의 호출은 다시 임의의 돌연변이를 생성한다. **MutationFuzzer** 대신 **MutationCoverageFuzzer** 를 사용하여, <strong style="color: red";>다시 범위별로 검색을 안내할 수 있으므로 여러 세계의 최고를 하나로 모을 수 있다.</strong>

---
## A Grammar Toolbox

이제 문법을 작성하는 데 도움이 되는 몇 가지 기술을 알아보자.

### Escapes
---

< and >가 문법에서 비터미널을 구분하면서, 일부 입력이 < and >를 포함해야 한다는 것을 어떻게 표현할 수 있을까? 답은 간단하다. 문법에 기호를 명시하면 된다.

```python
simple_nonterminal_grammar: Grammar = {
    "<start>": ["<nonterminal>"],
    "<nonterminal>": ["<left-angle><identifier><right-angle>"],
    "<left-angle>": ["<"],
    "<right-angle>": [">"],
    "<identifier>": ["id"]  # for now
}
```

**simple_nonterminal_grammar**에서 &lt;left-angle&gt;의 확장이나 &lt;right-angle&gt;의 확장은 비터미널로 오인될 수 없다. 따라서, 우리는 우리가 원하는 만큼 생산할 수 있다.

### Extending Grammars
---

이 책의 과정에서, 우리는 종종 새로운 기능으로 기존 문법을 확장하여 문법을 만드는 문제를 마주친다. 이러한 확장은 객체 지향 프로그래밍의 서브클래스와 매우 비슷하다.

기존 문법 g에서 새로운 문법 g'를 생성하려면 먼저 g를 g'에 복사한 다음 새로운 대안으로 기존 규칙을 확장하거나 새 기호를 추가한다. 다음은 식별자에 대한 더 나은 규칙으로 위의 **nonterminal** 문법을 확장한 예시이다.

```python
>>> import copy
>>> nonterminal_grammar = copy.deepcopy(simple_nonterminal_grammar)
>>> nonterminal_grammar["<identifier>"] = ["<idchar>", "<identifier><idchar>"]
>>> nonterminal_grammar["<idchar>"] = ['a', 'b', 'c', 'd']  # for now
>>> nonterminal_grammar
{'<start>': ['<nonterminal>'],
 '<nonterminal>': ['<left-angle><identifier><right-angle>'],
 '<left-angle>': ['<'],
 '<right-angle>': ['>'],
 '<identifier>': ['<idchar>', '<identifier><idchar>'],
 '<idchar>': ['a', 'b', 'c', 'd']}
```

이러한 문법 확장은 일반적인 작업이기 때문에, 먼저 주어진 문법을 복사한 다음 파이썬 딕셔너리의 **update()** 메서드를 사용하여 딕셔너리에서 업데이트하는 사용자 지정 함수인 ** extend_grammar()** 를 소개합니다.

```python
def extend_grammar(grammar: Grammar, extension: Grammar = {}) -> Grammar:
    new_grammar = copy.deepcopy(grammar)
    new_grammar.update(extension)
    return new_grammar
```

**extend_grammar()** 에 대한 이 호출은 위의 **"manual"** 예와 마찬가지로 **simple_nonterminal_grammar**를 **nonterminal_grammar**로 확장한다.

```python
nonterminal_grammar = extend_grammar(simple_nonterminal_grammar,
                                     {
                                         "<identifier>": ["<idchar>", "<identifier><idchar>"],
                                         # for now
                                         "<idchar>": ['a', 'b', 'c', 'd']
                                     }
                                     )
```

### Character Classes
---

위의 **nonterminal_grammar** 에서는 처음 몇 글자만 열거했지만, 실제로 **&lt;idchar&gt; ::= 'a' | 'b' | 'c' ...** 와 같이 문법에서 모든 문자나 숫자를 수동으로 열거하는 것은 조금 고통스럽다.

그러나, 문법은 프로그램의 일부이며, 따라서 프로그래밍 방식으로 구성될 수 있다는 것을 기억하자. 문자열의 문자 목록을 구성하는 함수 **srange()** 를 사용해보자.

```python
import [string](https://docs.python.org/3/library/string.html)

def srange(characters: str) -> List[Expansion]:
    """Construct a list with all characters in the string"""
    return [c for c in characters]
```

모든 ASCII 문자를 보유하고 있는 상수 **string.ascii_letters** 를 전달하면, **srange()** 는 모든 ASCII 문자 목록을 반환한다.

```python
>>> string.ascii_letters
'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
>>> srange(string.ascii_letters)[:10]
['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j']
```

문법에서 이러한 상수를 사용하여 식별자를 빠르게 정의할 수 있다.

```python
>>> nonterminal_grammar = extend_grammar(nonterminal_grammar,
	    {
	        "<idchar>": (srange(string.ascii_letters) + 
	                        srange(string.digits) + 
	                        srange("-_"))
        }
    )

>>> [simple_grammar_fuzzer(nonterminal_grammar, "<identifier>") for i in range(10)]
['b', 'd', 'V9', 'x4c', 'YdiEWj', 'c', 'xd', '7', 'vIU', 'QhKD']
```

단축키 **crange(start, end)** 는 ASCII **start** 에서 **end** 까지의 모든 문자 목록을 반환한다.

```python
def crange(character_start: str, character_end: str) -> List[Expansion]:
    return [chr(i) for i in range(ord(character_start), ord(character_end) + 1)]
```

우리는 이것을 사용하여 문자 범위를 표현할 수 있다.

```python
>>> crange('0', '9')
['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']
>>> assert crange('a', 'z') == srange(string.ascii_lowercase)
>>>
```

### Grammar Shortcuts
---

위의 **nonterminal_grammar** 에서, 다른 문법과 마찬가지로, 우리는 재귀, 즉 원래 정의를 참조하여 문자의 반복을 표현해야 한다.

```python
>>> nonterminal_grammar["<identifier>"]
['<idchar>', '<identifier><idchar>']
```

예를 들어, 단순히 **nonterminal** 이 비어있지 않은 문자 시퀀스여야 한다고 말할 수 있다면 조금 더 쉬울 수 있다.

```
<identifier> = <idchar>+
```

여기서 **+** 는 따르는 기호의 비어 있지 않은 반복을 나타낸다.

**+** 와 같은 연산자는 종종 문법에서 편리한 단축키로 소개된다. 공식적으로, 우리의 문법은 소위 **Backus-Naur form** 또는 간단히 말해서 **BNF** 에서 비롯되었다. 연산자는 **BNF** 를 소위 **_extended BNF** 또는 줄여서 **EBNF*** 로 확장합니다:

* **&lt;symbol&gt;?** 형태는 **&lt;symbol&gt;** 가 선택 사항임을 나타낸다. 즉, 0번 또는 1번 발생할 수 있다.
* **&lt;symbol&gt;+** 형태는 **&lt;symbol&gt;** 이 반복적으로 1번 이상 발생할 수 있음을 나타낸다.
* **&lt;symbol&gt;*** 형태는 **&lt;symbol&gt;** 이 0번 이상 발생할 수 있음을 나타낸다. (즉, 선택적 반복이다.)


문제를 더욱 흥미롭게 만들기 위해, 위의 단축키와 함께 괄호를 사용할 것이다. 그러므로 **(&lt;foo&gt;&lt;bar&gt;)?** 는 &lt;foo&gt;와 &lt;bar&gt;의 시퀀스가 선택 사항임을 나타낸다.

이러한 연산자를 사용하여 식별자 규칙을 더 간단하게 정의할 수 있다. 이를 위해 원본 문법의 복사본을 만들고 &lt;identifier&gt; 규칙을 수정하자.

```python
nonterminal_ebnf_grammar = extend_grammar(nonterminal_grammar,
	{
	    "<identifier>": ["<idchar>+"]
    }
)
```

마찬가지로, 우리는 표현 문법을 단순화할 수 있다. 기호가 어떻게 선택 사항인지, 그리고 정수가 숫자의 시퀀스로 어떻게 표현될 수 있는지 고려해라.

```python
EXPR_EBNF_GRAMMAR: Grammar = {
    "<start>":
        ["<expr>"],

    "<expr>":
        ["<term> + <expr>", "<term> - <expr>", "<term>"],

    "<term>":
        ["<factor> * <term>", "<factor> / <term>", "<factor>"],

    "<factor>":
        ["<sign>?<factor>", "(<expr>)", "<integer>(.<integer>)?"],

    "<sign>":
        ["+", "-"],

    "<integer>":
        ["<digit>+"],

    "<digit>":
        srange(string.digits)
}
```

이러한 **EBNF** 문법을 사용하여 자동으로 **BNF** 문법으로 변환하는 함수 **convert_ebnf_grammar()** 를 구현하자.

> ### convert_ebnf_grammar() 함수 구현
> 
> 우리의 목표는 위의 문법과 같은 **EBNF** 문법을 일반 **BNF** 문법으로 변환하는 것이다. 이것은 네 가지 규칙에 의해 수행된다.
> 
> 1.  op가 ?, +, * 중 하나인 표현식 **(content)op** 은 **&lt;new-symbol&gt;op** 이 되며, 새로운 규칙 **&lt;new-symbol&gt; ::= content**가 된다.
> 2. **&lt;symbol&gt;?** 표현은 **&lt;new-symbol&gt; ::= &lt;empty&gt; | &lt;symbol&gt;** 인 **&lt;new-symbol&gt;** 이 된다.
> 3. **&lt;symbol&gt;+** 표현은 **&lt;new-symbol&gt; ::= &lt;symbol&gt; | &lt;symbol&gt;&lt;new-symbol&gt;** 인 **&lt;new-symbol&gt;** 이 된다.
> 4. **&lt;symbol&gt;*** 표현은 **&lt;new-symbol&gt; ::= &lt;empty&gt; | &lt;symbol&gt;&lt;new-symbol&gt;** 인 **&lt;new-symbol&gt;** 이 된다.
>
> 여기서 **&lt;empty&gt;** 는 **&lt;empty&gt; ::=** 에서와 같이 빈 문자열로 확장된다. (이것은 엡실론 확장이라고도 불린다.)
>
> 이러한 연산자가 정규식을 상기시킨다면, 이것은 우연이 아니다. 사실, 기본 정규 표현식은 위의 규칙(그리고 위에서 정의한 대로 crange()가 있는 문자 클래스)을 사용하여 문법으로 변환할 수 있다.
> 
> 위의 예에 이러한 규칙을 적용하면 다음과 같은 결과가 나올 수 있다.
> 
> * **&lt;idchar&gt;+** 는 **&lt;new-symbol&gt; ::= &lt;idchar&gt; | &lt;idchar&gt;&lt;new-symbol&gt;** 으로 **&lt;idchar&gt;&lt;new-symbol&gt;** 이 된다.
> * **&lt;integer&gt;(.&lt;integer&gt;)?** 는 **&lt;new-symbol&gt; ::= &lt;empty&gt; | .&lt;integer&gt;** 로 **&lt;integer&gt;&lt;new-symbol&gt;** 가 된다.
> 
> 이 규칙을 세 단계로 시행해보자.
> 
> ##### Creating New Symbols
> 
> 첫째, 우리는 새로운 기호를 만드는 메커니즘이 필요하다. 이건 꽤 간단하다.
> 
```python
>>> def new_symbol(grammar: Grammar, symbol_name: str = "<symbol>") -> str:
	    """Return a new symbol for `grammar` based on `symbol_name`"""
	    if symbol_name not in grammar:
	        return symbol_name

	    count = 1
	    while True:
	        tentative_symbol_name = symbol_name[:-1] + "-" + repr(count) + ">"
	        if tentative_symbol_name not in grammar:
	            return tentative_symbol_name
	        count += 1

>>> assert new_symbol(EXPR_EBNF_GRAMMAR, '<expr>') == '<expr-1>'
```
> ##### Expanding Parenthesized Expressions
> 
> 다음으로, 확장에서 괄호화된 표현을 추출하고 위의 규칙에 따라 확장할 수 있는 수단이 필요하다. 표현식을 추출하는 것부터 시작하자.
```python
>>> RE_PARENTHESIZED_EXPR = re.compile(r'\([^()]*\)[?+*]')
>>> def parenthesized_expressions(expansion: Expansion) -> List[str]:
	    # In later chapters, we allow expansions to be tuples,
	    # with the expansion being the first element
	    if isinstance(expansion, tuple):
	        expansion = expansion[0]

	    return re.findall(RE_PARENTHESIZED_EXPR, expansion)
	    
>>> assert parenthesized_expressions("(<foo>)* (<foo><bar>)+ (+<foo>)? <integer>(.<integer>)?") == [
    '(<foo>)*', '(<foo><bar>)+', '(+<foo>)?', '(.<integer>)?']
```
> 이제 이것들을 사용하여 위의 규칙 1을 적용하여 괄호 안에 표현식에 대한 새로운 기호를 도입할 수 있다.
```python
def convert_ebnf_parentheses(ebnf_grammar: Grammar) -> Grammar:
    """Convert a grammar in extended BNF to BNF"""
    grammar = extend_grammar(ebnf_grammar)
    for nonterminal in ebnf_grammar:
        expansions = ebnf_grammar[nonterminal]

        for i in range(len(expansions)):
            expansion = expansions[i]
            if not isinstance(expansion, str):
                expansion = expansion[0]

            while True:
                parenthesized_exprs = parenthesized_expressions(expansion)
                if len(parenthesized_exprs) == 0:
                    break

                for expr in parenthesized_exprs:
                    operator = expr[-1:]
                    contents = expr[1:-2]

                    new_sym = new_symbol(grammar)

                    exp = grammar[nonterminal][i]
                    opts = None
                    if isinstance(exp, tuple):
                        (exp, opts) = exp
                    assert isinstance(exp, str)

                    expansion = exp.replace(expr, new_sym + operator, 1)
                    if opts:
                        grammar[nonterminal][i] = (expansion, opts)
                    else:
                        grammar[nonterminal][i] = expansion

                    grammar[new_sym] = [contents]

    return grammar
```
> 이것은 위에서 스케치한 대로 변환을 수행한다.
```python
>>> convert_ebnf_parentheses({"<number>": ["<integer>(.<integer>)?"]})
{'<number>': ['<integer><symbol>?'], '<symbol>': ['.<integer>']}
```
> 그것은 중첩된 괄호 표현식에서도 작동한다.
```python
>>> convert_ebnf_parentheses({"<foo>": ["((<foo>)?)+"]})
{'<foo>': ['<symbol-1>+'], '<symbol>': ['<foo>'], '<symbol-1>': ['<symbol>?']}
```
> ##### Expanding Operators
> 
> 괄호 표현식을 확장한 후, 이제 기호와 연산자(?, *, +)를 처리해야 한다. 위의 **convert_ebnf_parentheses()** 와 마찬가지로, 먼저 모든 기호와 연산자를 추출한다.
```python
>>> RE_EXTENDED_NONTERMINAL = re.compile(r'(<[^<> ]*>[?+*])')
>>> def extended_nonterminals(expansion: Expansion) -> List[str]:
	    # In later chapters, we allow expansions to be tuples,
	    # with the expansion being the first element
	    if isinstance(expansion, tuple):
	        expansion = expansion[0]

	    return re.findall(RE_EXTENDED_NONTERMINAL, expansion)

>>> assert extended_nonterminals("<foo>* <bar>+ <elem>? <none>") == ['<foo>*', '<bar>+', '<elem>?']
```
> 
> 해당 변환기는 기호와 연산자를 추출하고, 위에 제시된 규칙에 따라 새로운 기호를 추가한다.
```python
>>> def convert_ebnf_operators(ebnf_grammar: Grammar) -> Grammar:
	    """Convert a grammar in extended BNF to BNF"""
	    grammar = extend_grammar(ebnf_grammar)
	    for nonterminal in ebnf_grammar:
	        expansions = ebnf_grammar[nonterminal]

	        for i in range(len(expansions)):
	            expansion = expansions[i]
	            extended_symbols = extended_nonterminals(expansion)

	            for extended_symbol in extended_symbols:
	                operator = extended_symbol[-1:]
	                original_symbol = extended_symbol[:-1]
	                assert original_symbol in ebnf_grammar, \
	                    f"{original_symbol} is not defined in grammar"

	                new_sym = new_symbol(grammar, original_symbol)

	                exp = grammar[nonterminal][i]
	                opts = None
	                if isinstance(exp, tuple):
	                    (exp, opts) = exp
	                assert isinstance(exp, str)

	                new_exp = exp.replace(extended_symbol, new_sym, 1)
	                if opts:
	                    grammar[nonterminal][i] = (new_exp, opts)
	                else:
	                    grammar[nonterminal][i] = new_exp

	                if operator == '?':
	                    grammar[new_sym] = ["", original_symbol]
	                elif operator == '*':
	                    grammar[new_sym] = ["", original_symbol + new_sym]
	                elif operator == '+':
	                    grammar[new_sym] = [
	                        original_symbol, original_symbol + new_sym]

	    return grammar

>>> convert_ebnf_operators({"<integer>": ["<digit>+"], "<digit>": ["0"]})
{'<integer>': ['<digit-1>'],
 '<digit>': ['0'],
 '<digit-1>': ['<digit>', '<digit><digit-1>']}
```
> ##### All Together
> 
> 우리는 먼저 괄호를 확장한 다음 연산자를 결합할 수 있다.
```python
def convert_ebnf_grammar(ebnf_grammar: Grammar) -> Grammar:
    return convert_ebnf_operators(convert_ebnf_parentheses(ebnf_grammar))
```

다음은 convert_ebnf_grammar()를 사용하는 예이다.

```python
>>> convert_ebnf_grammar({"<authority>": ["(<userinfo>@)?<host>(:<port>)?"]})
{'<authority>': ['<symbol-2><host><symbol-1-1>'],
 '<symbol>': ['<userinfo>@'],
 '<symbol-1>': [':<port>'],
 '<symbol-2>': ['', '<symbol>'],
 '<symbol-1-1>': ['', '<symbol-1>']}
>>> expr_grammar = convert_ebnf_grammar(EXPR_EBNF_GRAMMAR)
>>> expr_grammar
{'<start>': ['<expr>'],
 '<expr>': ['<term> + <expr>', '<term> - <expr>', '<term>'],
 '<term>': ['<factor> * <term>', '<factor> / <term>', '<factor>'],
 '<factor>': ['<sign-1><factor>', '(<expr>)', '<integer><symbol-1>'],
 '<sign>': ['+', '-'],
 '<integer>': ['<digit-1>'],
 '<digit>': ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9'],
 '<symbol>': ['.<integer>'],
 '<sign-1>': ['', '<sign>'],
 '<symbol-1>': ['', '<symbol>'],
 '<digit-1>': ['<digit>', '<digit><digit-1>']}
```

성공했다. EBNF 문법을 BNF로 잘 변환했다.

문자 클래스와 **EBNF** 문법 변환을 통해 문법을 더 쉽게 작성할 수 있는 두 가지 강력한 도구가 있다. 문법 작업과 관련하여 이것들을 반복해서 사용할 것이다.

### Grammar Extensions
---
이 책의 과정에서, 종종 확률이나 제약과 같은 문법에 대한 추가 정보를 지정하고 싶다. 이러한 확장과 다른 확장을 지원하기 위해, 우리는 주석 메커니즘을 정의한다

문법에 주석을 달기 위한 개념은 개별 확장에 주석을 추가하는 것이다. 이를 위해, 우리는 확장이 문자열뿐만 아니라 문자열 한 쌍과 속성 집합이 될 수 있도록 허용한다.

```
"<expr>":
        [("<term> + <expr>", opts(min_depth=10)),
         ("<term> - <expr>", opts(max_depth=2)),
         "<term>"]
```

여기서 **opts()** 함수는 개별 확장에 적용되는 주석을 표현할 수 있다. 이 경우, 덧셈은 **min_depth** 값 10으로 주석을 달고, 뺄셈은 **max_depth** 값이 2이다. 이러한 주석의 의미는 문법을 다루는 개별 알고리즘에 맡겨진다. 하지만 일반적인 아이디어는 무시될 수 있다는 것이다.

---
## Checking Grammars

문법은 문자열로 표현되기 때문에 오류를 도입하는 것은 상당히 쉽다. 그래서 일관성을 위해 문법을 확인하는 도우미 기능을 알아보자.

도우미 함수 **is_valid_grammar()** 는 사용된 모든 기호가 정의되었는지 확인하기 위해 문법을 반복하며, 그 반대도 마찬가지이며, 이는 디버깅에 매우 유용하다. 또한 시작 기호에서 모든 기호에 도달할 수 있는지 확인한다. 여기서 세부 사항을 탐구할 필요는 없지만, 항상 그렇듯이, 사용하기 전에 입력 데이터를 바로 얻는 것이 중요하다.

> #### **is_valid_grammar()** 함수 구현
> 
```python
import [sys](https://docs.python.org/3/library/sys.html)

def def_used_nonterminals(grammar: Grammar, start_symbol: 
                          str = START_SYMBOL) -> Tuple[Optional[Set[str]], 
                                                       Optional[Set[str]]]:
    """Return a pair (`defined_nonterminals`, `used_nonterminals`) in `grammar`.
 In case of error, return (`None`, `None`)."""

    defined_nonterminals = set()
    used_nonterminals = {start_symbol}

    for defined_nonterminal in grammar:
        defined_nonterminals.add(defined_nonterminal)
        expansions = grammar[defined_nonterminal]
        if not isinstance(expansions, list):
            print(repr(defined_nonterminal) + ": expansion is not a list",
                  file=sys.stderr)
            return None, None

        if len(expansions) == 0:
            print(repr(defined_nonterminal) + ": expansion list empty",
                  file=sys.stderr)
            return None, None

        for expansion in expansions:
            if isinstance(expansion, tuple):
                expansion = expansion[0]
            if not isinstance(expansion, str):
                print(repr(defined_nonterminal) + ": "
                      + repr(expansion) + ": not a string",
                      file=sys.stderr)
                return None, None

            for used_nonterminal in nonterminals(expansion):
                used_nonterminals.add(used_nonterminal)

    return defined_nonterminals, used_nonterminals

def reachable_nonterminals(grammar: Grammar,
                           start_symbol: str = START_SYMBOL) -> Set[str]:
    reachable = set()

    def _find_reachable_nonterminals(grammar, symbol):
        nonlocal reachable
        reachable.add(symbol)
        for expansion in grammar.get(symbol, []):
            for nonterminal in nonterminals(expansion):
                if nonterminal not in reachable:
                    _find_reachable_nonterminals(grammar, nonterminal)

    _find_reachable_nonterminals(grammar, start_symbol)
    return reachable

def unreachable_nonterminals(grammar: Grammar,
                             start_symbol=START_SYMBOL) -> Set[str]:
    return grammar.keys() - reachable_nonterminals(grammar, start_symbol)

def opts_used(grammar: Grammar) -> Set[str]:
    used_opts = set()
    for symbol in grammar:
        for expansion in grammar[symbol]:
            used_opts |= set(exp_opts(expansion).keys())
    return used_opts

def is_valid_grammar(grammar: Grammar,
                     start_symbol: str = START_SYMBOL, 
                     supported_opts: Set[str] = set()) -> bool:
    """Check if the given `grammar` is valid.
 `start_symbol`: optional start symbol (default: `<start>`)
 `supported_opts`: options supported (default: none)"""

    defined_nonterminals, used_nonterminals = \
        def_used_nonterminals(grammar, start_symbol)
    if defined_nonterminals is None or used_nonterminals is None:
        return False

    # Do not complain about '<start>' being not used,
    # even if start_symbol is different
    if START_SYMBOL in grammar:
        used_nonterminals.add(START_SYMBOL)

    for unused_nonterminal in defined_nonterminals - used_nonterminals:
        print(repr(unused_nonterminal) + ": defined, but not used",
              file=sys.stderr)
    for undefined_nonterminal in used_nonterminals - defined_nonterminals:
        print(repr(undefined_nonterminal) + ": used, but not defined",
              file=sys.stderr)

    # Symbols must be reachable either from <start> or given start symbol
    unreachable = unreachable_nonterminals(grammar, start_symbol)
    msg_start_symbol = start_symbol

    if START_SYMBOL in grammar:
        unreachable = unreachable - \
            reachable_nonterminals(grammar, START_SYMBOL)
        if start_symbol != START_SYMBOL:
            msg_start_symbol += " or " + START_SYMBOL

    for unreachable_nonterminal in unreachable:
        print(repr(unreachable_nonterminal) + ": unreachable from " + msg_start_symbol,
              file=sys.stderr)

    used_but_not_supported_opts = set()
    if len(supported_opts) > 0:
        used_but_not_supported_opts = opts_used(
            grammar).difference(supported_opts)
        for opt in used_but_not_supported_opts:
            print(
                "warning: option " +
                repr(opt) +
                " is not supported",
                file=sys.stderr)

    return used_nonterminals == defined_nonterminals and len(unreachable) == 0
```

**is_valid_grammar()** 함수를 사용해보자. 위에 정의된 우리의 문법은 테스트를 통과한다.

```python
>>> assert is_valid_grammar(EXPR_GRAMMAR)
>>> assert is_valid_grammar(CGI_GRAMMAR)
>>> assert is_valid_grammar(URL_GRAMMAR)
```

하지만 아래 예시는 테스트를 통과하지 못한다.

```python
>>> assert not is_valid_grammar({"<start>": ["<x>"], "<y>": ["1"]})
'''
'<y>': defined, but not used
'<x>': used, but not defined
'<y>': unreachable from <start>
'''
>>> assert not is_valid_grammar({"<start>": "123"})
'''
'<start>': expansion is not a list
'''
>>> assert not is_valid_grammar({"<start>": []})
'''
'<start>': expansion list empty
'''
>>> assert not is_valid_grammar({"<start>": [1, 2, 3]})
'''
'<start>': 1: not a string
'''
```

이제부터, 우리는 문법을 정의할 때 항상 **is_valid_grammar()** 함수를 사용할 것이다.