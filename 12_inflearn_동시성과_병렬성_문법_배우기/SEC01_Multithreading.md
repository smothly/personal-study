# SEC1 Multithreading

## 1-1 Process와 Thread의 차이점

---

![process vs thread](https://blog.kakaocdn.net/dn/cllUdv/btq0LPK8Czt/rx4QEXCBpUKhMz7qOcnpiK/img.jpg)
### Process
  - 운영체제에서 할당 받는 자원 단위 = **실행 중인 프로그램**
  - CPU 동작시간, 주소공간이 다 **독립적**
  - code, data, stack, heap이 전부 독립적
  - 최소 1개의 메인 스레드를 가지고 있음
  - 파이프, 파일, 소켓 등을 사용해서 프로세스간 통신 -> context switching 비용이 큼
### Thread
  - 프로세스 내에 실행 흐름 단위
  - 프로세스 자원 사용
  - **Stack만 별도 할당**하고 나머지는 공유(code, data, heap)
  - 한 스레드의 결과가 다른 스레드의 영향 끼침 => 동기화 문제를 주의해야함
### Multi Thread
  - 단일 어플리케이션에서 여러 스레드로 구성 후 작업 처리
  - 시스템 자원 소모가 감소됨
  - 통신 부담은 감소, 디버깅 어려움, 단일 프로세스에는 효과 미약, 자원 공유 문제(데드락), 프로세스 영향을 줌
### Multi Process
  - 단일 어플리케이션에서 여러 프로세스로 구성 후 작업 처리
  - 프로세스 문제 발생은 확산이 없음. 프로세스를 kill 하면 됨
  - cache change, 복잡한 통신 방식으로 인해 cost가 높음(context switching으로 인한 오버헤드)

## 1-2 GIL(Global Interpreter Lock)

---

### Python GIL
- ![GIL 사진](https://velog.velcdn.com/images/chaduri7913/post/a50fddb4-2c86-4013-ac39-808f66c65fcd/image.png)
- 단일 스레드만이 python object에 접근 하게 제한하는 **mutex**
- cpython이 메모리 관리가 취약함. **thread-safe하게 하기 위해 GIL 사용**
- 단일 스레드로 충분히 빠름
- 프로세스 사용이 가능하기 때문에 GIL이 큰 제약이 되지 않음
- 병렬 처리는 multi processing, asyncio가 있음
- thread 동시성 완벽 처리를 위해 jython, ironpython, stackless python 등이 존재하긴 함

## 1-3 Thread - baic

---

- 로깅 함수 사용
- 자식스레드여도 자신이 맡은 일을 끝까지 하고 종료함
- join을 추가하면 자식스레드 일을 끝날때 까지 기다리기
```python
import logging
import threading
import time

# 스레드 실행 함수
def thread_func(name):
    logging.info("Sub-Thread: %s: starting", name)
    time.sleep(3)
    logging.info("Sub-Thread: %s: finishing", name)

# 메인 영역
if __name__ == "__main__":
    # Logging format 설정
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO, datefmt="%H:%M:%S")
    logging.info("Main-Thread: before creating thread")

    # 함수 인자 확인
    x = threading.Thread(target=thread_func, args=('First',))
    logging.info("Main-Thread: before running thread")

    # 서브 스레드 시작
    x.start()

    # 주석 전후 결과 확인
    x.join()

    logging.info("Main-Thread: wait for the thread to finish")

    logging.info("Main-Thread: all done")

```