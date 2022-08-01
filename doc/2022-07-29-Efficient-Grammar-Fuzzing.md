---
title: "[The Fuzzing Book] - Efficient Grammar Fuzzing"
comments: true
categories:
- tech
tags:
- tech
- python
- testing
- fuzzing
last_modified_at: 2022-07-29T18:15+09:00
toc: true
---

# 목차
* Synopsis
  * Efficient Grammar Fuzzing
  * Derivation Trees
* An Insufficient Algorithm
* Derivation Trees
* Representing Derivation Trees
* Expanding a Node
  * Picking a Children Alternative to be Expanded
  * Getting a List of Possible Expansions
  * Putting Things Together
* Expanding a Trees
* Closing the Expansion
* Node Inflation
* Three Expansion Phases
* Putting it all Together

---
# Efficient Grammar Fuzzing

이전 Grammar 장에서 우리는 효과적이고 효율적인 테스트를 위한 Grammar 사용법을 알아보았다. 이번 장에서는 이전의 **"문자열 기반"** 알고리즘을 더 빠르고 퍼즈 입력값 생성을 넘어서 더 제어하기 쉬운 **"트리 기반"** 알고리즘으로 재정의 할 것이다. 

이번 장에서 배우는 알고리즘은 다른 여러 기술들의 기초를 제공한다. 

---
## Synopsis
---
### Efficient Grammar Fuzzing
---

이번 장에서는 체계적인 입력 문자열 생성을 위한 문법을 사용하는 효율적인 문법 퍼저인 **GrammarFuzzer**를 소개한다. 아래 코드는 해당 퍼저의 사용법이다.

```python 
>>> from Grammars import US_PHONE_GRAMMAR
>>> phone_fuzzer = GrammarFuzzer(US_PHONE_GRAMMAR)
>>> phone_fuzzer.fuzz()
'(519)333-4454'
```

**GrammarFuzzer** 생성자는 클래스의 행동을 제어하기 위해 몇개의 키워드 인자를 받는다. 예를 들어서 **start_symbol** 키워드 인자는 확장을 처음 시작하는 기호를 설정할 수 있게 해준다.

```python
>>> area_fuzzer = GrammarFuzzer(US_PHONE_GRAMMAR, start_symbol='<area>')
>>> area_fuzzer.fuzz()
'718'
```

GrammarFuzzer 생성자를 매개 변수화하는 방법은 다음과 같다.

```python
Produce strings from `grammar`, starting with `start_symbol`.
If `min_nonterminals` or `max_nonterminals` is given, use them as limits 
for the number of nonterminals produced.  
If `disp` is set, display the intermediate derivation trees.
If `log` is set, show intermediate steps as text on standard output.
```

![fuzzer4-1.png](../src/fuzzer4-1.png)

### Derivation Trees
---

내부적으로 **GrammarFuzzer**는 파생 트리를 사용하여 단계별로 확장한다. 문자열을 생성한 후, 생성된 트리는 **derivation_tree** 속성에서 액세스할 수 있다.

```python
>>> display_tree(phone_fuzzer.derivation_tree)
```

![fuzzer4-2.png](../src/fuzzer4-2.png)

파생 트리의 내부 표현에서 **node**는 (symbol, children) 쌍이다. 비단말 문자에서 **symbol** 은 확장되어지는 기호이고, **children**은 추가된 **node**들의 리스트이다. 단말문자에서는 **symbol** 은 단말 문자열이고, **children** 은 비어있다.

```python
>>> phone_fuzzer.derivation_tree
('<start>',
 [('<phone-number>',
   [('(', []),
    ('<area>',
     [('<lead-digit>', [('5', [])]),
      ('<digit>', [('1', [])]),
      ('<digit>', [('9', [])])]),
    (')', []),
    ('<exchange>',
     [('<lead-digit>', [('3', [])]),
      ('<digit>', [('3', [])]),
      ('<digit>', [('3', [])])]),
    ('-', []),
    ('<line>',
     [('<digit>', [('4', [])]),
      ('<digit>', [('4', [])]),
      ('<digit>', [('5', [])]),
      ('<digit>', [('4', [])])])])])
```

