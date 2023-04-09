# CH3 딕셔리와 집합

- 해시 테이블이라는 엔진을 사용하여 딕셔니리를 구현

## 3.1  일반적인 매핑형

---

- key만 해싱 가능해야한다는 제한을 dict를 이용해 구현
- **Hashable**의 의미
  - 수명주기동안 변하지 않는 해시값을 가지고 있음(`__hash__()` 메서드)
  - `__eq__()` 메서드로 해시값 비교 가능
- 불변형은 모두 해시 가능, `list`는 불가
- 사용자 정의 자료형은 기본적으로 해시 가능

```python
a = dict(one=1, two=2, three=3)
b = {'three': 3, 'two': 2, 'one': 1}
c = dict([('two', 2), ('one', 1), ('three', 3)])
d = dict(zip(['one', 'two', 'three'], [1, 2, 3]))
e = dict({'three': 3, 'one': 1, 'two': 2})
a == b == c == d == e
```

## 3.2 Dict comprehensions

---

- dict comprehension은 dict를 만드는 간결한 문법

```python
{code: country.upper() 
  for country, code in sorted(country_dial.items())
  if code < 70}
```

## 3.3 공통적인 매핑 메서드

---

- `orderdDict` `defaultDict`가 dict 변형중에서 많이 사용됨
- 존재하지 않는 키를 `setdefault()`로 처리
  - d[key]로 접근시 키가 없으면 에러 발생
  - 일반적으로는 `d.get(key, default)`를 사용
  - 참조횟수를 줄이기 위해 `setdefault()` 많이 사용함

  ```python
  # this is ugly; coded like this to make a point
  occurrences = index.get(word, [])  # <1>
  occurrences.append(location)       # <2>
  index[word] = occurrences          # <3>

  # 아래와 같이 한줄로 처리 가능
  index.setdefault(word, []).append(location)  # <1>
  ```

## 3.4 융통성 있게 키를 조회하는 매핑

---

- defaultdict를 사용
  - 존재하지 않는 키로 `__getitem__()` 메서드를 호출할 때마다 기본값을 생성하기 위해 사용되는 callable을 제공함
  - 기본값을 생성하는 callable은 default_factory라는 객체 속성에 저장됨
  - default_factory를 호출하게 만드는 메커니즘은 `__missing__()` 특수 메서드에 의존
  
  ```python
  index = collections.defaultdict(list)
  index[word].append(location)
  ```

- dict등의 매핑형을 상속해서 `__missing__()` 메서드 사용
  - 존재하지 않는 키를 처리하는 특수 메서드
  - 설정할 경우 `KeyError`를 발생시키지 않고 `__missing__()`를 호출함
  - 주의할 점으로는 `get_item()` 메서드를 호출할 때만 호출되고, get, contains 메소드 등에는 호출되지 않음
  - 검색할 때 키를 str형으로 변환하는 매핑
  
  ```python
  class StrKeyDict0(dict):  # <1>

      def __missing__(self, key):
          if isinstance(key, str):  # <2>
              raise KeyError(key)
          return self[str(key)]  # <3>

      def get(self, key, default=None):
          try:
              return self[key]  # <4>
          except KeyError:
              return default  # <5>

      def __contains__(self, key):
          return key in self.keys() or str(key) in self.keys()  # <6>
  ```

## 3.5 그 외 매핑형

---

- `collections.OrderedDict`
  - 키를 삽입한 순서대로 유지하는 dict
  - popitem() 메서드를 사용하면 마지막에 삽입한 키를 얻을 수 있음. popitem(last=False)로 FIFO처럼 사용 가능
- `collections.ChainMap`
  - 매핑들의 목록을 담고 있으며 한꺼번에 모두 검색 가능
- `collections.Counter`
  - 해시 가능한 객체를 세는 데 사용하는 dict의 서브클래스
  - `most_common()` 메서드를 사용해 가장 많이 등장하는 항목을 찾을 수 있음 
  
  ```python
  ct = collections.Counter('abracadabra')
  ct.update('aaaaazzz')
  ct.most_common(2)
  ```

- `collections.UserDict`
  - 표준 dict를 상속받아 사용자 정의 dict를 만들 수 있음

## 3.6 UserDict를 상속하기

---

- 내장형의 문제없이 override해야하기 때문에 dict보다는 **UserDict를 상속하는 것이 나음**
- userdict는 data라고 하는 dict객체를 갖고 있기 때문에, `__set_item__()` 등의 특수 메서드를 구현할 때 원치않는 재귀적 호출을 피할 수 있음
- `UserDict`클래스가 `MutableMapping`, `Mapping`을 상속하므로 매핑의 모든 기능을 가지게 됨
  - `MutableMapping.update()`는 __init__ 이나 다른 매핑에서도 많이 사용할 수 있고, 내부적으로 self[key] = value이므로 set_item을 호출하도록 되어 있음
  - `Mapping.get()` 도 비슷하게 상속받아 동작함

## 3.7 불변 매핑

---

- 3.3이후 `MappingProxyType`을 사용하면 dict를 불변으로 만들 수 있음. 읽기 전용의 mappingproxy 객체를 반환함

