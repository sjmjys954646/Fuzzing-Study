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

확장할 기호가 남지 않을 때까지 확장을 반복한다.

![fuzzer4-6.png](../src/fuzzer4-6.png)

이제 문자열 **2 + 2** 에 대한 표현이 있다. 그러나 문자열과 달리, 파생 트리는 생성된 문자열의 전체 구조(및 생산 이력 또는 파생 기록)를 기록한다. 이 특징은 서브트리(하부구조)를 다른 서브트리(하부구조)로 대체하는 등의 간단한 비교와 조작을 가능하게 한다.


---
## Representing Derivation Trees
---

파생트리를 파이썬에서 구현하기 위해 우리는 아래와 같은 표현을 사용할 것이다.

```
(SYMBOL_NAME, CHILDREN)
```

**(SYMBOL_NAME, CHILDREN)** 쌍으로 이루어져 있는 **node**에서 **SYMBOL_NAME** 은 **&lt;start&gt;** 나 **+** 와 같이 **node**를 표현하는 문자열이고, **CHILDREN**은 자식 노드로 이루어진 리스트이다. 

**CHILDREN** 은 특별한 값을 가질 수 있다.

1. 미래 확장을 위한 지시표지자로서의 **None** . 이것은 노드가 추가로 더 확장되어야 하는 **비단말 기호** 라는 것을 나타낸다.
2. **CHILDREN**이 없음을 나타내는 **[]** (비어있는 리스트). 이것은 노드가 더 이상 확장할 수 없는 **단말 기호**라는 것을 나타낸다.

**DerivationTree** 타입은 바로 이 구조를 포착한다. (**Any** 타입은 실제로 **DerivationTree** 타입을 읽어야 하지만, 파이썬 정적 유형 검사기는 재귀 유형을 잘 처리할 수 없다.)

```python
DerivationTree = Tuple[str, Optional[List[Any]]]
```

위에서 설명한 **&lt;expr&gt; + &lt;term&gt;** 의 중간 단계를 표현하는 간단한 파생 트리를 만들어보자.

```python
derivation_tree: DerivationTree = ("<start>",
	[
		(
			"<expr>",
			[
				("<expr>", None),
				(" + ", []),
				("<term>", None)
			]
		)
    ]
)
```

위 트리 구조를 더 잘 이해하기 위해 위 트리를 가시화 하는 **display_tree()** 함수에 대해서 알아보자.

### display_tree() 구현
---
우리는 위 구조를 탐색할 때 **graphviz** 패키지에서 **dot drawing program** 을 알고리즘적으로 사용할 것이다. (트리 시각화에 관심이 없다면 아래 예시로 직접 건너뛸 수 있다.)

```python
>>> from graphviz import Digraph
>>> from IPython.display import display
>>> import re
>>> def dot_escape(s: str) -> str:
      """Return s in a form suitable for dot"""
      s = s.replace('\n', '\\n')
      s = re.sub(r'([^a-zA-Z0-9" ])', r"\\\1", s)
      return s
>>> assert dot_escape("hello") == "hello"
>>> assert dot_escape("<hello>, world") == "\\<hello\\>\\, world"
>>> assert dot_escape("\\n") == "\\\\n"
>>> assert dot_escape("\n") == "\\\\n"
```

**derivation_tree** 를 시각화하는데 관심이 있기 떄문에 시각화 과정을 일반화 하는 것이 우리의 관심사이다. 특히 **display_tree()** 함수가 데이터 구조와 같은 어떤 트리도 보여줄 수 있다면 더 도움이 될것이다. 이것을 가능하게 하기 위해 주어진 데이터 구조에서 **symbol**과 **children**을 추출하는 **extract_node()** 헬퍼 함수를 정의할 것이다.

기본 구현은 간단하게 어떠한 파생 트리 노드(node)에서 **symbol** , **children** , **annotation** 을 추출하기만 하면 된다.

```python
def extract_node(node, id):
    symbol, children, *annotation = node
    return symbol, children, ''.join(str(a) for a in annotation)
```

트리 시각화 과정에서 특정 노드들은 다르게 보여주는 것은 종종 유용하다. 예를 들어 **처리되지 않은 노드** 와 **처리된 노드** 를 구분하는 것은 종종 유용할 때가 있다. 우리는 사용자가 커스터마이징 할 수 있는 기본 디스플레이를 제공하는 헬퍼 함수인 **default_node_attr()** 를 정의할 것이다. 

```python
def default_node_attr(dot, nid, symbol, ann):
    dot.node(repr(nid), dot_escape(unicode_escape(symbol)))
```

노드(node)와 비슷하게 **edge** 또한 수정이 필요하다. 이것을 위해서 사용자가 커스터마이징 할 수 있는 **default_edge_attr()** 헬퍼 함수를 정의할 것이다. 

```python
def default_edge_attr(dot, start_node, stop_node):
    dot.edge(repr(start_node), repr(stop_node))
```

트리를 시각화할 때 어떤 사람들은 트리의 외형을 변경하고 싶을 수도 있다. 예를 들어 만약 트리가 위에서 아래가 아니라 왼쪽에서 오른쪽으로 배치된다면 더 보기 쉬울 수도 있다. 이것을 위해서 **default_graph_attr()** 헬퍼 함수를 정의할 것이다. 

```python
def default_graph_attr(dot):
    dot.attr('node', shape='plain')
```

마지막으로 위에서 정의한 4가지 함수 **extract_node(), default_edge_attr(), default_node_attr(),  default_graph_attr()** 를 매개변수로 받고, 트리를 보여주기 위해 사용할 수 있도록 **display_tree()** 함수를 정의할 것이다.

