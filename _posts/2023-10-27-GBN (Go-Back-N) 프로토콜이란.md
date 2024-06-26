---
title: "GBN (Go-Back-N) 프로토콜이란"
date: 2023-10-27 23:10:00 +09:00
categories: [CS, 네트워크]
tags:
  [
    프로토콜,
    네트워크,
    트랜스포트 계층,
    GBN
  ]
---

> [컴퓨터 네트워킹 하향식 접근 8판]을 이용하여 네트워크를 공부하며 책에 있는 예시가 부족하여서 조금 시간을 들여 생각했던 부분을 정리해 놓고 싶어서 쓰게되었습니다. Go-Back-N 프로토콜의 FSM(유한 상태 머신)을 이용하여 공부하였고 책의 내용과 제가 결론 지은 내용을 합쳐서 정리한 내용입니다.

<br>

## 0. GBN 이란?
---

<br>

먼저 Go-Back-N 프로토콜을 공부하기 앞서 신뢰적 데이터 전송의 원리에 대해 짚고 넘어가겠습니다. 트랜스포트 계층에서 말하는 신뢰적인 데이터 전송의 기준은 데이터가 손실되지 않고 순서대로 송신자에서 수신자로 데이터가 이동해야합니다.<br>
 이런 신뢰적 데이터 전송을 위해 stop-and-wait방식을 이용하는 것이 rdt 방식이고 여기에 성능을 위해 파이프라이닝을 적용한 방식이 GBN 프로토콜입니다. 따라서 rdt까지 사용되던 방법들이 대부분 그대로 GBN에 적용이 되고 파이프 라이닝을 위한 추가적인 Sequence Number와 버퍼링을 이용하는것이 GBN 프로토콜입니다. **GBN 프로토콜은 파이프라이닝을 통하여 송신자의 확인응답을 기다리지 않고 여러 패킷을 전송할 수 있습니다.**

 ✏️저는 GBN을 공부하며 누적 확인 응답이라는 개념과 Sequence Number를 어떻게 이용하는지 자세히 알 수 있었고 해당 내용을 정리해 보고자 합니다.


<br>


## 1. GBN 프로토콜의 구성
---
<br>

GBN에서 확인 응답 (ACK)이 안오고도 계속 보내기 위해서 해당 정보를 버퍼링해야합니다. 따라서 GBN에서 사용하는 방법은 바로 `**윈도우 프로토콜**`입니다. 확인응답이 되지 않은 패킷의 최대 허용 수를 Widnow size 보다 작거나 같게하여 파이프라인으로 한번에 보낼 수 있는 패킷의 양을 조절합니다. 아래 그림으로 추가 설명을 해드리겠습니다.

