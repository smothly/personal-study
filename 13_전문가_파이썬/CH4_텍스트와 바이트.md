# CH5 일급 함수

- 파이썬의 함수는 **일급 객체**다.
- 일급 객체의 특징
  - 런타임에 생성할 수 있음
  - 데이터 구조체의 변수나 요소에 할당할 수 있음
  - 함수 인수로 전달할 수 있음
  - 함수 결과로 반환할 수 있음
  - 파이썬 딕셔너리, 문자열 등 전부 일급 객체임

## 5.1 함수를 객체처럼 다루기

---

- `__doc__` 속성은 함수의 독스트링을 가져옴
- 함수의 type이 function인지 확인
- 함수를 다른 이름으로 사용하고 함수의 인수로 전달할 수 있음
  - 함수형 스타일로 프로그래밍 가능 (고위 함수)

```python
fact = factorial
list(map(fact, range(11)))
```

## 5.2 고위 함수

---

- high-order function
  - 함수를 인수로 받거나 함수를 결과로 반환하는 함수
  - map, sorted, filter, reduce 등이 있음
- map, filter 대신 **지능형 리스트와 제너레이터**를 주로 쓰고 있음

```python
list(map(factorial, filter(lambda n: n%2, range(6)))) # Map사용
[factorial(n) for n in range(6) if n % 2] # list comprehension 사용

reduce(add, range(100))
sum(range(100))
```

## 5.3 익명 함수

---

- lambda 키워드로 익명 함수를 생성할 수 있음. 순수한 표현식으로만 구성되어야 해서 while, try 등의 문장을 사용할 수 없음
- 고위 함수의 인수로 사용하는 경우가 대부분임

```python
fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
sorted(fruits, key=lambda word: word[::-1])
```

## 5.4 일곱 가지 맛의 콜러블 객체

---

- 호출할 수 있는 객체인지 확인하는 방법
  - `callable()` 내장 함수를 사용
  - `__call__()` 메서드가 구현되어 있는지 확인
- 7가지 콜러블 객체
  - 사용자 정의 함수
  - 내장 함수
  - 내장 메서드
  - 메서드
  - 클래스
  - 클래스 객체
  - 제너레이터 함수

## 5.5 사용자 정의 콜러블형

---

- 모든 파이썬 객체가 함수처럼 동작하게 만들 수 있음
  - `__call__()` 인스턴스 메서드를 구현하면 됨

```python
"""
# tag::BINGO_DEMO[]

>>> bingo = BingoCage(range(3))
>>> bingo.pick()
1
>>> bingo()
0
>>> callable(bingo)
True

# end::BINGO_DEMO[]

"""

import random

class BingoCage:

  def __init__(self, items):
    self._items = list(items)  # <1>
    random.shuffle(self._items)  # <2>

  def pick(self):  # <3>
    try:
        return self._items.pop()
    except IndexError:
        raise LookupError('pick from empty BingoCage')  # <4>

  def __call__(self):  # <5>
    return self.pick()
```

## 5.6 함수 인트로스펙션

---

- `__dict__` 속성을 확인하면 함수의 사용자 속성을 확인할 수 있음
- IDE에서 함수 시그니처에 대한 정보를 추출할 때 `__defaults__`, `__code__`, `__annotations__` 속성을 사용함

## 5.7 위치 매개변수에서 키워드 전용 매개변수까지

---

- `keyword-only argument`는 위치 매개변수를 지원하지 않는 경우에 유용함
  - 아래 예제에서 cls매개변수는 키워드 인수로만 전달될 수 있고 `positional argument`로 전달되지 않음
  - 키워드 전용 인수로 지정할려면 * 가 붙은 인수 뒤에 지정하면 됨