```python
def display_tree(derivation_tree: DerivationTree,
                 log: bool = False,
                 extract_node: Callable = extract_node,
                 node_attr: Callable = default_node_attr,
                 edge_attr: Callable = default_edge_attr,
                 graph_attr: Callable = default_graph_attr) -> Any:

    # If we import display_tree, we also have to import its functions
    from [graphviz](https://graphviz.readthedocs.io/) import Digraph

    counter = 0

    def traverse_tree(dot, tree, id=0):
        (symbol, children, annotation) = extract_node(tree, id)
        node_attr(dot, id, symbol, annotation)

        if children:
            for child in children:
                nonlocal counter
                counter += 1
                child_id = counter
                edge_attr(dot, id, child_id)
                traverse_tree(dot, child, child_id)

    dot = Digraph(comment="Derivation Tree")
    graph_attr(dot)
    traverse_tree(dot, derivation_tree)
    if log:
        print(dot)
    return dot
```
---
아래는 우리의 트리를 시각화한 것이다.

```python
display_tree(derivation_tree)
```
![fuzzer4-7](../src/fuzzer4-7.png)

```python
>>> derivation_tree
('<start>', [('<expr>', [('<expr>', None), (' + ', []), ('<term>', None)])])
```

우리는 때때로 개별 노드에 **annotation** 을 추가할 수 있는 **display_annotated_tree()** 함수를 사용한다. 

### **display_annotated_tree()** 소스 코드와 예제
---

**display_annotated_tree()** 함수는 타입 주석이 표시된 데이터 구조를 보여주고, 그래프를 왼쪽에서 오른쪽으로 표시해준다. 

```python
def display_annotated_tree(tree: DerivationTree,
                           a_nodes: Dict[int, str],
                           a_edges: Dict[Tuple[int, int], str],
                           log: bool = False):
    def graph_attr(dot):
        dot.attr('node', shape='plain')
        dot.graph_attr['rankdir'] = 'LR'

    def annotate_node(dot, nid, symbol, ann):
        if nid in a_nodes:
            dot.node(repr(nid), 
                     "%s (%s)" % (dot_escape(unicode_escape(symbol)),
                                  a_nodes[nid]))
        else:
            dot.node(repr(nid), dot_escape(unicode_escape(symbol)))

    def annotate_edge(dot, start_node, stop_node):
        if (start_node, stop_node) in a_edges:
            dot.edge(repr(start_node), repr(stop_node),
                     a_edges[(start_node, stop_node)])
        else:
            dot.edge(repr(start_node), repr(stop_node))

    return display_tree(tree, log=log,
                        node_attr=annotate_node,
                        edge_attr=annotate_edge,
                        graph_attr=graph_attr)
```

```python
display_annotated_tree(derivation_tree, {3: 'plus'}, {(1, 3): 'op'}, log=False)
```
![fuzzer4-8](../src/fuzzer4-8.png)

---

만약 트리에 있는 모든 노드를 문자열 형식으로 보고 싶으면 아래 **all_terminals()** 함수를 사용할 수 있다. 

```python
>>> def all_terminals(tree: DerivationTree) -> str:
	    (symbol, children) = tree
	    if children is None:
	        # This is a nonterminal symbol not expanded yet
	        return symbol

	    if len(children) == 0:
	        # This is a terminal symbol
	        return symbol

	    # This is an expanded symbol:
	    # Concatenate all terminal symbols from all children
	    return ''.join([all_terminals(c) for c in children])

>>> all_terminals(derivation_tree)
'<expr> + <term>'
```

대체 **tree_to_string()** 함수는 트리를 문자열로 바꿔주기는 하지만 비단말 기호는 공백으로 바꿔버린다는 단점이 있다. 

```python
>>> def tree_to_string(tree: DerivationTree) -> str:
	    symbol, children, *_ = tree
	    if children:
	        return ''.join(tree_to_string(c) for c in children)
	    else:
	        return '' if is_nonterminal(symbol) else symbol

>>> tree_to_string(derivation_tree)
' + '
```

---
## Expanding a Node

이제 확장되지 않은 기호(ex. derivation_tree)가 있는 트리를 가져와 이 모든 기호를 차례로 확장하는 알고리즘을 개발하자. 이전 퍼저와 마찬가지로, 퍼저의 특별한 서브클래스인 **GrammarFuzzer** 를 만든다. **GrammarFuzzer** 는 문법과 시작 기호를 얻고, 다른 매개 변수는 나중에 생성을 더 제어하고 디버깅을 지원하는 데 사용된다.

```python
from [Fuzzer](https://www.fuzzingbook.org/html/Fuzzer.html) import Fuzzer

class GrammarFuzzer(Fuzzer):
    """Produce strings from grammars efficiently, using derivation trees."""

    def __init__(self,
                 grammar: Grammar,
                 start_symbol: str = START_SYMBOL,
                 min_nonterminals: int = 0,
                 max_nonterminals: int = 10,
                 disp: bool = False,
                 log: Union[bool, int] = False) -> None:
        """Produce strings from `grammar`, starting with `start_symbol`.
 If `min_nonterminals` or `max_nonterminals` is given, use them as limits 
 for the number of nonterminals produced. 
 If `disp` is set, display the intermediate derivation trees.
 If `log` is set, show intermediate steps as text on standard output."""

        self.grammar = grammar
        self.start_symbol = start_symbol
        self.min_nonterminals = min_nonterminals
        self.max_nonterminals = max_nonterminals
        self.disp = disp
        self.log = log
        self.check_grammar()  # Invokes is_valid_grammar()
```

