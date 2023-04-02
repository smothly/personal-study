# CH2 시퀀스

- 문자열, 리스트, 바이트 시퀀스 등에 반복, 슬라이싱, 정렬, 연결 등을 공통된 연산을 적용할 수 있음

## 2.1 내장 시퀀스 개요

---

- 참조에 따른 분류
  - 컨테이너 시퀀스
    - 다른 자료형 항목 담을 수 있음 ex) list, tuple, deque 등
    - 객체에 대한 참조를 담고 있음
  - 균일 시퀀스
    - 같은 자료형만 담을 수 있음 ex) str, bytes, array 등
    - 값을 직점 담고, 메모리를 더 적게 사용함
- 가변성에 따른 분류
  - 가변 ex) list, array, deque
  - 불변 ex) tuple, str, bytes

## 2.2 list comprehension & generator expression

---

- list comprehension이 더 보기 좋음
- 물론 두줄이상 넘어가는 반복문이나 복잡한 조건이 들어가면 분리하는 게 나음. 무조건적이지는 않음

```python
symbols = '$¢£¥€¤'\n"
beyond_ascii = [ord(s) for s in symbols if ord(s) > 127]
```

### list comprehension 과 map/filter 비교

- 시간차이는 없음. 가독성은 list comprehension이 더 좋음

```python
list(filter(lambda c: c > 127, map(ord, symbols)))
```

### 제너레이터 표현식

