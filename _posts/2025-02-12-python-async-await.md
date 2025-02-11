---
title: "파이썬 GIL 비동기 (async/await, gevent)"
date: 2025-2-12 02:12:00 +09:00
categories: [비동기, python]
tags:
  [
    python,
    파이썬,
    비동기,
    async/await,
    greenlet,
    gevent
  ]
---


## 1. 파이썬의 GIL 환경에서 동작하는 비동기

<br>

Locust를 이용하여 부하테스트를 진행하다가가 문득 파이썬의 애플리케이션 네트워크 I/O가 어떻게 동작하는지 궁금하여 파이썬 환경에서 비동기가 어떻게 동작하는지 찾아보게 되었습니다.

먼저 공식 코루틴 방식인 async/await + asyncio를 살펴보고 Locust에서 사용하는 gevent에 대해서 살펴보려고합니다.

<br>

## 2. Async / Await + asyncio

<br>

먼저 동작과정을 살펴보기 전에 코루틴이라는 개념을 알아야합니다.

우리가 파이썬에서 함수를 선언하고 사용하면 보통 아래와 같이 작성하고 사용합니다.

```python
def myMethod():
```

이는 함수라고 부르기도하고 `서브루틴`이라고 부를 수도 있습니다. 이런 함수(서브루틴)들은 한번 호출되면 완전히 끝날 때까지 제어권을 돌려주지 않습니다.

하지만 함수 앞에 async를 붙인다면 이 함수는 서브루틴이 아닌 `코루틴`이 됩니다.

코루틴은 실행하다 중간에 멈추고 양보를 한 뒤 다시 실행할 수 있는 함수입니다.

여기서 핵심은 서브루틴과 다르게 코루틴은 제어권을 양보하여 동시에 여러 코루틴 작업을 처리할 수 있게 도와준다는 것입니다.

```python
import asyncio

async def my_coroutine():
    print("코루틴 시작")
    await asyncio.sleep(1)  # 1초 동안 대기 (양보)
    print("코루틴 재개")
    return "결과"

async def main():
   print("main 시작")
   coro_obj = my_coroutine()
	
   await coro_obj
   print("코루틴 종료")


asyncio.run(main())
```

위 코드는 코루틴을 사용한 코드의 예시입니다.

코루틴은 일반 함수와 다르게 호출하면 바로 실행되지 않고 코루틴 객체를 반환합니다.

예를 들어 main()메서드 내에서 my_coroutine()을 호출하면 해당 메서드가 바로 실행되는 것이 아니라 코루틴 객체가 생성되고
실행되지 않은 상태로 이벤트 루프에 Task로 등록됩니다. 등록된 Task들은 내부적으로 Ready Queue에 들어가고 현재 실행 중인 코루틴이 양보하거나 종료하게될 때 이벤트 루프의 스케줄링을 통해 실행될 수 있습니다.

이벤트 루프에 등록된다는 것은 asynico.run()메서드로 부터 시작됩니다.

asyncio.run(main())은 내부적으로 다음과 같이 동작합니다.

```markdown
1.  새로운 이벤트 루프를 생성
2.  main 코루틴을 스케줄링하여 실행
3.  main()이 종료되면 이벤트 루프 종료
```

main()메서드도 async 가 붙은 코루틴이며 main()에서 새로운 코루틴이 생성되면 이벤트 루프에 Task로 등록되어 main()과과 같이 이벤트 루프의 스케줄링 대상이 됩니다.

