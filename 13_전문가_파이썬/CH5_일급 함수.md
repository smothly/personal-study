# CH4 텍스트와 바이트

- 문자열(str)과 바이트(bytes)는 구분해야 함

## 4.1 문자 문제

---

- 문자를 가장 잘 정의한 것은 유니코드(Unicode) 문자
- 유니코드 표준은 문자의 단위 원소와 특정 바이트 표현을 명확히 구분
  - code point(문자의 단위 원소)는 0~1,114,111(16진수 0x10FFFF) 사이의 정수로, 'U+'를 접두사로 붙여 표현
  - 문자를 표현하는 실제 바이트는 사용하는 인코딩에 따라 다름
    - 인코딩 = 코드 포인트를 바이트 시퀀스로 변환하는 알고리즘
    - 코드 포인트 -> 바이트: **인코딩**
    - 바이트 -> 코드 포인트: **디코딩**

```python
s = 'café'
len(s) # 4
b = s.encode('utf8')
b
# b'caf\xc3\xa9'
len(b) # 5  é는 2바이트로 인코딩됨
b.decode('utf8')
# 'café'
```

## 4.2 바이트에 대한 기본 지식

---

- 이진 시퀀스를 위해 사용되는 내장 타입은 bytes와 bytearray 두 가지가 있음
- bytes는 파이썬 3에소 소개된 불변형이고, bytearray는 python2.6에서 추가된 가변형
- 각 항목은 0~225사이의 정수로, bytearray는 리터럴 구문은 없고 슬라이싱 해도 bytearray를 반환

```python
cafe = bytes('café', encoding='utf_8')
cafe
# b'caf\xc3\xa9'
cafe[0]
# 99
cafe[:1]
# b'c'
cafe_arr = bytearray(cafe)
cafe_arr
cafe_arr[-1:]
# bytearray(b'\xa9')
```

- bytes나 bytearray로 객체를 생성하면 언제나 바이트를 복사함. MemoryView는 바이트를 복사하지 않고 메모리를 공유함
- 구조체와 메모리 뷰
  - 바이트를 복사하지 않고 메모리를 공유하는 메모리 뷰는 바이트 시퀀스를 구조체로 해석할 수 있음
- memoryview를 slicing하면 바이트를 복사하지 않고 새로운 memoryview를 생성함

```python
import struct
fmt = '<3s3sHH'
with open('filter.gif', 'rb') as fp:
    img = memoryview(fp.read())
header = img[:10]
bytes(header)
# b'GIF89a\x01\x00\x01\x00\x80\x00\x00'
struct.unpack(fmt, header)
# (b'GIF', b'89a', 1, 1)
del header
del img
```

## 4.3 기본 인코더/디코더

---

- 파이썬은 100여개의 코덱이 포함됨

```python
for codec in ['latin_1', 'utf_8', 'utf_16']:
    print(codec, 'El Niño'.encode(codec), sep='\t')
# latin_1	b'El Ni\xf1o'
# utf_8	b'El Ni\xc3\xb1o'
# utf_16 b'\xff\ .... 
```

## 4.4 인코딩/디코딩 문제 이해하기

---

- UnicodeEncodeError: 인코딩할 수 없는 문자를 인코딩하려고 할 때 발생
- UnicodeEncodeError 처리 방법

```python
city = 'São Paulo'
city.encode('utf_8')
city.encode('utf_16')
city.encode('iso8859_1')
city.encode('cp437')
# UnicodeEncodeError: 'charmap' codec can't encode character '\xe3' in position 1: character maps to <undefined>
city.encode('cp437', errors='ignore')
# b'So Paulo'
city.encode('cp437', errors='replace')
city.encode('cp437', errors='xmlcharrefreplace') # xml객체로 치환
```

- UnicodeDecodeError: 디코딩할 수 없는 바이트를 디코딩하려고 할 때 발생
- UnicodeDecodeError 처리 방법

```python
octets = b'Montr\xe9al'
octets.decode('cp1252')
octets.decode('iso8859_7')
octets.decode('koi8_r')
octets.decode('utf_8')
# UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 5: invalid continuation byte
octets.decode('utf_8', errors='replace') # \xe9를 ?(U+FFFD, 공식 유니코드 치환 문자)로 치환
```