```python
from types import MappingProxyType
d = {1: 'A'}
d_proxy = MappingProxyType(d)
d_proxy[2] = 'x' # TypeError: 'mappingproxy' object does not support item assignment
d[2] = 'B' # no error
```

## 3.8 집합 유형

---

- 집합은 고유한 객체의 모음으로서, 기본적으로 중복항목을 허용하지 않음
- 집합 요소는 반드시 해시할 수 있어야 함. set은 해시 가능하지 않지만 frozenset은 해시 가능하므로 set의 요소로 사용 가능
- 집합 리터럴
  - `{1,2,3}`처럼 선언하는 것이 `set([1,2,3])` 보다 빠르고 가독성이 좋음. 실제로 바이트 코드를 살펴봐도 BUILD_SET이라는 바이트코드가 거의 모든일을 처리하여 빠름
  - frozzenset은 별도의 리터럴 구문이 없어, 생성자를 호출해 생성해야함 ex) `frozenset([1,2,3])`
- set comprehension
  - `{x for x in iterable}`
- 집합 연산과 메서드 지원
  - `s | t` : 합집합
  - `s & t` : 교집합
  - `s - t` : 차집합
  - `s ^ t` : 대칭차집합
  - `s <= t` : 부분집합인지 여부
  - `s.pop()` : 임의의 원소를 제거하고 반환

## 3.9 dict와 set의 내부 구조

---

- 성능 실험
  - dict, set이 list보다 검색시간이 훨씬 빠름!
  - Why?
    - 딕셔너리안의 해시 테이블
      - sparse array로서 버킷에 키에대한 참조와 값에대한 참조가 들어가 버킷의 크기가 동일함
      - 모든 버킷의 크기가 동일하므로, 오프셋을 계산해서 각 버킷에 바로 접근할 수 있음
      - 1/3이상 비워둘려고 하며, 항목이 많아지면 더 넓은 공간을 할당하고 항목을 복사함
    - 해시와 동치성
      - `__hash__()` 메서드는 객체의 해시값을 반환하며, `__eq__()` 메서드는 객체의 동치성을 검사함
      - 1과 1.0의 해시값은 같으나, 1.0001과는 완전히 다름
    - 해시 테이블 알고리즘
      - search_key와 found_key가 다를 때 **hash_collision이 발생**함. 이 때 해시의 다른 비트들을 가져와 조작한 후 다른 버킷을 조회함
      - 해시 테이블에 항목이 너무 많을 경우 새로운 위치에 해시 테이블을 다시 만듬
      - 발생하는 평균 충돌 횟수는 1에서 2이므로 성능에 큰 영향을 미치지 않음
      - ![알고리즘](https://feifeiyum.github.io/images/python/dict_hash.png)
    - 키 객체는 반드시 해시 가능해야 함
      - 수명주기동안 동일한 값을 반환하는 `__hash__()` 메서드를 제공
      - `__eq__()` 메서드를 제공하고, 동일한 객체에 대해 True를 반환
      - **a == b가 참이면, hash(a) == hash(b)도 참**이어야 함
    - 사용자 정의 자료형은 `id()`를 해시값으로 사용함
- dict의 메모리 오버헤드
  - 해시가 제대로 동작하려면, 해시 테이블의 크기가 충분히 커야함
  - 많은양의 데이터일 경우 튜플로 교체하는게 더 효율적일 수 있음
  - 일반적인 상황에서는 최적화가 필요하지는 않음
- 키 검색이 빠름
- 키 순서는 삽입 순서에 따라 달라짐
  - 충돌이 발생하느냐 안하냐에 따라 차이가 발생
  - 순서가 달라도 동일한 dict라고 판단
  - dict 항목을 추가하면 기존키의 순서가 변경될 수 있으므로, dict 순회중에는 항목을 추가하지 않는 것이 좋음
- 집합의 동작방식
  - set도 해시테이블을 구현하지만, **각 버킷이 항목에 대한 참조**만을 담고 있음
  - 항목 자체가 dict에서의 key처럼 사용되지만 이 키를 통해 접근할 값이 없음. dict는 키와 값을 모두 저장하지만, set은 키만 저장함
  - dict와 기본적인 작동 방식은 같음
    - set요소는 반드시 해시 가능해야 함
    - 메모리 오버헤드는 dict와 동일
    - 항목 순서는 collison에 따라 달라짐  

## 3.10 요약

---

- dict
  - 간편한 매핑형
    - defaultdict
    - ordereddict
    - chainmap
    - counter
    - 확장 가능한 userdict
  - 강력 메소드 제공
    - setdefault(): 키가 존재하면 값을 반환하고, 없으면 기본값을 저장하고 반환
    - update(): 데이터 추가 덮어쓰기 가능
    - missing은 키가 없을 때 호출되는 메서드
  - 추상 클래스 제공
    - mapping
    - mutablemapping
    - mappingproxytype은 불변형 매핑 생성
  - 해시 테이블
    - 속도가 빠른 반면 메모리를 많이 사용
