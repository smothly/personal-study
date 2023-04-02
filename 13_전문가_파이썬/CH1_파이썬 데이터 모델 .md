# CH1 파이썬 데이터 모델

- `obj.len()`이 아닌 `len(obj)`를 사용함
- `obj[key]`는 `__get_item__()`이라는 특별 메소드(**매직 메소드)**를 지원함

## 1.1 파이썬 카드 한 벌

---

- 매직 메소드를 사용할 때 장점
  - 1. 클래스 자체에서 구현한 임의 메소드명을 암기할 필요가 없음 ex) size
  - 2. `random.choice()`와 같이 많은 기능을 직접 구현할 필요가 없음
  - 3. `__get_item__()`메소드에서 slicing, reverse, sort 등 자동 지원

## 1.2 특별 메소드는 어떻게 사용되나?

---

- 특별 메소드를 직접 정의하여 사용할 수 있지만 흔치 않음
  - `for i in x:` 문은 `iter(x)`
  - `len()`같은 함수가 내부적으로 파이썬 인터프리터가 특별 메소드를 호출함
- 자주 호출하는 것은 `__init__`만 있음
- 벡터 연산을 위해 직접 특별 메소드를 구축한 예시
  - str보다는 repr을 구현해라
  - 산술연산자는 새 객체를 만들어라
  - bool이 구현되어있지 않으면 len을 호출 함

    ```python
    import math

    class Vector:

        def __init__(self, x=0, y=0):
            self.x = x
            self.y = y

        def __repr__(self):
            return f'Vector({self.x!r}, {self.y!r})'

        def __abs__(self):
            return math.hypot(self.x, self.y)

        def __bool__(self):
            return bool(abs(self))

        def __add__(self, other):
            x = self.x + other.x
            y = self.y + other.y
            return Vector(x, y)

        def __mul__(self, scalar):
            return Vector(self.x * scalar, self.y * scalar)
    ```

## 1.3 특별 메서드 개요

---

[데이터 모델에서 특별 메소드 나열](https://docs.python.org/3/reference/datamodel.html)

## 1.4 왜 len()은 메소드가 아닐까?

---

- 파이썬은 실용성 > 순수성을 우선하기 때문에 `abs`나 `len`은 특별한 대우로 내장형 객체의 효율성과 언어의 일관성 간의 타협점을 찾음

## 1.5 요약

---

- 특별 메소드를 구현하면 사용자 정의 객체도 내장형 객체처럼 작동하게 되어, 파이썬 스러운 표현력 있는 코딩 스타일을 구사할 수 있음
- 특별메소드를 활용하는 방법은 책에서 전체적으로 설명함