**GrammarFuzzer** 에 추가로 메서드를 추가하기 위해 **MutationFuzzer** 클래스에 이미 도입한 방법을 사용한다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def new_method(self, args):
        pass
```

위 구조는 **GrammarFuzzer** 클래스에 새로운 **new_method()** 메서드를 추가할 수 있게 해준다.

### check_grammar() 구현
---

위의 방법을 사용하여 주어진 문법에 일관성을 확인하는 헬퍼 메서드 **check_grammar()** 를 정의할 수 있다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def check_grammar(self) -> None:
        """Check the grammar passed"""
        assert self.start_symbol in self.grammar
        assert is_valid_grammar(
            self.grammar,
            start_symbol=self.start_symbol,
            supported_opts=self.supported_opts())

    def supported_opts(self) -> Set[str]:
        """Set of supported options. To be overloaded in subclasses."""
        return set()  # We don't support specific options
```
---

이제 시작 기호로 트리를 구성하는 메서드 **init_tree()**를 정의하자.

```python
>>> class GrammarFuzzer(GrammarFuzzer):
	    def init_tree(self) -> DerivationTree:
	        return (self.start_symbol, None)

>>> f = GrammarFuzzer(EXPR_GRAMMAR)
>>> display_tree(f.init_tree())
```
![fuzzer4-9](../src/fuzzer4-9.png)

이것이 이제 우리가 확장할 트리이다. 

### Picking a Children Alternative to be Expanded
---

**GrammarFuzzer** 의 핵심 메서드 중 하나는 **choose_node_expansion()** 이다. 이 메서드는 노드(ex. <start> 노드)와 확장 가능한 자식 목록(문법에서 가능한 모든 확장에 대해 하나씩)를 얻은 후, 그 중 하나를 선택하고, 가능한 자식 목록에서 인덱스를 반환한다.

이 메서드(특히 이후 장)에 과부하를 부여함으로써, 다른 전략을 구현할 수 있다. 현재로서는 주어진 자식 목록 중 하나를 무작위로 선택한다(즉, 파생 트리 목록이다).