![coroutine1](https://github.com/user-attachments/assets/6854d60a-6751-49be-8db5-55e021e323fc)

위 코드를 실행시키면 다음과 같은 결과를 얻을 수 있는데 동작 과정을 살펴 보면 main()메서드가 시작되고 코루틴 객체를 생성한 후 await coro_obj 라인을 실행할 때 제어권을 양보하게 되고 이벤트 루프에 존재하는 코루틴을 실행하게 됩니다.
이후, 해당 코루틴이 종료되면 다시 main()메서드가 제어권을 받아와 출력을 하고 종료합니다.

my_coroutine()메서드 내부를 살펴보면 await asyncio.sleep(1)라는 코드가 존재하는데 이 부분이 바로 파이썬 비동기의 특징인 `협조적 Context Switching`를 보여줍니다.

지금상황에서는 코루틴이 main()과 my_coroutine()메서드 밖에 존재하지 않아 1초 대기를 하고 돌아와도 Ready queue에 task가 존재하지 않아 그 사이에 어떤 코루틴도 실행되지 않았지만, 만약 task에 코루틴이 대기하고 있었다면 양보와 동시에 대기하던 코루틴이 실행되게 됩니다.

아래 코드는 코루틴 10개를 동시에 생성하여 Task에 등록하고 이벤트 루프가 스케줄링으로 코루틴을 실행하는 예시입니다.

```python
import asyncio

async def coco(i):
    print(f"코루틴 {i} 대기")
    # 3초 대기: 이 동안 이벤트 루프가 다른 코루틴으로 전환
    await asyncio.sleep(3)
    print(f"코루틴 {i} 종료")
    return i

async def main():
    # 10개의 코루틴 객체 생성
    print("main() 코루틴 시작")
    coroutines = [coco(i) for i in range(10)]


    # asyncio.gather()를 통해 모든 코루틴을 동시에 실행
    task = asyncio.gather(*coroutines)

    print("main() 코루틴 양보")
    results = await task

    print("모든 코루틴 종료")


# 이벤트 루프 실행
asyncio.run(main())
```

![coroutine2](https://github.com/user-attachments/assets/2fd333c0-32c8-429d-96aa-e362dff24656)


코루틴 10개가 Task로 등록되었다가 main() 코루틴이 양보하는 순간간 10개의 코루틴이 시작하며 asyncio.sleep(3)으로 3초간 양보하게되고 내부적으로 가장 빨리 깨어난 코루틴부터 스케줄링으로 시작되어 종료되는 것을 볼 수 있습니다.

여기서 핵심은 await asyncio.sleep(3)에 있습니다.
만약 이 코드가 존재하지 않는다면 먼저 실행된 코루틴이 종료되기 전까지 다음 코루틴이 실행될 수 없어 아래와 같이 실행됩니다.

![coroutine3](https://github.com/user-attachments/assets/50bbc88b-4866-47f5-8aed-ac92dbb15d4a)

따라서, async / await의  핵심은 프로그램을 작성하는 사람이 직접 코루틴이 I/O 작업을 하게될 때 양보를 하도록 asyncio.sleep()을 적절하게 작성하는 것이라고 생각합니다.

-> 파이썬 3.13 버전부터는 No GLI가 나와 위 개념은 그 이전에 async / await에 대해서 다루고 있습니다.

<br>

## 3. greenlet 비동기

<br>

greenlet은 저수준 라이브러리로 협조적 Context Switching을 이용한다는 것은 같지만 그 내부 동작 과정이 async/await 과 다릅니다.

다음은 greenlet을 이용한 코드 예시입니다.



```python
from greenlet import greenlet

def task_a():
    print("task_a: 시작")
    # 여기서 task_b로 실행 제어를 넘김
    gr_b.switch()
    print("task_a: 다시 재개")
    gr_b.switch()
    print("task_a: 종료")

def task_b():
    print("task_b: 시작")
    # 여기서 task_a로 실행 제어를 넘김
    gr_a.switch()
    print("task_b: 다시 재개")
    gr_a.switch()
    print("task_b: 종료")

# greenlet 객체 생성
gr_a = greenlet(task_a)
gr_b = greenlet(task_b)

# 처음에 task_a를 실행
gr_a.switch()
```

먼저 이 코드에서는 gr_a메서드가 실행됩니다. gr_a() 메서드에서 gr_b.switch()에서 task_b()메서드로 제어가 넘어가고 gr_a.switch()에서는 다시 task_a() 메서드로 제어가 넘어갑니다.

주요 포인트는 개발자가 직접 명시적으로 특정 메서드로 실행 흐름을 바꾼다는 점이 async/await과 다른점이라고 볼 수 있습니다.

이벤트 루프의 스케줄링을 따르는 async/await과는 다르게 실행순서가 개발자가 정하는 전환 시점에 따라 결정됩니다.

또한, greenlet은 C 확장으로 구현되어 있어 메모리 사용량이 낮고 Context Switching 오버헤드가 극소화 되어있다는 장점이 있습니다.

그에 반해 async/await으로 동작하는 코루틴의 경우 Python 레벨의 객체로 관리되기 때문에 greenlet보다 약간의 오버헤드가 존재할 수 있지만, 고수준 문법으로 코드의 가독성과 유지 보수성을 높여주며 내부적으로 이벤트 루프가 동작하는 편리함이 존재합니다.

따라서, 두 가지 선택지는 어떤 목적이냐에 따라서 선택이 바뀔 것이라고 생각됩니다. Locust의 경우 부하테스트 툴로 주 목적이 많은 가상 유저를 만들어 서버에 요청을 보내는 것이기 때문에 Context Switching이 적고 경량 쓰레드인 greenlet을 선택했다고 생각합니다.

<br>

## 4. Locust의 greenlet 기반 gevent

<br>

gevent는 파이썬에서 비동기 I/O를 쉽게 구현하도록 도와주는 라이브러리입니다.

내부적으로 greenlet을 사용하며 Mokey Patching을 이용합니다.

Monkey Patching은 파이썬 표준 라이브러리들의 Blocking 함수(socket, time)들을 gevent 버전으로 대체합니다.

예를 들어 아래와 같이 호출하면

```python
from gevent import monkey
monkey.patch_all()
```
socket.socket이나 time.sleep 같은 I/O 함수들이 gevent의 비동기 함수로 덮어씁니다. 이를 통해 기존 동기 코드도 별도의 수정없이 비동기적으로 실행할 수 있게 됩니다.

또한 내부적으로 중앙 이벤트 루프를 두고 모든 비동기 이벤트를 모니터링하여 준비된 greenlet들을 재개하는 역할을 수행합니다.

`greenlet + monkey patch + 중앙 이벤트 루프`를 바탕으로 아래 코드처럼 동기로 짠 코드도 비동기로 동작하게 만들 수 있습니다.

```python
import gevent

def my_task(n):
    print(f"Task {n} 시작")
    gevent.sleep(1)  # 비동기적으로 1초 대기 (await와 유사한 역할)
    print(f"Task {n} 종료")

# 3개의 태스크를 동시에 실행
tasks = [gevent.spawn(my_task, i) for i in range(3)]
gevent.joinall(tasks)
```

이는 greenlet의 장점과 async/await의 장점을 합친 구조로 Locust에서는 이 방식을 사용하여 동기 메서드로 짜여진 시나리오를 monkey patching을 통해 blocking I/O 호출(socket, time.sleep 등)을 gevent가 제공하는 비동기버전으로 교체합니다.

<br>

## 5. 결론

<br>

Locust의 네트워크 I/O 비동기가 어떻게 동작하는지 궁금하여 시작된 학습으로 gevent를 이해하기 위해 기본적인 파이썬의 async/await에 대해서 자세하게 공부하게 되었습니다. 이를 바탕으로 greenlet을 학습하며 Locust가 왜 greenlet을 기반으로 만들었는지 이해하게 되었고 또, greenlet 자체를 사용하지 않고 왜 gevent를 사용하는지도 학습할 수 있었습니다. 

파이썬 3.13 부터는 no GIL이 등장했다고 하는데 기회가 된다면 이 no GIL이 등장한 뒤에 어떻게 비동기가 바뀌었는지를 학습해보며 블로그 글을 작성해 보겠습니다.
