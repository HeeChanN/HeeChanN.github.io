---
title: "Spring WebSocket-Jmeter-테스트(1)"
date: 2024-04-08 21:00:00 +09:00
categories: [부하 테스트, Jmeter]
tags:
  [
    WebSocket,
    웹소켓,
    Jmeter,
    부하 테스트,
    Stomp,
    Spring boot
  ]
---

<br>

> 부하테스트를 진행하기위해 Jmeter 샘플러를 사용하는 방법을 적용해보는 과정 중에 부족했던 Stomp 프로토콜 개념을 확인할 수 있었습니다. Jmeter로 Spring Stomp를 테스트할 때 고려해야 하는 한가지 요소와 Jmeter로 Spring Stomp로 만든 api를 테스트하는 방법을 정리한 내용입니다.

<br>

## 1. 테스트 환경 
---

<br>

가장 먼저 간단하게 Spring Stomp API를 테스트해 볼 수 있는 환경을 만들겠습니다. 첫 번째로 웹소켓연결과 Broker를 Config파일로 설정합니다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic"); // 구독과 BroadCasting 을 처리
        registry.setApplicationDestinationPrefixes("/app"); // 메시지 전달을 할때 사용할 uri
    }


    /** 웹소켓 HandShake를 위한 endpoint */
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-stomp");
    }
}
```

이후, 구독과 메시지를 처리할 Mapping들을 지닌 Controller를 만들어 준다.

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class MessageController {

    private final SimpMessagingTemplate template;

    //구독
    @SubscribeMapping("/room/chat")
    public ChatDto sub(){
        log.info("Sub : /room/chat");
        return new ChatDto("hello");
    }

    //메시지 처리
    @MessageMapping("/stomp-test")
    public void go(@RequestBody MessageDto message) throws Exception{

        String text = message.getNickname() + " : " + message.getContent();
        ChatDto chatDto = new ChatDto(text);
        log.info(text);
        template.convertAndSend("/topic/room/chat", chatDto);
    }

    
}
```
해당 Controller에서 사용하는 Dto들은 아래와 같다. MessageDto는 메시지를 받을 떄 사용하고 ChatDto는 서버에서 메시지를 구독한 Client에게 전달할 때 사용합니다.

```java
@Getter
@NoArgsConstructor
public class MessageDto {
    private String nickname;
    private String content;

    public MessageDto(String nickname, String content) {
        this.nickname = nickname;
        this.content = content;
    }
} 
```

```java
@Getter
@NoArgsConstructor
public class ChatDto {
    private String message;

    public ChatDto(String message) {
        this.message = message;
    }
}
```

위 코드들로 우리는 이제 Jmeter를 통해 연결이 잘 수행되는지 테스트를 수행해볼 준비가 되었습니다.

<br>

## 2. Jmeter 설치하기
---

<br>

먼저, https://jmeter.apache.org/ 해당 사이트에서 윈도우의 경우 Source에서 zip 파일을 다운 받고 압축을 해제합니다.