- 파이썬 3는 UTF-8을 기본 인코딩으로 사용함. 굳이 주석을 달지 않아도 됨
- 인코딩 방식을 알아내는 방법
  - **할 수 없음**
  - 별도로 인코딩 정보를 가져와서 해야 함.
    - 헤더를 통해 알 수 있음
    - 127번째 바이트 이후에는 ASCII가 아닌 문자가 나오면 인코딩 방식을 알 수 있음
    - b'\x00'이 많이 나오면 UTF-16 LE, b'\x00\x00'이 나오면 UTF-32 LE 같이 경험적 추론 가능
  - `Chardet` 패키지를 사용하면 인코딩 방식을 알 수 있음. 30가지 인코딩 방식을 지원함
- BOM
  - 바이트 순서 표시(Byte Order Mark, BOM)는 유니코드 문자를 인코딩할 때 사용하는 바이트 순서를 나타내는 특수 문자
  - b'\xff\xfe'로 시작하는 파일은 UTF-16 LE로 인코딩된 파일임을 알려줌
  - b'\xef\xbb\xbf'로 시작하는 파일은 UTF-8로 인코딩된 파일임을 알려줌

```python
u16le = 'El Niño'.encode('utf_16le')
list(u16le)ㄴㅇㅁㄴ
# [69, 0, 108, 0, 32, 0, 78, 0, 105, 0, 241, 0, 111, 0]
u16be = 'El Niño'.encode('utf_16be')
list(u16be)
# [0, 69, 108, 0, 32, 0, 78, 0, 105, 0, 241, 0, 111]
```

## 4.5 텍스트 파일 다루기

---

- `locale.getpreferredencoding()`을 사용하면 기본 인코딩을 알 수 있음
- 대부분은 기본 인코딩으로 해결되지만, 잘 되지 않으면 encoding 옵션을 활용

## 4.6 제대로 비교하기 위해 유니코드 정규화하기

---

- é와 e\u0301은 같은 문자이지만 다른 코드 포인트를 가짐. `unicodedata.normalize()`를 사용하면 같은 코드 포인트로 만들 수 있음
- `NFC`는 코드 포인트를 조합해서 하나의 문자로 만듦. `NFD`는 코드 포인트를 분리해서 하나의 문자로 만듦
- `NFKC`와 `NFKD`는 호환성이 있는 문자를 사용함. 호환성이 있는 문자는 같은 의미를 가지지만 다른 코드 포인트를 가짐
- `case folding`은 대소문자를 구분하지 않는 방식으로 문자를 비교함. `lower` 메소드와 0.11% 정도 차이가 있음


## 4.7 유니코드 텍스트 정렬하기

---

- `locale.strxfrm()`을 정렬키로 사용하면 유니코드 문자열을 정렬할 수 있음. `setlocale()`을 사용해서 로케일을 설정해야 함
- 표준이 아닐경우 문제가 있어 `PyUCA` 패키지를 사용하면 됨. 지역 정보를 고려하지 않음

## 4.8 유니코드 데이터베이스

---

- `unicodedata` 모듈은 유니코드 데이터베이스를 사용함. `lookup()` 메소드는 문자명을, `name()` 메소드는 문자명을 반환함

## 4.9 이중 모드 str 및 bytes API

---

- bytes로 정규 표현식을 만들면 \d, \w 같은 패턴은 아스키 문자만 매칭함. str로 만들면 전부 매칭함


## 4.10 요약

---

- UTF-8은 가장 널리 사용되는 인코딩 방식임
- 문자열과 이진 시퀀스와 분리해야 함
- `Chardet` 패키지를 사용하면 인코딩 방식을 추론할 수 있음. UTF-16, UTF-32에서는 인코딩 방식을 추론하기 위해 BOM을 사용함
- 텍스트파일 읽을 때 encoding옵션 사용하는 것을 권고, 기본 인코딩 방식은 `locale.getpreferredencoding()`을 사용하면 알 수 있음
- 유니코드 문자열 비교를 할려면 정규화 해야함. 그 후에는 `locale.strxfrm()` 모듈을 활영해서 정렬하거나 `PyUCA` 패키지를 사용하면 됨
- 유니코드 DB는 모든 문자에 대한 메타데이터 제공하고, 이중 모드 API는 str, bytes를 모두 지원하나 처리방식이 다름