- 리스트를 통째로 만들지 않고 iterator protocol을 이용해 하나씩 생성하는 제너레이터 표현식은 메모리를 더 적게 사용
- 대괄호 [ 대신 괄호 ( 를 사용함
- 데카르트 곱 제너레이터 예시

```python
for tshirt in ('%s %s' % (c, s) for c in colors for s in sizes):
    print(tshirt)
```

## 2.3 튜플은 단순한 불변 리스트가 아니다

---

- 불변 리스트뿐만 아니라 필드명이 없는 레코드로 사용할 수 있음
- 레코드로서의 튜플

```python
traveler_ids = [('USA', '31195855'), ('BRA', 'CE342567'), ('ESP', 'XDA205856')]
```

- tuple unpacking하여 가져올 수 있음
  - 초과된 항목을 *을 통해 가져올 수도 있음
  - 내포된 tuple을 unpacking할 수 있음

```python
a, *body, c, d = range(5)

metro_areas = [
    ('Tokyo', 'JP', 36.933, (35.689722, 139.691667)),
    ('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889)),
    ('Mexico City', 'MX', 20.142, (19.433333, -99.133333)),
    ('New York-Newark', 'US', 20.104, (40.808611, -74.020386)),
    ('São Paulo', 'BR', 19.649, (-23.547778, -46.635833)),
]

def main():
    print(f'{"":15} | {"latitude":>9} | {"longitude":>9}')
    for record in metro_areas:
        match record:  # <1>
            case [name, _, _, (lat, lon)] if lon <= 0:  # <2>
                print(f'{name:15} | {lat:9.4f} | {lon:9.4f}')
```

- `collections.namedtuple`
  - 인덱스 뿐만 아니라 dict처럼 키값으로도 접근할 수 있음

```python
# 출처: https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=wideeyed&logNo=221682049802
def calc(x, y):
    """두수를 입력받아 합, 곱, 나눈값을 반환한다 """
    return namedtuple(typename='return_calc', field_names=['sum', 'multiplication', 'division'])(x + y, x * y, x / y)

rr = calc(3, 2)
print(rr._fields) # 키(=필드명)들을 확인할 수 있다 => ('sum', 'multiplication', 'division')
dd = rr._asdict() # OrderedDict형으로 변환하여 사용할 수도 있다
print('곱셈결과:', dd['multiplication']) # 곱셈한 결과를 출력한다
```

- 불변 리스트로서의 튜플
  - 리스트와 달리 reverse, append, pop, remove 등의 함수를 지원하지 않음

## 2.4 슬라이싱

---

- `s[a:b:c]`는 c만큼 stride(보폭) 건너뛴다는 의미
- 슬라이스에 따라 할당하는 여러가지 방법

```python
# 정렬된 카드에서 에이스 카드만 추려낼 때 사용하는 예시
deck[12::13]

l[2:5] = [100]
```

## 2.5 시퀀스에 덧셈과 곱셈 연산자 사용하기

---

- 곱셈 연산자를 잘못 생성할 경우, 동일한 리스트에 대해 같은 참조를 가지게 됨

```python
board = [['_'] * 3 for i in range(3)]

weird_board = [['_'] * 3] * 3
weird_board[1][2] = '0'
# [['_', '_', 'O'], ['_', '_', 'O'], ['_', '_', 'O']]
```

## 2.6 시퀀스의 복합 할당

---

- `+=` == `__i_add__()`
  - i는 **inplace**를 의미함
    - iadd는 a의 값이 변경되는데(id값 동일), add는 새로운 객체를 a에 할당함
  - 만약 `__iadd__()`가 구현되지 않았을 경우 `add`를 수행함
- 불변 시퀀스의 경우 연결 연산을 하면 `add`연산자가 수행됨. 고로 불변 시퀀스에 연결 연산은 비효율적
- 가변 항목을 튜플에 넣는 것은 좋은 생각이 아님

## 2.7 list.sort()와 sorted()

---

- `sort()`는 내부에서 변경해서 정렬하고 None을 반환
- `sorted()`는 새로운 리스트를 생성해서 반환. 불변 시퀀스 및 제너레이터도 가능
- 두 함수 다 parameter는 똑같음

```python
sorted(fruits, key=len, reverse=True)
```

## 2.8 정렬된 시퀀스를 bisect로 관리하기

---

- `bisect()`는 이진 검색 알고리즘이고 `insort()`는 정렬된 시퀀스안에 항목 삽입
- index보다 더 빠름! O(logN)

```python
from bisect import bisect_left, bisect_right

nums = [4, 5, 5, 5, 5, 5, 5, 5, 5, 6]
n = 5

print(bisect_left(nums, n)) # 1
print(bisect_right(nums, n)) # 9

result = []
for score in [33, 99, 77, 70, 89, 90, 100]:
    pos = bisect.bisect_left([60, 70, 80, 90], score)
    grade = 'FDCBA'[pos]
    result.append(grade)

print(result)

a = [60, 70, 80, 90]
bisect.insort(a, 85)
```

## 2.9 리스트가 답이 아닐 때

---

- float을 천만 개 저장해야할때는 list보다는 `array`가 효율적임
  - C 배열만큼 가벼움
  - 특정 타입에 맞는 항목만 저장 가능
  - 가변시퀀스가 제공하는 연산은 다 제공. `tofile` `fromfile`도 제공
  - `pickle.dump` 의 경우도 array만큼 빠르게 저장 및 복수 등 할 수 있음
  - 정렬된 array에 항목 추가할려면 `bisect.insort()`사용하기
- 메모리 뷰
  - bytes를 복사하지 않고 배열의 슬라이스를 다룰 수 있음
- numpy와 scipy
  - scipy: numpy를 기반으로 선형대수학, 수치해석, 통계확에 나오는 계산 알고리즘 제공
  - numpy: 고급 배열 및 행렬 연산
- 리스트에 양쪽 끝에 항목을 추가/삭제를 하는 FIFO/LIFO의 경우 `deque`가 효과적
  - append와 pop 같은 경우 FIFO방식으로 동작하여 0번 인덱스에 삽입하면 전체 리스트가 이동됨
  - deque는 양방향 큐
  - 중간항목을 삭제하는 연산은 빠르지 않음
  
  ```python
  dq = collections.deque(range(10), maxlen=10)
  dq.extend([11, 22, 33]
  dq.rotate(3)
  ```
- 항목검사 작업이 많다면 `set`

## 2.10 요약

---

- 파이썬 시퀀스
  - 가변형 vs 불변형
  - 균일(작고, 빠르고, 쉽지만 원자적인 데이터) vs 컨테이너(융통성, 가변 객체)
- list comphrension generaotr 유용
- 튜플을 레코드 및 불변 리스트 사용 가능
  - \* 사용하여 언패킹
  - namedtuple 사용. 객체 저장 효율
- 시퀀스 슬라이싱은 뛰어남
- 복할 할당 연산자는 가변일 경우 변경 불변일 경우 새로운 시퀀스 생성
- sort <-> sorted 함수와 key인자를 통한 정렬 방법. feat. bisect
- list대신 사용해야 할 `array.array`(확장된 numpy)와 `collections.deque`