![Jmeter1](https://github.com/HeeChanN/HeeChanN/assets/88177732/e7c83cb7-c9db-4752-85e6-d5b9b233f45e)

이후, https://jmeter-plugins.org/install/Install/ 해당 사이트에서 플러그인을 다운 받아 Jmeter 디렉토리의 lib/ext 디렉토리에 넣습니다.

이후, Java 8버전 이상의 환경에서 Jmeter를 실행 후 옵션에서 PluginManger를 등록합니다. 아래 화면에서 Options를 눌러 PluginManager를 선택한 후 WebSocket Samplers by Peter Doornbosch 를 선택하여 플러그인을 등록해 줍니다. 
![Jmeter2](https://github.com/HeeChanN/HeeChanN/assets/88177732/3bff288d-837e-4ed5-b264-9774dad65e62)

여기까지 진행했다면 이제 WebSocket Stomp를 테스트할 준비가 완료되었습니다. 다음 단계에서는 Sampler를 생성하여 어떤 과정으로 Stomp 프로토콜을 테스트하는지 작성해 보겠습니다.

<br>

## 3. WebSocket Sampler 이용하기
---

<br>

테스트를 진행하기 앞서 웹소켓이 이루어지는 개념을 간단하게 정리하고 시작하겠습니다. 우리가 사용하는 Stomp 프로토콜은 웹소켓 연결에서 사용하는 프로토콜입니다. 그럼 웹소켓은 정확히 무엇일까요?

웹소켓은 HTTP 프로토콜을 이용한 통신과 다르게 한번 연결이 되면 지속적으로 연결되어있는 것으로 보통 실시간 메시지를 서로 주고받을 경우 사용합니다.

Spring Boot에서는 웹소켓 연결을 다음과 같은 과정으로 진행합니다.

![WebSocket](https://github.com/HeeChanN/HeeChanN/assets/88177732/f51ed64c-8da9-4fef-bd91-b1b4be4154e0)

1. 먼저 클라이언트가 웹소켓 연결 요청을 보냅니다. Server의 특정 URI에 웹소켓 연결 요청을 HTTP GET 메서드로 그 내용은 아래와 같습니다.
```Java
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
Origin: http://localhost:3000
``` 

2. 서버가 이 요청을 수락하면 HTTP 응답코드 101 Switching Protocols를 사용하여 응답하고 해당 연결을 웹소켓으로 Upgrade 합니다.

3. 이후 웹소켓으로 양방향 통신이 가능해집니다.

웹소켓의 개념을 간단하게 보았는데요, 그럼 Stomp 프로토콜은 무엇일까요? 

이 웹소켓 위에서 어떻게 메시지를 주고받을지에 대한 규약입니다. 이 Stomp 프르토콜을 바탕으로 SimpleBrocker는 구독을 관리하고 BroadCasting을 담당합니다. 이 개념들을 바탕으로 Jmeter의 Sampler를 만들어 보겠습니다. 

<br>

## 4. Jmeter Sampler 생성
---

<br>

먼저 TestPlan을 우클릭을 adds -> Threads(Users) --> Thread Group을 눌러 쓰레드 그룹을 생성해줍니다. 설정할 내용은 쓰레드 그룹 이름과 쓰레드 개수, Ramp-up, Loop Count를 설정해 줍니다. 각각의 속성에 대해 간단하게 설명하자면 
- 쓰레드 수는 말 그대로 생성할 쓰레드의 개수입니다.
- Ramp-up은 설정한 쓰레드의 개수를 몇초안에 만들어야하는지에 대한 설정입니다. Ramp-up 기간 동안 쓰레드들은 순차적으로 일정한 시간 간격으로 시작합니다.
- 각각의 쓰레드가 설정한 시나리오를 몇 번 반복할지에 대한 설정으로 만약 5라고 설정하였다면 설정한 시나리오를 5번 반복합니다.

![Jmeter3](https://github.com/HeeChanN/HeeChanN/assets/88177732/6c1ce840-8fbb-45c3-95b4-abeeeec8bdb2)

저희는 간단하게 연결이 잘 이루어지는지 테스트하기 위해 각 요소들을 1로 설정하고 테스트하겠습니다.

다음은 만들어야할 Sampler와 Listenr입니다. 총 5개의 Sampler와 1개의 Listenr를 만들 예정입니다. 
![Sampler](https://github.com/HeeChanN/HeeChanN/assets/88177732/8efcfc98-af3c-4acf-b2d9-2b97923704c0)

1. WebSocket Open Connection은 아래와 같이 쓰레드 그룹을 우클릭으로 누른 후 Sampler에서 WebSocket Open Connection을 누릅니다.  ![image](https://github.com/HeeChanN/HeeChanN/assets/88177732/fd4503c1-ec29-410a-af1b-98ffa247e823)
해당 과정은 Spring 서버에서 설정한 웹소켓 연결 URI에 웹소켓 연결 요청을 보내는 Sampler를 만드는 과정입니다. 아래는 Sampler 설정에 들어갈 내용입니다.![image](https://github.com/HeeChanN/HeeChanN/assets/88177732/d2661a99-c296-4dc0-83e5-564cc6073d5b)

<br>

2. 다음으로는 WebSocket Single Write Sampler를 만들어 Stomp Session을 생성하는 과정입니다. 1번 과정까지 진행하면 아직 웹소켓 연결만 진행된 상태이고 Stomp Session은 생성되지 않은 상태로 이 상태에서 Subscribe를 진행하게 된다면 StompSession이 Null값이 되어 아무리 메시지를 보내도 쓰레드로 메시지를 다시 전달받지 못하는 상태가 됩니다. 따라서 Subscribe를 진행하기 앞서 Connect 명령어를 통해 Stomp Session을 생성해야 합니다. 설정은 아래와 같습니다.![Sampler2](https://github.com/HeeChanN/HeeChanN/assets/88177732/884af972-1140-482a-b126-18a134fbafa1)

<br>

3. StompSession까지 생성했다면 구독과 메시지를 보내는 과정을 진행합니다. 설정은 아래와 같습니다.
![Sampler3](https://github.com/HeeChanN/HeeChanN/assets/88177732/836a9198-ea59-424a-adde-31445c0b1b01)![Sampler4](https://github.com/HeeChanN/HeeChanN/assets/88177732/536d40f2-b5cd-4d2b-a334-b3c6d55dcb5f)

<br>

4. 마지막으로 자신이 보낸 메시지도 자신이 받아야 하므로 WebSocket Single Read Sampler를 생성합니다. 해당 Sampler로 SimpleBrocker가 구독자들에게 BroadCasting할 때 메시지를 받아올 수 있습니다.![Sampler5](https://github.com/HeeChanN/HeeChanN/assets/88177732/69878feb-c73f-497e-83d3-598ae9592b73)

<br>

5. 위에서 만든 Sampler들은 하나의 시나리오로 쓰레드가 생성되고 수행할 일들입니다. 이 결과를 받기위해 쓰레드 그룹을 우클릭해 Listenr에서 View Result Trees를 생성합니다. 해당 화면에서 스레드를 만들어서 실행시켰을 때 결과를 받아볼 수 있습니다.

테스트에 필요한 모든 요소를 준비했고 다음으로는 쓰레드를 생성하여 결과를 보고 테스트가 잘 수행되는지 확인해 보겠습니다.

<br>

## 5. Jmeter 테스트 결과
---

<br>

작성한 Spring Boot를 로컬에서 실행시킵니다. 이후 Jmeter의 초록색 화살표 버튼을 눌러 쓰레드를 생성합니다. 그럼 다음과 같은 결과를 얻을 수 있습니다.
![result](https://github.com/HeeChanN/HeeChanN/assets/88177732/b46a2d64-bf00-40e5-94d3-781f51fb1d4d)

각, 샘플러 결과를 살펴보면 가장 먼저 웹소켓 연결이 잘 수행됬다는 사실을 파악할 수 있습니다. 연결 요청을 HTTP로 보낸 후 101 Status Code가 응답으로 온 것을 확인해 볼 수 있었습니다.
![WebSocket](https://github.com/HeeChanN/HeeChanN/assets/88177732/bb51e309-1821-45a6-9928-b6d28ec7aa8b)

다음으로 Stomp 프로토콜을 이용하여 Connect하고 Subscribe한 뒤 메시지를 Send하는 부분에서 모두 200 ok 상태 코드를 받은 것을 확인해 볼 수 있다.

최종적으로, 이 Stomp Session이 만들어졌다는 사실을 확인할 수 있는 곳은 Stomp-read Sampler다. 이 Sampler에서 보낸 메시지가 잘 받아진다면 Stomp Session이 생성된 것으로 이후에 부하테스트에서 메시지 전달이 잘 진행됨을 알 수 있습니다. ![Response](https://github.com/HeeChanN/HeeChanN/assets/88177732/c96d9fc6-4d3a-4c77-a57c-399b82042ff0)

<br>

## 6. 진행 중 발견했 던 문제점
---

<br>

웹소켓 테스트를 Jmeter로 어떻게 하는지 공부해 보았는데, 처음에는 Connect 커맨드를 사용하지 않아 BroadCasting되는 메시지를 제대로 전달받지 못하는 문제가 있었다. StompClient를 사용할 때는 웹소켓 연결과 동시에 StompSession이 생성되어 따로 Connect 명령을 사용할 일이 없어 학습을 하는 과정에서 빈틈이 존재하였다. 

Jmeter에서는 웹소켓 연결까지만 Sampler가 진행하고 Connect 프레임을 통해 StompSession을 생성해야 본격적으로 Stomp 프로토콜로 통신을 할 수 있었다. 처음 공부할 당시 Stomp 프로토콜에 대해서 빈틈이 존재해서 발생했던 트러블이었고 log를 debug모드로 설정한 후 발견할 수 있었다.

<br>

## 7. 마지막으로...
---

<br>

이번 기회에 Jmeter 툴을 자세하게 다룰 수 있었고 앞으로 채팅 서버 부하 테스트에 오늘 공부한 내용을 적용하여 필요한 자원에 대한 개념을 실습으로 학습할 예정입니다. 처음에는 단일 서버에 Spring 서버 + MongDB를 구축하여 어느정도 처리할 수 있는지 테스트해보며 서버의 자원을 스케일 업으로 올려가며 얼마만큼의 부하를 견딜 수 있는지 테스트해 볼 예정입니다.
이후에는, 스케일 아웃 방식과 각 서버를 분리하여 얼마만큼의 효율이 좋아지는지 확인해 볼려고 합니다.
