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