![GBN](https://github.com/HeeChanN/HeeChanN/assets/88177732/39bb82e9-75f4-4faf-8c2f-e6a5993d795a)

이 그림은 송신자의 Sequence Number 범위를 보여주는 그림입니다. send_base이전의 패킷은 이미 전부 ACK를 받은 패킷들이고 send_base부터 nextseqnum이 현재 ACK를 받아야 하는 패킷들이며 nextseqnum부터 send_base + N에 해당하는 부분은 지금 현재 송신자가 보낼 수 있는 패킷입니다. 
<br>이런 요소들을 이용하여 송신자는 ACK가 도착하지 않은 패킷들을 체크하며 앞으로 추가적으로 몇개의 패킷을 수신자에게 보낼 수 있는지 판단합니다. 따라서 핵심은 

> 전송되었지만 아직 ACK가 도착하지 않은 패킷과 앞으로 보낼 수 있는 패킷을 한번에 묶는 크기 N인 윈도우가 중요합니다.
{: .prompt-info }

<br>

## 2. GBN의 동작 과정
---
<br>

그럼 윈도우를 GBN동작과정과 함께 살펴볼까요?

먼저 `**송신자의 입장**`입니다. 

1. 송신자는 상위로부터 호출이 되면 해당 데이터를 받아와서 수신자에게 전송해야합니다. 이 때 송신자의 윈도우 사이즈 N 보다 확인 응답을 받지 못한 패킷의 개수가 적다면 송신자는 데이터를 받아올 수 있고 해당 데이터를 패킷으로 전송할 것입니다. 하지만 N과 확인 응답이 되지 않은 패킷의 수가 같다면 상위 계층에서 온 데이터를 거부합니다.

<br>

2. 수신자가 보낸 ACK를 받게되면 해당하는 ACK의 이전의(윈도우 범위 내의) 패킷들을 모두 확인 응답 받은 것으로 처리하고 send_base의 값을 받은 ACK의 Sequence Number + 1로 설정합니다. 핵심은 바로 수신 측에서 관리하는 올바르게 수신된 누적확인 응답 번호입니다.

> 저는 이 부분에서 누적 확인 응답의 개념이 정확하게 잡혀 있지 않아 처음에 잘못 이해하였고 나중에 ACK 손실에 대한 경우를 생각해 보다가 누적 확인 응답에 대한 개념이 정확하게 잡혔습니다.
{: .prompt-warning }

<br>

3. 타이머는 1개를 이용하며 처음 패킷을 보내기 시작할 때send_base값과 next_seqnum값이 같을경우 timer를 작동시킵니다. 패킷에 대한 ACK를 받은 상태라면 그 시점에서 다음 확인 응답을 받지 않은 패킷에 대한 timer를 작동시킵니다. 타임아웃이 나면 send_base부터 next_seqnum까지 모든 패킷을 재전송합니다. 


> 그럼 수신자는 어떻게 작용할까요? 아직까지 수신자의 상황에 대한 설명은 제가 한 것이없는데요. 수신자 같은 경우 GBN에서는 매우 단순하기 때문입니다. 
{: .prompt-info }

<br>

수신자는 Sequence Number n을 가진 패킷이 오류없이 그리고 순서대로 수신된다면 수신자는 패킷 n에 대한 ACK를 송신하고 상위 계층(Application Layer)으로 데이터를 전달합니다. 그 외에 경우 (순서대로 패킷이 오지 않는 경우) 해당 패킷을 버리고 가장 최근에 보낸 Sequence Number의 ACK를 보냅니다. 
<br>예를 들면, 가장 최근에 4번 패킷에 대한 ACK를 보냈고 현재 5번 패킷이 와야되는데 6번 패킷이 먼저온다면 4번 패킷에 대한 ACK를 보냅니다. 

```text
GBN에서 누적 확인 응답이란?
패킷이 상위 계층에 한번에 하나씩 전송되므로 만일 Sequnece Number가 k인 패킷이 수신되고 상위 계층에 전달 되었다면 k-1까지의 패킷은 모두 수신자에게 전달 되었다는 점을 의미합니다.
``` 

<br>

## 3. GBN 프로토콜 예시
---

<br>

저는 예시를 보면서 왜 GBN에서 윈도우 프로토콜의 size를 Sequence Number보다 한 사이즈 더 작아야 되는 줄 알았습니다.


정상적인 수행 예시와 비정상적일 경우의 예시를 보며 어떻게 하면 이런 기준이 나오는지 살펴보겠습니다.

![GBN2](https://github.com/HeeChanN/HeeChanN/assets/88177732/fa3dc2ac-4c0e-4f11-badf-ce60da90c551)

먼저 정상적인 수행 과정입니다. 
1. 2번 패킷이 손실되고 이후 0번과 1번 ACK가 도착한 후에 윈도우가 이동합니다. 이후 3번 패킷의 ACK에는 가장 최근에 받은 패킷인 1번 ACK가 보내집니다.

2. 이후 4번, 5번 패킷이 보내지고 3번 패킷에 대한 NAK(1번 ACK)가 보내지면 송신자는 아무 행위도 하지 않습니다.

3. 이후 2번 패킷에 대한 타이머가 끝나며 2번패킷부터 5번 패킷까지 모두 재전송을 하는 것을 볼 수 있습니다.


<br>

다음은 제가 교안의 ppt를 수정하여서 만든 예시입니다.

![GBN4](https://github.com/HeeChanN/HeeChanN/assets/88177732/6d699517-e754-4c06-bc0a-8960e913dc93)


윈도우 사이즈와 Sequence Number가 같을 경우 발생할 수 있는 문제점입니다.

1. 정상적인 예시와 똑같이 3번 패킷 까지 보내지만 1번과 2번 패킷이 정상적으로 수신자에게 도착하지 않습니다.

2. 이 상황에서 0번 패킷에대한 ACK는 정상적으로 도착하고 윈도우가 오른쪽으로 한칸 이동합니다.

3. 3번 패킷에 대한 ACK로 아직 누적확인 응답이 0번까지밖에 안쌓여서 0번에 대한 ACK를 보냅니다.

4. 송신자는 0번 ACK를 현재 윈도우에서 제일 마지막 0번으로 확인하고 send_base를 1에서 다음 1까지 이동합니다. 이때, 아직 1번 2번 3번 패킷을 수신자가 받지 않았지만 송신자에서는 받았다고 처리하게 되며 문제가 발생하게 됩니다. 

만약 Sequence Number가 4까지였다면 어떤 결과가 나왔을까요?

아마 Time out 전까지 0번 ACK가 도착하면 위의 경우처럼 윈도우를 이동시키지 않았을것입니다. 


따라서 저희는 아래와 같은 결론을 내릴 수 있습니다.

> (window size) N < Sequence Number
{: .prompt-info }


<br>


## 4. GBN 정리
---

<br>

먼저, GBN의 장점은 파이프라이닝으로 이전의 rdt의 stop-and-wait을 해결했다는 점입니다. 또한 timer를 하나만 써서 해결하여 타이머에 대한 오버헤드가 적다는 점을 들 수 있습니다. 하지만, 매 타임아웃마다 모든 확인 응답을 받지 못한 패킷을 재전송 한다는 점에서 불필요한 데이터로 파이프라인이 채워지게 되므로 성능적인 면에서 좋지 못하다는 점을 들 수 있습니다.<br>
따라서, Selective Repeat 이라는 Timer를 여러개 두는 방법과 TCP상에서 누적 확인 응답을 적절하게 사용하는 방법으로 이런 여러번 재전송 하는 문제점을 해소한 방법들이 존재합니다. 

<br>


## 5. 마무리 하며...
---

<br>

저는 GBN을 처음 공부할 때 책에 예시로 안나와 있어서 고려하지 못했습니다. 하지만 이후에 Selective Reapet을 공부하며 window size의 값을 Sequence Number / 2 보다 작거나 같은 범위만 가능하다는 조건을 보며 GBN에도 적용하다가 알게 되었습니다.<br> 
 최근에 네트워크를 공부하며 Spring boot에서 어떤 방식으로 네트워크가 연결 되는지 모르며 사용했던 부분들에 대해 다시 한번 공부를 해야 겠다는 결심을 하게 되었습니다. 또한 자바 언어로 한번도 소켓을 열어 통신을 시도해보지 않았는데 이런 부분도 직접 실습을 해보며 네트워크에 대한 이해도를 높여야 겠다는 생각이 들었습니다.