```python
"""
# tag::TAG_DEMO[]
>>> tag('br')  # <1>
'<br />'
>>> tag('p', 'hello')  # <2>
'<p>hello</p>'
>>> print(tag('p', 'hello', 'world'))
<p>hello</p>
<p>world</p>
>>> tag('p', 'hello', id=33)  # <3>
'<p id="33">hello</p>'
>>> print(tag('p', 'hello', 'world', class_='sidebar'))  # <4>
<p class="sidebar">hello</p>
<p class="sidebar">world</p>
>>> tag(content='testing', name="img")  # <5>
'<img content="testing" />'
>>> my_tag = {'name': 'img', 'title': 'Sunset Boulevard',
...           'src': 'sunset.jpg', 'class': 'framed'}
>>> tag(**my_tag)  # <6>
'<img class="framed" src="sunset.jpg" title="Sunset Boulevard" />'

# end::TAG_DEMO[]
"""


# tag::TAG_FUNC[]
def tag(name, *content, cls=None, **attrs):
    """Generate one or more HTML tags"""
    if cls is not None:
        attrs['class'] = cls
    attr_pairs = (f' {attr}="{value}"' for attr, value
                    in sorted(attrs.items()))
    attr_str = ''.join(attr_pairs)
    if content:
        elements = (f'<{name}{attr_str}>{c}</{name}>'
                    for c in content)
        return '\n'.join(elements)
    else:
        return f'<{name}{attr_str} />'
# end::TAG_FUNC[]
```

## 5.8 매개변수에 대한 정보 읽기

---

- 함수의 매개변수에 대한 정보를 얻는 방법
  - `inspect.signature()` 함수를 사용하면 됨
  - `inspect.Signature` 객체는 매개변수와 관련된 정보를 담고 있음
    - kind속성은 5가지 값을 가짐
      - POSITIONAL_ONLY
      - POSITIONAL_OR_KEYWORD - 위치인수 키워드인수로 전달할 수 있는 매개변수로 대부분 여기에 속함
      - VAR_POSITIONAL
      - KEYWORD_ONLY
      - VAR_KEYWORD
  - `inspect.Parameter` 객체는 매개변수 하나에 대한 정보를 담고 있음

```python
>>> from clip import clip
>>> from inspect import signature
>>> sig = signature(clip)
>>> sig  # doctest: +ELLIPSIS
<inspect.Signature object at 0x...>
>>> str(sig)
'(text, max_len=80)'
>>> for name, param in sig.parameters.items():
...     print(param.kind, ':', name, '=', param.default)
...
POSITIONAL_OR_KEYWORD : text = <class 'inspect._empty'>
POSITIONAL_OR_KEYWORD : max_len = 80
```

## 5.9 함수 애노테이션

---

- 함수 선언에 사용하는 메타데이터
- `inspect.signature()` 함수는 함수의 매개변수에 대한 애노테이션을 읽을 수 있음

```python
def clip(text:str, max_len:'int >0'=80) -> str:
  pass
```

## 5.10 함수형 프로그래밍을 위한 패키지

---

- operator 모듈과 functools 모듈 지원으로 함수형 코딩을 할 수는 있음

```python
from functools import reduce

def fact(n):
  return reduce(lambda a, b: a*b, range(1, n+1))
```

- Lambda구축이 귀찮으니 operator 모듈을 사용하면 됨

```python
from operator import mul
from functools import reduce

def fact(n):
  return reduce(mul, range(1, n+1))
```

- `functools.partial()` 함수를 사용하면 함수의 인수를 고정할 수 있음

```python
from operator import mul
from functools import partial
triple = partial(mul, 3)
list(map(triple, range(1, 10)))
```

## 5.11 요약

- 파이썬 함수의 일급 특성으로, 기본 개념은 함수를 변수에 할당하고, 다른 함수의 인수로 전달하고, 함수의 결과로 반환할 수 있다는 것.
또한, 이 함수 속성에 접근해서 속성 정보를 활용할 수 있음
- callable한 함수는 `lambda`부터 `__call__`메서드를 가진 클래스 객체까지 다양함
- `inspect` 모듈은 함수의 매개변수에 대한 정보를 읽을 수 있음
- `operator` 모듈은 람다 표현식을 대체할 수 있는 함수를 제공함. `functools` 모듈은 함수형 프로그래밍을 위한 도구를 제공함