이번장에는 위에서 쓰인 **display_tree()** 같은 가시화 툴을 비롯하여 파생 트리를 작업하기 위한 다양한 도우미들을 포함하고 있다.

---
## An Insufficient Algorithm
---
이전 장에서는 문법을 받아서 이것으로부터 자동으로 체계화된 문자열을 만들어주는 **simple_grammar_fuzzer()** 함수를 사용했었다. 하지만 **simple_grammar_fuzzer()** 함수는 그 이름에서도 알수 있듯이 너무 간단한 함수다. 문제를 설명하기 위해 **[Fuzzing with Grammar](https://www.fuzzingbook.org/html/Grammars.html)** 챕터에서 **EXPR_GRAMMAR_BNF** 로 만들었던 **expr_grammar** 로 돌아갈 것이다.

```python
>>> import bookutils
>>> from bookutils import quiz
>>> from typing import Tuple, List, Optional, Any, Union, Set, Callable, Dict
>>> from bookutils import unicode_escape
>>>from Grammars import EXPR_EBNF_GRAMMAR, convert_ebnf_grammar, Grammar, Expansion
>>> from Grammars import simple_grammar_fuzzer, is_valid_grammar, exp_string
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

**expr_grammar** 은 흥미로운 특징을 가지고 있다. 이 문법을 **simple_grammar_fuzzer()** 함수의 인자로 넘겨주면 에러가 발생한다.

```python
>>> from ExpectError import ExpectTimeout
>>> with ExpectTimeout(1):
      simple_grammar_fuzzer(grammar=expr_grammar, max_nonterminals=3)

Traceback (most recent call last):
  File "/var/folders/n2/xd9445p97rb3xh7m1dfx8_4h0006ts/T/ipykernel_71164/3259437052.py", line 2, in <cell line: 1>
    simple_grammar_fuzzer(grammar=expr_grammar, max_nonterminals=3)
  File "/Users/zeller/Projects/fuzzingbook/notebooks/Grammars.ipynb", line 87, in simple_grammar_fuzzer
    symbol_to_expand = random.choice(nonterminals(term))
  File "/Users/zeller/Projects/fuzzingbook/notebooks/Grammars.ipynb", line 61, in nonterminals
    return RE_NONTERMINAL.findall(expansion)
  File "/Users/zeller/Projects/fuzzingbook/notebooks/Timeout.ipynb", line 43, in timeout_handler
    raise TimeoutError()
TimeoutError (expected)
```

왜 이런일이 발생할까? 문법을 다시한번 살펴보자. <strong style="color: red">remember what you know about simple_grammar_fuzzer(); and run simple_grammar_fuzzer() with log=true argument to see the expansions.</strong>

게다가 문제는 아래래 규칙에 있다.

```python
>>> expr_grammar['<factor>']
['<sign-1><factor>', '(<expr>)', '<integer><symbol-1>']
```

여기서, **(expr)** 를 제외한 모든 선택은 일시적일지라도 기호의 수를 증가시킨다. 확장할 기호의 수에 어려운 제한을 두기 때문에, **&lt;factor&gt;** 를 확장하기 위한 유일한 선택은 **(&lt;expr&gt;)**이며, 이로 인해 괄호가 무한히 추가된다.

잠재적으로 무한한 확장 문제는 **simple_grammar_fuzzer()** 의 문제 중 하나이다. 그 밖에 아래와 같은 문제들이 포함되어 있다.

1. 비효율적이다. 각각의 반복에서 이 퍼저는 기호를 확장하기 위해 지금까지 생성된 문자열을 검색할 것이다. 이것은 생산 문자열이 증가함에 따라 비효율적이 된다.
2. 통제하기 어렵다. **symbol**의 수를 제한하더라도, 위에서 논의한 것처럼 매우 긴 문자열과 무한히 긴 문자열을 얻을 수 있다.

길이가 다른 문자열에 필요한 시간을 표시하여 두 문제를 모두 설명해보자.

```python
>>> from Grammars import simple_grammar_fuzzer
>>> from Grammars import START_SYMBOL, EXPR_GRAMMAR, URL_GRAMMAR, CGI_GRAMMAR
>>> from Grammars import RE_NONTERMINAL, nonterminals, is_nonterminal
>>> from Timer import Timer
>>> trials = 50
>>> xs = []
>>> ys = []
>>> for i in range(trials):
      with Timer() as t:
        s = simple_grammar_fuzzer(EXPR_GRAMMAR, max_nonterminals=15)
      xs.append(len(s))
      ys.append(t.elapsed_time())
      print(i, end=" ")
      
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 
>>> average_time = sum(ys) / trials
>>> print("Average time:", average_time)
Average time: 0.20005843754
```

```python
%matplotlib inline

import matplotlib.pyplot as plt
plt.scatter(xs, ys)
plt.title('Time required for generating an output');
```

![fuzzer4-3](../src/fuzzer4-3.png)

그래프를 보면 

(1) 출력을 생성하는 데 필요한 시간이 그 ouptut의 길이에 따라 이차적으로 증가하고, 

(2) 생성된 출력의 상당 부분이 수만 문자라는 것을 알 수 있다.

이러한 문제를 해결하기 위해, 우리는 더 효율적이고 확장을 더 잘 제어할 수 있으며, **expr_grammar** 에서 **(expr)** 대안이 다른 두 가지와 달리 잠재적으로 무한한 확장을 산출한다는 것을 예측할 수 있는 더 똑똑한 알고리즘이 필요하다.

---
## Derivation Trees
---

더 효율적인 알고리즘을 얻고 확대를 더 잘 제어하기 위해, 우리는 문법이 생산하는 문자열에 특별한 표현을 사용할 것이다. 일반적인 아이디어는 나중에 확장될 트리 구조인 소위 파생 트리를 사용하는 것이다. 이 표현을 통해 우리는 항상 확장 상태를 추적할 수 있다 - 어떤 요소가 다른 요소로 확장되었는지, 어떤 기호를 여전히 확장해야 하는지와 같은 질문에 답할 수 있다. 게다가, 트리에 새로운 요소를 추가하는 것은 문자열을 반복해서 교체하는 것보다 훨씬 더 효율적이다.

프로그래밍에 사용되는 다른 트리와 마찬가지로, 파생 트리(구문 분석 트리 또는 콘크리트 구문 트리라고도 함)는 다른 노드(자식 노드라고 함)를 자식으로 가진 노드로 구성된다. 트리는 부모가 없는 하나의 노드로 시작한다. 이것은 루트 노드라고 불린다. 자식이 없는 노드는 리프라고 불린다.

파생 트리가 있는 문법 확장 과정은 문법 장에서 산술 문법을 사용하여 다음 단계에 설명되어 있다. 우리는 시작 기호를 나타내는 트리의 루트로 단일 노드로 시작한다.

```
<start>
```

트리를 확장하기 위해 트리를 탐색하고 자식이 없는 비단말 기호 **S** 를 찾는다. 따라서 **S** 는 여전히 확장되어야 하는 기호다. 그런 다음 문법에서 **S** 에 대한 확장을 선택한다. 그 후 확장을 **S** 의 새 자식으로 추가한다. 시작 기호 **&lt;start&gt;** 의 경우 유일한 확장은 **&lt;expr&gt;** 이므로 자식으로 추가한다.

![fuzzer4-4.png](../src/fuzzer4-4.png)

파생 트리에서 생성된 문자열을 구성하기 위해, 트리를 순서대로 탐색하고 기호를 수집한다. 위의 경우에는 **"&lt;expr&gt;"** 문자열을 얻는다.

트리를 더 확장하기 위해, 확장할 또 다른 기호를 선택하고, 확장을 새로운 자식으로 추가합니다. 이렇게 하면 **&lt;expr&gt;, +, &lt;term&gt;** 으로 확장되어 세 명의 자식을 추가하는 **&lt;expr&gt;** 기호를 얻을 수 있다.

![fuzzer4-5.png](../src/fuzzer4-5.png)