```python
class GrammarFuzzer(GrammarFuzzer):
    def choose_node_expansion(self, node: DerivationTree,
                              children_alternatives: List[List[DerivationTree]]) -> int:
        """Return index of expansion in `children_alternatives` to be selected.
 'children_alternatives`: a list of possible children for `node`.
 Defaults to random. To be overloaded in subclasses."""
        return random.randrange(0, len(children_alternatives))
```
---
### Getting a List of Possible Expansions
---

실제로 가능한 자식 목록을 얻으려면 확장 문자열을 가져와 문자열의 각 기호(단말 또는 비단말)에 대해 하나씩 파생 트리 목록으로 분해하는 헬퍼 함수 **expansion_to_children()** 가 필요하다.

### expansion_to_children() 구현
---

**expansion_to_children()** 함수는 **re.split()** 메서드를 사용하여 확장 문자열을 자식 노드 목록으로 분할한다.

```python
def expansion_to_children(expansion: Expansion) -> List[DerivationTree]:
    # print("Converting " + repr(expansion))
    # strings contains all substrings -- both terminals and nonterminals such
    # that ''.join(strings) == expansion

    expansion = exp_string(expansion)
    assert isinstance(expansion, str)

    if expansion == "":  # Special case: epsilon expansion
        return [("", [])]

    strings = re.split(RE_NONTERMINAL, expansion)
    return [(s, None) if is_nonterminal(s) else (s, [])
            for s in strings if len(s) > 0]
```
---

```python
>>> expansion_to_children("<term> + <expr>")
[('<term>', None), (' + ', []), ('<expr>', None)]
```

엡실론 확장의 경우, 즉 **&lt;symbol&gt; ::=** 에서와 같이 빈 문자열로 확장하는 경우 특별한 조치가 필요하다.

```python
>>> expansion_to_children("")
[('', [])]
```

**Grammar** 장의 **nonterminals()** 함수와 마찬가지로, 향후 확장을 제공하여 확장이 추가 데이터가 있는 튜플이 될 수 있도록 한다.

```python
>>> expansion_to_children(("+<term>", {"extra_data": 1234}))
[('+', []), ('<term>', None)]
```

우리는 이 도우미를 하위 클래스에 의해 오버로드될 수 있도록 **GrammarFuzzer** 의 메서드로 인식한다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def expansion_to_children(self, expansion: Expansion) -> List[DerivationTree]:
        return expansion_to_children(expansion)
```

---
### Putting Things Together
---

지금 우리는 아래 3가지 요소를 가지고 있다. 

1. 트리의 확장되지 않은 노드
2. 무작위 확장 선택
3. 새로운 트리 반환


아래가 **expand_node_randomly()** 메서드가 하는 일이다.

### expand_node_randomly() 구현
---

**expand_node_randomly()** 메서드는 도우미 함수 **choose_node_expansion()** 을 사용하여 가능한 자식 배열에서 인덱스를 무작위로 선택한다. (**choose_node_expansion()** 은 서브클래스에서 오버로드할 수 있다.)

```python
import [random](https://docs.python.org/3/library/random.html)

class GrammarFuzzer(GrammarFuzzer):
    def expand_node_randomly(self, node: DerivationTree) -> DerivationTree:
        """Choose a random expansion for `node` and return it"""
        (symbol, children) = node
        assert children is None

        if self.log:
            print("Expanding", all_terminals(node), "randomly")

        # Fetch the possible expansions from grammar...
        expansions = self.grammar[symbol]
        children_alternatives: List[List[DerivationTree]] = [
            self.expansion_to_children(expansion) for expansion in expansions
        ]

        # ... and select a random expansion
        index = self.choose_node_expansion(node, children_alternatives)
        chosen_children = children_alternatives[index]

        # Process children (for subclasses)
        chosen_children = self.process_chosen_children(chosen_children,
                                                       expansions[index])

        # Return with new children
        return (symbol, chosen_children)
```

일반적인 **expand_node()
** 메서드는 나중에 다른 확장 전략을 선택하는 데 사용할 수 있다. 현재로서는 **expand_node_randomly()** 만 사용한다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def expand_node(self, node: DerivationTree) -> DerivationTree:
        return self.expand_node_randomly(node)
```

도우미 함수 **process_chosen_children()**는 아무것도 하지 않는다. 선택한 자식을 처리하기 위해 하위 클래스에 의해 오버로드될 수 있다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def process_chosen_children(self,
                                chosen_children: List[DerivationTree],
                                expansion: Expansion) -> List[DerivationTree]:
        """Process children after selection.  By default, does nothing."""
        return chosen_children
```

---

아래가 **expand_node_randomly()** 이 작동하는 방식이다.

```python
f = GrammarFuzzer(EXPR_GRAMMAR, log=True)

print("Before expand_node_randomly():")
expr_tree = ("<integer>", None)
display_tree(expr_tree)
```
![fuzzer4-10](../src/fuzzer4-10.png)

```python
print("After expand_node_randomly():")
expr_tree = f.expand_node_randomly(expr_tree)
display_tree(expr_tree)
```
![fuzzer4-11](../src/fuzzer4-11.png)

아래 코드를 통해 확실히 이것을 시험해 볼 수 있다.

```python
digit_subtree = expr_tree[1][0]
display_tree(digit_subtree)
```
![fuzzer4-12](../src/fuzzer4-12.png)

```python
print("After expanding the <digit> subtree:")
digit_subtree = f.expand_node_randomly(digit_subtree)
display_tree(digit_subtree)
```
![fuzzer4-13](../src/fuzzer4-13.png)

**&lt;digit&gt;** 이 문법 규칙에 따라 다시 한 자리 숫자로 확장되는 것을 볼 수 있다.

하위 트리 중 하나를 변경했지만, 원래 expr_tree는 영향을 받지 않는다.

```python
display_tree(expr_tree)
```
![fuzzer4-14](../src/fuzzer4-14.png)

**expand_node_randomly()** 는 새로운 (확장된) 트리를 반환하고, 인수로 전달된 트리를 변경하지 않기 때문이다.

---
## Expanding a Tree

이제 단일 노드를 트리의 일부 노드로 확장하기 위한 기능을 적용해 보자. 이를 위해 먼저 트리에 확장되지 않은 노드를 검색해야 한다. **possible_expansions()** 는 트리에 확장되지 않은 기호가 몇 개 있는지 계산한다.

```python
>>> class GrammarFuzzer(GrammarFuzzer):
	    def possible_expansions(self, node: DerivationTree) -> int:
	        (symbol, children) = node
	        if children is None:
	            return 1

	        return sum(self.possible_expansions(c) for c in children)

>>> f = GrammarFuzzer(EXPR_GRAMMAR)
>>> print(f.possible_expansions(derivation_tree))
2
```

**any_possible_expansions()** 메서드는 트리에 확장되지 않은 노드가 있는 경우 **True**를 반환한다.

```python
>>> class GrammarFuzzer(GrammarFuzzer):
	    def any_possible_expansions(self, node: DerivationTree) -> bool:
	        (symbol, children) = node
	        if children is None:
	            return True

	        return any(self.any_possible_expansions(c) for c in children)

>>> f = GrammarFuzzer(EXPR_GRAMMAR)
>>> f.any_possible_expansions(derivation_tree)
True
```

다음은 트리 확장 알고리즘의 핵심 방법인 **expand_tree_once()** 메서드다. 먼저 확장 없이 현재 비단말 기호에 적용되고 있는지 확인한다. 만약 그렇다면 위에서 논의한 대로 **expand_node()** 를 호출한다.

노드가 이미 확장된 경우(즉, 자식이 있는 경우), 여전히 확장되지 않은 기호가 있는 자식의 하위 집합을 확인하고, 그 중 하나를 무작위로 선택하고, 해당 자식에 재귀적으로 적용한다.

### expand_tree_once() 구현
---

**expand_tree_once()** 메서드는 children의 자리 그대로를 대체하며, 이는 실제로 새 트리를 반환하는 대신 인수로 전달되는 트리를 변경한다는 것을 의미한다. 이 자체로의 변경은 이 기능을 특히 효율적으로 만드는 것이다. 다시 말하지만, 헬퍼 메서드**(choose_tree_expansion())** 를 사용하여 확장할 수 있는 자식 목록에서 선택한 인덱스를 반환한다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def choose_tree_expansion(self,
                              tree: DerivationTree,
                              children: List[DerivationTree]) -> int:
        """Return index of subtree in `children` to be selected for expansion.
 Defaults to random."""
        return random.randrange(0, len(children))

    def expand_tree_once(self, tree: DerivationTree) -> DerivationTree:
        """Choose an unexpanded symbol in tree; expand it.
 Can be overloaded in subclasses."""
        (symbol, children) = tree
        if children is None:
            # Expand this node
            return self.expand_node(tree)

        # Find all children with possible expansions
        expandable_children = [
            c for c in children if self.any_possible_expansions(c)]

        # `index_map` translates an index in `expandable_children`
        # back into the original index in `children`
        index_map = [i for (i, c) in enumerate(children)
                     if c in expandable_children]

        # Select a random child
        child_to_be_expanded = \
            self.choose_tree_expansion(tree, expandable_children)

        # Expand in place
        children[index_map[child_to_be_expanded]] = \
            self.expand_tree_once(expandable_children[child_to_be_expanded])

        return tree
```

---

위에서 사용한 파생 트리를 사용하여 **expand_tree_once()** 가 어떻게 작동하는지 알아보자. 

```python
>>> derivation_tree = (
		"<start>", 
		[
			(
				"<expr>", 
				[
					("<expr>", None),
                    (" + ", []),
                    ("<term>", None)
                ]
            )
        ]
    )
    
>>> display_tree(derivation_tree)
```
![fuzzer4-15](../src/fuzzer4-15.png)

그 후 두번 확장하게 된다.
```python
>>> f = GrammarFuzzer(EXPR_GRAMMAR, log=True)
>>> derivation_tree = f.expand_tree_once(derivation_tree)
>>> display_tree(derivation_tree)
```
![fuzzer4-16](../src/fuzzer4-16.png)

```python
>>> derivation_tree = f.expand_tree_once(derivation_tree)
>>> display_tree(derivation_tree)
```
![fuzzer4-17](../src/fuzzer4-17.png)

각 단계에서 하나의 기호가 더 확장되는 것을 볼 수 있다. 이제 필요한 것은 이것을 반복해서 적용하여 트리를 점점 더 확장하는 것이다.

---
## Closing the Expansion

**expand_tree_once()** 를 사용하면 트리를 계속 확장할 수 있지만 실제로 어떻게 멈출 수 있을까? 루크가 **[Luke et al, 2000]** 에 도입한 핵심 아이디어는 파생 트리를 최대 크기로 팽창시킨 후, 트리의 크기를 최소한으로 증가시키는 확장만 적용하고 싶다는 것이다. 예를 들어, **&lt;factor&gt;** 의 경우, 더 이상의 재귀(및 잠재적인 크기 인플레이션)를 도입하지 않기 때문에 **&lt;integer&gt;** 로의 확장을 선호한다. 마찬가지로 **&lt;integer&gt;** 의 경우 **&lt;digit&gt;&lt;integer&gt;** 보다 트리 크기를 덜 증가시키기 때문에 **&lt;digit&gt;** 로의 확장이 선호된다.

기호 확장 비용을 식별하기 위해, 서로 의존하는 두 가지 함수를 알아보자.

* **symbol_cost()** 함수는 각 확장 비용을 계산하기 위해 **expansion_cost()** 를 사용하여 기호의 모든 확장의 최소 비용을 반환한다.
* **expansion_cost()** 함수는 **expansion** 의 모든 확장의 합계를 반환한다. 트리 탐색  중에 비단말을 다시 만나면, 확장 비용은 ∞이고, (잠재적으로 무한한) 재귀를 나타낸다.

### 비용 함수 구현 
---

```python
class GrammarFuzzer(GrammarFuzzer):
    def symbol_cost(self, symbol: str, seen: Set[str] = set()) \
            -> Union[int, float]:
        expansions = self.grammar[symbol]
        return min(self.expansion_cost(e, seen | {symbol}) for e in expansions)

    def expansion_cost(self, expansion: Expansion,
                       seen: Set[str] = set()) -> Union[int, float]:
        symbols = nonterminals(expansion)
        if len(symbols) == 0:
            return 1  # no symbol

        if any(s in seen for s in symbols):
            return float('inf')

        # the value of a expansion is the sum of all expandable variables
        # inside + 1
        return sum(self.symbol_cost(s, seen) for s in symbols) + 1
```

---

아래에 두 가지 예가 있다. 확장 중 하나를 선택해야 하기 때문에 숫자를 확장하는 최소 비용은 1이다.

```python
>>> f = GrammarFuzzer(EXPR_GRAMMAR)
>>> assert f.symbol_cost("<digit>") == 1
```

하지만 **&lt;expr&gt;** 확장의 최소 비용은 필요한 최소 확장 수이기 때문에 5이다. **(&lt;expr&gt; → &lt;term&gt; → &lt;factor&gt; → &lt;integer&gt; → &lt;digit&gt; → 1)**

```python
>>> assert f.symbol_cost("<expr>") == 5
```

위의 비용을 고려한 **expand_node()** 의 변형인 **expand_node_by_cost(self, node, choose)** 를 정의하자. 모든 자식의 최소 비용 **cost** 를 결정한 다음 기본적으로 최소 비용인 선택 기능을 사용하여 리스트에서 자식을 선택한다. 여러 자식들이 모두 같은 최소 비용을 가지고 있다면, 이들 사이에서 무작위로 선택한다.

### expand_node_by_cost() 구현
---

```python
class GrammarFuzzer(GrammarFuzzer):
    def expand_node_by_cost(self, node: DerivationTree, 
                            choose: Callable = min) -> DerivationTree:
        (symbol, children) = node
        assert children is None

        # Fetch the possible expansions from grammar...
        expansions = self.grammar[symbol]

        children_alternatives_with_cost = [(self.expansion_to_children(expansion),
                                            self.expansion_cost(expansion, {symbol}),
                                            expansion)
                                           for expansion in expansions]

        costs = [cost for (child, cost, expansion)
                 in children_alternatives_with_cost]
        chosen_cost = choose(costs)
        children_with_chosen_cost = [child for (child, child_cost, _) 
                                     in children_alternatives_with_cost
                                     if child_cost == chosen_cost]
        expansion_with_chosen_cost = [expansion for (_, child_cost, expansion)
                                      in children_alternatives_with_cost
                                      if child_cost == chosen_cost]

        index = self.choose_node_expansion(node, children_with_chosen_cost)

        chosen_children = children_with_chosen_cost[index]
        chosen_expansion = expansion_with_chosen_cost[index]
        chosen_children = self.process_chosen_children(
            chosen_children, chosen_expansion)

        # Return with a new list
        return (symbol, chosen_children)
```

---

**expand_node_min_cost()** 은 간단하게 **min()** 을 **choose** 함수로 전달하여 최소 비용으로 노드를 확장할 수 있다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def expand_node_min_cost(self, node: DerivationTree) -> DerivationTree:
        if self.log:
            print("Expanding", all_terminals(node), "at minimum cost")

        return self.expand_node_by_cost(node, min)
```

이제 위의 **expand_node_min_cost()** 를 확장 함수로 사용하여, 파생 트리의 확장을 닫을 때 이 함수를 적용할 수 있다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def expand_node(self, node: DerivationTree) -> DerivationTree:
        return self.expand_node_min_cost(node)
```

```python
>>> f = GrammarFuzzer(EXPR_GRAMMAR, log=True)
>>> display_tree(derivation_tree)
```
![fuzzer4-18](../src/fuzzer4-18.png)

```python
>>> if f.any_possible_expansions(derivation_tree):
	    derivation_tree = f.expand_tree_once(derivation_tree)

>>> display_tree(derivation_tree)
```
![fuzzer4-19](../src/fuzzer4-19.png)

```python
>>> if f.any_possible_expansions(derivation_tree):
	    derivation_tree = f.expand_tree_once(derivation_tree)

>>> display_tree(derivation_tree)
```
![fuzzer4-20](../src/fuzzer4-20.png)

```python
>>> if f.any_possible_expansions(derivation_tree):
	    derivation_tree = f.expand_tree_once(derivation_tree)
>>> display_tree(derivation_tree)
```
![fuzzer4-21](../src/fuzzer4-21.png)

모든 비단말 기호가 확장될 때까지 계속 확장한다.

```python
>>> while f.any_possible_expansions(derivation_tree):
			derivation_tree = f.expand_tree_once(derivation_tree)    

Expanding <integer> at minimum cost
Expanding <digit> at minimum cost
Expanding <factor> at minimum cost
Expanding <integer> at minimum cost
Expanding <digit> at minimum cost
Expanding <expr> at minimum cost
Expanding <term> at minimum cost
Expanding <term> at minimum cost
Expanding <factor> at minimum cost
Expanding <factor> at minimum cost
Expanding <integer> at minimum cost
Expanding <digit> at minimum cost
Expanding <integer> at minimum cost
Expanding <digit> at minimum cost
```

아래가 최종적인 트리다.
```python
>>> display_tree(derivation_tree)
```
![fuzzer4-22](../src/fuzzer4-22.png)

각 단계에서 **expand_node_min_cost()** 는 기호 수를 늘리지 않는 확장을 선택하고 결국 열려 있는 모든 확장을 닫는다.

---
## Node Inflation

특히 확장이 시작될 때, 우리는 가능한 한 많은 노드를 얻는 데 관심이 있을 수 있다. 즉, 더 많은 비단말 기호를 확장할 수 있는 **expansion** 을 선호한다. 이것은 실제로 **expand_node_min_cost()** 가 제공하는 것과 정반대이며, 항상 가장 높은 비용으로 노드 중에서 선택할 **expand_node_max_cost()** 메서드를 구현할 수 있다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def expand_node_max_cost(self, node: DerivationTree) -> DerivationTree:
        if self.log:
            print("Expanding", all_terminals(node), "at maximum cost")

        return self.expand_node_by_cost(node, max)
```

**expand_node_max_cost()** 를 구현하기 위해, **expand_node()** 를 다시 재정의한 다음 **expand_tree_once()** 를 사용하여 몇 가지 확장 단계를 표시할 수 있다.

```python
>>> class GrammarFuzzer(GrammarFuzzer):
	    def expand_node(self, node: DerivationTree) -> DerivationTree:
	        return self.expand_node_max_cost(node)

>>> derivation_tree = (
		"<start>",
		[
			(
				"<expr>",
                [
	                ("<expr>", None),
                    (" + ", []),
                    ("<term>", None)
                ]
            )
        ]
    )

>>> f = GrammarFuzzer(EXPR_GRAMMAR, log=True)
>>> display_tree(derivation_tree)
```
![fuzzer4-23](../src/fuzzer4-23.png)

```python
>>> if f.any_possible_expansions(derivation_tree):
	    derivation_tree = f.expand_tree_once(derivation_tree)
>>> display_tree(derivation_tree)
```
![fuzzer4-24](../src/fuzzer4-24.png)

```python
>>> if f.any_possible_expansions(derivation_tree):
	    derivation_tree = f.expand_tree_once(derivation_tree)
>>> display_tree(derivation_tree)
```
![fuzzer4-25](../src/fuzzer4-25.png)

```python
>>> if f.any_possible_expansions(derivation_tree):
	    derivation_tree = f.expand_tree_once(derivation_tree)
>>> display_tree(derivation_tree)
```
![fuzzer4-26](../src/fuzzer4-26.png)

위로 미루어 보아 각 단계에서 비단말 기호의 수가 증가한다는 것을 알 수 있다. 분명히, 우리는 이 숫자에 제한을 두어야 한다.

---
## Three Expansion Phases

이제 다음과 같이 작동할 단일 함수 **expand_tree()** 에 세 단계를 모두 모을 수 있다.

1. **Max cost expansion.** 최소 **min_nonterminals** 비단말이 될 때까지 최대 비용으로 확장을 사용하여 트리를 확장한다. 이 단계는 **min_nonterminals** 를 0으로 설정하여 쉽게 건너뛸 수 있다.
2. **Random expansion.** **max_nonterminals** 비단말에 도달할 때까지 트리를 무작위로 계속 확장한다.
3. **Min cost expansion.** 최소 비용으로 확장을 닫는다.

우리는 **expand_node** 가 적용할 확장 메서드를 참조하도록 함으로써 이 세 단계를 구현한다. 이것은 **expand_node(메소드 참조)** 를 먼저 **expand_node_max_cost** 로 설정하여고 (즉, **expand_node()** 를 호출하면 **expand_node_max_cost()** 를 호출한다.) 그런 다음 **expand_node_randomly**, 그리고 마지막으로 **expand_node_min_cost** 로 설정하여 제어된다. 처음 두 단계에서 각각 **min_nonterminals** 와 **max_nonterminals** 의 최대 한도를 설정했다.

### expand_tree() 세 단계 구현
---

```python
class GrammarFuzzer(GrammarFuzzer):
    def log_tree(self, tree: DerivationTree) -> None:
        """Output a tree if self.log is set; if self.display is also set, show the tree structure"""
        if self.log:
            print("Tree:", all_terminals(tree))
            if self.disp:
                display(display_tree(tree))
            # print(self.possible_expansions(tree), "possible expansion(s) left")

    def expand_tree_with_strategy(self, tree: DerivationTree,
                                  expand_node_method: Callable,
                                  limit: Optional[int] = None):
        """Expand tree using `expand_node_method` as node expansion function
 until the number of possible expansions reaches `limit`."""
        self.expand_node = expand_node_method
        while ((limit is None
                or self.possible_expansions(tree) < limit)
               and self.any_possible_expansions(tree)):
            tree = self.expand_tree_once(tree)
            self.log_tree(tree)
        return tree

    def expand_tree(self, tree: DerivationTree) -> DerivationTree:
        """Expand `tree` in a three-phase strategy until all expansions are complete."""
        self.log_tree(tree)
        tree = self.expand_tree_with_strategy(
            tree, self.expand_node_max_cost, self.min_nonterminals)
        tree = self.expand_tree_with_strategy(
            tree, self.expand_node_randomly, self.max_nonterminals)
        tree = self.expand_tree_with_strategy(
            tree, self.expand_node_min_cost)

        assert self.possible_expansions(tree) == 0

        return tree
```

---

아래 예시에서 이것을 시도해 보다. 우리는 반쯤 확장된 파생 트리로 시작한다.

```python
initial_derivation_tree: DerivationTree = (
	"<start>",
	[
		(
			"<expr>",
			[
				("<expr>", None),
                (" + ", []),
                ("<term>", None)
            ]
        )
    ]
)
```

```python
>>> display_tree(initial_derivation_tree)
```
![fuzzer4-27](../src/fuzzer4-27.png)

이제 이 트리에 확장 전략을 적용한다. 처음에 노드가 최대 비용으로 확장되고, 무작위로 확장한 다음 최소 비용으로 확장을 닫는 것을 알 수 있다.

```python
>>> f = GrammarFuzzer(
	    EXPR_GRAMMAR,
	    min_nonterminals=3,
	    max_nonterminals=5,
	    log=True)

>>> derivation_tree = f.expand_tree(initial_derivation_tree)
Tree: <expr> + <term>
Expanding <term> at maximum cost
Tree: <expr> + <factor> / <term>
Expanding <expr> randomly
Tree: <term> + <expr> + <factor> / <term>
Expanding <factor> randomly
Tree: <term> + <expr> + -<factor> / <term>
Expanding <term> randomly
Tree: <factor> + <expr> + -<factor> / <term>
Expanding <factor> randomly
Tree: -<factor> + <expr> + -<factor> / <term>
Expanding <factor> randomly
Tree: --<factor> + <expr> + -<factor> / <term>
Expanding <term> randomly
Tree: --<factor> + <expr> + -<factor> / <factor> / <term>
Expanding <factor> at minimum cost
Tree: --<factor> + <expr> + -<integer> / <factor> / <term>
Expanding <term> at minimum cost
Tree: --<factor> + <expr> + -<integer> / <factor> / <factor>
Expanding <expr> at minimum cost
Tree: --<factor> + <term> + -<integer> / <factor> / <factor>
Expanding <term> at minimum cost
Tree: --<factor> + <factor> + -<integer> / <factor> / <factor>
Expanding <factor> at minimum cost
Tree: --<factor> + <factor> + -<integer> / <factor> / <integer>
Expanding <factor> at minimum cost
Tree: --<factor> + <integer> + -<integer> / <factor> / <integer>
Expanding <integer> at minimum cost
Tree: --<factor> + <digit> + -<integer> / <factor> / <integer>
Expanding <factor> at minimum cost
Tree: --<integer> + <digit> + -<integer> / <factor> / <integer>
Expanding <factor> at minimum cost
Tree: --<integer> + <digit> + -<integer> / <integer> / <integer>
Expanding <integer> at minimum cost
Tree: --<integer> + <digit> + -<digit> / <integer> / <integer>
Expanding <digit> at minimum cost
Tree: --<integer> + 1 + -<digit> / <integer> / <integer>
Expanding <digit> at minimum cost
Tree: --<integer> + 1 + -3 / <integer> / <integer>
Expanding <integer> at minimum cost
Tree: --<digit> + 1 + -3 / <integer> / <integer>
Expanding <digit> at minimum cost
Tree: --0 + 1 + -3 / <integer> / <integer>
Expanding <integer> at minimum cost
Tree: --0 + 1 + -3 / <digit> / <integer>
Expanding <integer> at minimum cost
Tree: --0 + 1 + -3 / <digit> / <digit>
Expanding <digit> at minimum cost
Tree: --0 + 1 + -3 / 7 / <digit>
Expanding <digit> at minimum cost
Tree: --0 + 1 + -3 / 7 / 0
```

아래가 최종 파생 트리다.

```python
>>> display_tree(derivation_tree)
``` 
![fuzzer4-28](../src/fuzzer4-28.png)

그리고 이게 결과 문자열이다.

```python
>>> all_terminals(derivation_tree)
'--0 + 1 + -3 / 7 / 0'
```

---
## Putting it all Together

이를 바탕으로, 이제 **simple_grammar_fuzzer()** 와 같이 단순히 문법을 받아서 문자열을 생성하는 함수 **fuzz()** 를 정의할 수 있다. 따라서 새로 정의될 **fuzz()** 는 더 이상 파생 나무의 복잡성을 드러내지 않는다.

```python
class GrammarFuzzer(GrammarFuzzer):
    def fuzz_tree(self) -> DerivationTree:
        """Produce a derivation tree from the grammar."""
        tree = self.init_tree()
        # print(tree)

        # Expand all nonterminals
        tree = self.expand_tree(tree)
        if self.log:
            print(repr(all_terminals(tree)))
        if self.disp:
            display(display_tree(tree))
        return tree

    def fuzz(self) -> str:
        """Produce a string from the grammar."""
        self.derivation_tree = self.fuzz_tree()
        return all_terminals(self.derivation_tree)
```

이제 이것을 정의된 모든 문법에 적용할 수 있다. (그리고 파생 트리를 시각화할 수 있다)

```python
>>> f = GrammarFuzzer(EXPR_GRAMMAR)
>>> f.fuzz()
'++2.937 * -7 * ++-6'
```

**fuzz()** 를 호출한 후, 생성된 파생 트리는 **derivation_tree** 속성에서 액세스할 수 있다.

```python
>>> display_tree(f.derivation_tree)
```
![fuzzer4-29](../src/fuzzer4-29.png)

다른 문법 형식으로 문법 퍼저(트리)를 사용해 보자.
```python
>>> f = GrammarFuzzer(URL_GRAMMAR)
>>> f.fuzz()
'ftps://user:password@cispa.saarland:04/x74?x64=x64'
>>> display_tree(f.derivation_tree)
```
![fuzzer4-30](../src/fuzzer4-30.png)

```python
>>> f = GrammarFuzzer(CGI_GRAMMAR, min_nonterminals=3, max_nonterminals=5)
>>> f.fuzz()
'%18+0'
>>> display_tree(f.derivation_tree)
```
![fuzzer4-31](../src/fuzzer4-31.png)


**simple_grammar_fuzzer()** 와 성능 차이에서 어떻게 다른지 확인해보자

```python
>>> trials = 50
>>> xs = []
>>> ys = []
>>> f = GrammarFuzzer(EXPR_GRAMMAR, max_nonterminals=20)
>>> for i in range(trials):
	    with Timer() as t:
	        s = f.fuzz()
	    xs.append(len(s))
	    ys.append(t.elapsed_time())
	    print(i, end=" ")

0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 
>>> average_time = sum(ys) / trials
>>> print("Average time:", average_time)
Average time: 0.016987550000000056
```

```python
%matplotlib inline

import [matplotlib.pyplot](https://matplotlib.org/) as plt
plt.scatter(xs, ys)
plt.title('Time required for generating an output');
```
![fuzzer4-32](../src/fuzzer4-32.png)

우리의 테스트 생성은 훨씬 빠르지만, 우리의 입력도 훨씬 작다. 파생 트리를 통해 문법 생산을 훨씬 더 잘 통제할 수 있다는 것을 알 수 있다.

마지막으로, **GrammarFuzzer** 는 **simple_grammar_fuzzer()** 가 실패한 **expr_grammar** 문법으로도 아무런 문제 없이 작동한다.

```python
>>> f = GrammarFuzzer(expr_grammar, max_nonterminals=10)
>>> f.fuzz()
'4 + (3 - 2 + 4) * 8 * 6 - 4 / 6 * 0'
```

**GrammarFuzzer** 를 통해, 우리는 이제 더 많은 퍼저를 구축하고 소프트웨어 테스트를 생성하는 세계에서 더 흥미로운 개념을 설명할 수 있는 견고한 기반을 갖게 되었다. 이들 중 다수는 문법을 작성할 필요조차 없다. 대신, 그들은 당면한 도메인에서 문법을 추론하여 문법을 작성하지 않고도 문법 기반 퍼징을 사용할 수 있다.

