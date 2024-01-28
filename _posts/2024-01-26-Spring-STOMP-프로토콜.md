---
title: "Spring STOMP 프로토콜"
date: 2024-01-26 00:00:02 +09:00
categories: [스프링 부트, STOMP]
tags:
  [
    프로토콜,
    네트워크,
    STOMP,
    스프링 부트,
    STOMP 테스트 코드,
    WebSocket
  ]
---


> Spring으로 WebSocket 위에서 동작하는 STOMP를 다루어보며 STOMP 프로토콜에 대해 자세하게 공부하게 되었는데, Spring docs를 따라가며 프론트 구현 없이 Spring으로 짠 STOMP 프로토콜 통신 코드를 테스트하는 것까지 너무 재밌게 공부해서 글로 남겨보려고 합니다. 제가 공부할 때 참고한 자료는 Spring docs로 아래 링크로 남겨놓았습니다. 또한 STOMP 프로토콜 공식문서도 존재하며 저는 Spring에서 다루는 STOMP만 이 글에서 다루었습니다.


[Spring Stomp docs](https://docs.spring.io/spring-framework/reference/web/websocket/stomp.html)

<br>

## 1. STOMP 란?
---

<br>

STOMP는 Simple Text Oriented Messaging Protocol의 약자로 Client와 Server사이의 비동기 메시지 처리와 관련된 프로토콜이다. 이 프로토콜은 작은 단위의 메시지 패턴을 다루기 위해 만들어졌습니다. 양방향 통신이 가능한 TCP 나 WebSocket 위에서 사용 가능한 프로토콜입니다. 

**HTTP**는 메시지 포맷이 어떻게 되어있나요? HTTP 요청 메시지 같은 경우 아래와 같이 Request, Header 그리고 Body 부분으로 나뉘어져 있습니다. 이것은 웹 통신 규약으로 다같이 이렇게 보내자고 한 일종의 약속입니다. 
![HTTP](https://github.com/HeeChanN/HeeChanN/assets/88177732/8d3bc7d2-ac86-4f9b-932c-21e6ea84a1f7)

STOMP 프로토콜도 지켜야 하는 메시지 형태가 존재합니다. 우리가 STOMP를 이용하여 통신을 하려면 아래와 같은 형식을 지켜야 합니다. 이런 메시지 형식을 Frame이라고 부릅니다.

```text
COMMAND
header1:value1
header2:value2

Body^@
```

1. 먼저 COMMAND 라인입니다. Client는 SEND와 SUBSCRIBE 명령을 메시지 전송에 사용할 수 있습니다. SEND 명령은 말 그대로 메시지를 서버로 보내는 명령이고 SUBSCRIBE는 특정 Topic에 Subscribe라는 행위를 하겠다는 뜻입니다. 

2. 헤더라인에는 목적지 uri와 세션id, 구독id와 등과 같은 이 메시지 frame에 대한 부가 정보들을 넣어줍니다. HTTP 헤더라인과 비슷하게 동작합니다.

3. Body라인 역시 HTTP의 Body라인과 마찬가지로 보내고 싶은 내용이 들어갑니다. Subscribe 명령 같은경우 보통 이 Body 라인은 비어있습니다.

따라서 이 정보를 바탕으로 예시 Frame을 하나 만들어보면 아래와 같은 SUBSCRIBE Command를 가진 frame을 생각해 볼 수 있습니다. 

```text
SUBSCRIBE
id:sub-1
destination:/topic/price.stock.*

^@ (내용이 없다는 표시)
```

이러한 구조로 STOMP는 간단한 Publish / Subscribe 메커니즘을 사용합니다. Server는 요청을 받아서 메시지 Brocker에게 메시지를 전달하고 Brocker는 Subscirbed한 Client에게 메시지를 보내는 방식으로 작동합니다.

<br>

그럼 이런 프로토콜이 Spring에서는 어떻게 작동할까요? Spring Websocket 위에서 STOMP 프로토콜은 WebSocekt 통신을 매우 편리하게 수행하도록 해줍니다. 원시 WebSocket 통신은 WebSocket API를 이용하여 Handler를 구현해야합니다. 양방향 연결을 위한 HandShake와 관련된 Handler를 구현하고, 메시지를 받는 Handler도 구현하는 등, 서비스보다도 통신을 위한 환경에 시간을 쏟아야 합니다. 이런 시간을 온전히 Service에 집중하도록 만들어 주는 것이 Spring의 STOMP입니다. 

그럼 Spring에서 Stomp 프로토콜의 동작과정을 자세하게 살펴보겠습니다.

<br>

## 2. STOMP 사용을 위한 설정
---

<br>

먼저 메시지 흐름을 보기 앞서 Spring에서 STOMP를 사용하기 위한 설정을 소개하고 시작하겠습니다. 첫 번째로 Gradle에 dependency를 추가합니다.
```gradle
implementation 'org.springframework.boot:spring-boot-starter-websocket'
```

아래 코드는 STOMP를 이용하기 위해 설정해야하는 Config 클래스입니다.


```java
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/portfolio"); 
	}

	@Override
	public void configureMessageBroker(MessageBrokerRegistry config) {
		config.setApplicationDestinationPrefixes("/app"); 
		config.enableSimpleBroker("/topic", "/queue"); 
	}
}
```

1. ``@EnableWebSocketMessageBrocker`` 어노테이션은 Spring 프레임 워크의 WebSocket 기능을 사용하여 메시지 프로커를 설정하고 관리하는데 사용됩니다. publish / subscribe를 위한 메시지 브로커는 등록하면 클라이언트와 서버사이의 메시지를 전달하는 역할을 수행합니다. 또한 웹 소켓 handler를 등록하여 클라이언트에서 수신한 메시지를 처리하고 해당 메시지를 브로커에게 전달하는 작업을 수행합니다. 이런 웹 소켓 통신에 필요한 기능들을 이 어노테이션을 사용하면 간편하게 설정할 수 있습니다. STOMP 프로토콜을 위한 웬만한 Bean들도 다 등록시켜주어서 나중에 필요하다면 어노테이션을 타고 들어가서 직접 여러가지 요소들을 보는 것도 추천드립니다.

2. ``registry.addEndpoint("/portfolio");`` 부분은 WebSocket Client가 WebSocket HandShake를 위해 연결해야하는 HTTP URL입니다. WebSocket은 양방향 통신 채널로 TCP와 비슷하게 HandShake가 이루어진뒤 데이터가 이동합니다. 따라서 Web Socket 통신에는 항상 제일먼저 WebSocket 서버와 WebSocket 클라이언트간의 HandShake가 필요합니다.


3. ``config.setApplicationDestinationPrefixes("/app");`` 이 부분은 STOMP 메시지에서 목적지 헤더에 들어간 url이 /app으로 시작하게 설정되어 있으면 이 url을 handling하는 @MessageMapping()로 보내집니다. 예를들어 아래와 같은 코드가 @Controller에 작성되어있다면 /app/message를 목적지로 하는 STOMP 메시지들은 @MessageMapping("/message")로 향하게 됩니다.

```java
@RestController
@RequiredArgsConstructor
public class MessageController {

    @MessageMapping("/message")
    public void greeting(MessageDto message) throws Exception {
        ...
    }
}
```


4. ``config.enableSimpleBroker("/topic", "/queue");`` 은 간단한 메시지 브로커를 활성화하는 설정입니다. 외부 브로커(RabbitMQ, ActiveMQ)로 설정을 할 수 있지만 저는 이번단계에서는 SimpleBrocker를 사용하였습니다. /topic 같은경우 일반적으로 특정 주제에 관련한 메시지를 다수에 BroadCasting하는데 사용하고 /queue는 특정 대상에게 메시지를 전달할 때 사용합니다. Client는 먼저 /topic/...로 Subscribe하고 이후에 /topic/...로 메시지를 보내면 메시지 브로커는 해당 토픽이나 큐를 SubScribed하고 있는 Client들에게 메시지를 전달합니다. 

이 설정파일을 바탕으로 예시와 함께 핵심적인 STOMP 메시지 전달 순서를 설명하려고 합니다. 앞으로 나오게 될 메시지 전달 순서에서 Channel에 집중하여 Message Flow를 따라가시는 걸 추천드립니다!


<br>

## 3. 메시지 전달 과정
---

<br>

![STOMP](https://github.com/HeeChanN/STOMP-test/assets/88177732/700de423-c5d5-449c-bfeb-017ad3219180)


위 그림은 공식 문서에서 소개하는 메시지 흐름을 나타내는 그림입니다. 

1. 먼저 SEND라는 command로 메시지가 request channel로 들어옵니다. request channel (  clientInBoundChannel )은 WebSocket 클라이언트로부터 수신한 메시지를 전달하는데 사용됩니다.
이 때, /app/... 을 목적지로하는 메시지는 등록된 MessageMapping으로 향하고 /topic/...을 목적지로 하는 메시지는 Simple Brocker에게 직접 전달 되어 brocker가 바로 BroadCasting 합니다. <br>
제가 생각했을 때 두 방식의 차이는 /app으로 보내게 되면 Controller의 @MessageMapping이 메시지를 받아서 
**"상대방 : 안녕하세요"** 와 같은 형식으로 메시지 내용을 만들어 brocker에게 전달하면 brocker가 그 메시지를 Subscribed한 Client들에게 BroadCasting하고 <br>
직접 /topic/...으로 전달하는 경우, 받은 메시지를 그대로 brocker에게 전달하고 brocker는 받은 메시지 그대로 client들에게 broadcasting 합니다. 따라서 둘다 공통적으로 brokcer에게 보내지만 메시지를 바로 사용할 수 있게 만들었느냐에 차이가 있다고 생각합니다.


2. /app/...으로 간 경우에 대해 자세하게 보면 먼저 메시지 destination에 /app/message로 설정되어있다면 아래와 같은 Controller의 @MessageMapping이 해당 메시지를 Handling 합니다.
```java
@RestController
@RequiredArgsConstructor
public class MessageController {

    private final SimpMessagingTemplate template;
    
    @MessageMapping("/message")
    public void greeting(MessageDto message) throws Exception {
        String text = message.nickname + " : " + message.content;
        template.convertAndSend("/topic/room/" +message.getRoomId(),text);
    }
}
```

3. /app/...으로 향한 메시지는 @MessageMapping을 거쳐서 Simple Brocker에게 가기 위한 통로인 brocker channel로 전달됩니다. 위 코드에서 SimpMessagingTemplate의 convertAndSend() 메서드가 이 역할을 합니다. convertAndSend()메서드는 메시지의 destination을 /app/message에서 /topic/room/{roomId}로 변경하고 payload부분도 바꾼 내용으로 변경해서 brocker channel로 보냅니다. brocker channel에서는 destination에 해당하는 topic으로 메시지를 전달하고 Brocker는 메시지를 구독한 client에게 BroadCasting 합니다.

4. Client에게 전달되는 메시지들은 모두 response channel(clientOutBoundChannel)을 거쳐서 전달됩니다.


> 외부 Message Brocker를 사용하는 경우도 비슷하게 동작하는데 Relay를 이용하여 외부 brocker와 통신하는 과정이 로직에 추가되는 것 말고 차이가 크게 없다고 생각합니다. 이 과정은 공식문서에 자세하게 나와있으므로 이후에 필요하다면 공식문서를 참고하시는걸 추천드립니다!

<br>

이 부분을 공부하고 Spring 공식 문서에서 소개한 STOMP 테스트 코드를 보며 STOMP의 동작 원리를 확인하려면 위 그림에서 소개한 3개의 채널을 확인하면 되겠다는 생각을 갖을 수 있었습니다. 제가 소개는 안했지만 공식 문서에 channel을 가로채는 방법도 나와있었는데 저는 이 channel을 가로채는게 테스트 코드에서 진가를 발휘한다고 생각합니다. 이제 위에서 설명한 메시지 흐름을 바탕으로 DB가 없는 간단한 채팅방을 구현해 보려고 합니다. 또한 오로지 Server Side로 우리가 만든 코드가 잘 작동하는지 test code를 통해 확인해 보겠습니다.


<br>

## 4. 채팅방 구독, 채팅메시지 전달 구현 및 테스트 코드 작성
---

<br>


저는 총 2가지 간단한 시나리오를 가지고 코드를 짜려고 합니다. 
1. 먼저 Client가 특정 채팅방을 구독하는 상황 
2. Client가 구독한 채팅방에 메시지를 보내는 상황을 구현해 보려고 합니다. 

프로젝트에 필요한 디펜던시는 아래와 같습니다.

```gradle
implementation 'org.springframework.boot:spring-boot-starter-websocket'
testImplementation 'junit:junit:4.12'
```

다음으로 위에서 보여드렸던 Stomp 프로토콜을 사용하기 위한 설정 파일입니다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfiguration implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableSimpleBroker("/topic"); 
        registry.setApplicationDestinationPrefixes("/app"); 
    }

    /** 웹소켓 HandShake를 위한 endpoint */
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-stomp");
    }
}
```
SimpleBrocker를 사용하고 메시지 Handling을 할 uri 시작주소는 /app Brocker가 관리한 uri는 /topic으로 설정하였습니다. 웹소켓 HandShake를 위한 url은 /ws-stomp로 설정하였습니다. 만약 WebSocket HandShake 까지 테스트고 싶다면 Stomp Client를 이용하여 연결을 테스트 해 볼수 있습니다. 저는 채팅방을 구독하고 메시지를 보내는 부분만 테스트 해 보겠습니다.

다음으로, clientInBoundChannel로 들어오는 메시지들중 /app을 목적지로 하느 메시지를 Handling하기 위한 Controller와 MessageDto입니다.

```java
@RestController
@RequiredArgsConstructor
public class MessageController {

    private final SimpMessagingTemplate template;
    private final MessageService messageService;

    @SubscribeMapping("/room/{roomId}") // --> return 값이 바로 clientOutBoundChannel로 감
    public String getRoomInfos(@DestinationVariable String roomId, Principal principal){
        return principal.getName() + "님 환영합니다.";
    }
    // 채팅방 구독 --> 환영 메시지 보내주기


    // 채팅방에 메시지 전달.
    @MessageMapping("/message")
    public void sendMessage(MessageDto message) throws Exception {
        String text = message.nickname + " : " + message.content;
        template.convertAndSend("/topic/room/" +message.getRoomId(),text);
    }
}
```

```java
@Getter
@Setter
@NoArgsConstructor
public class MessageDto {
    String nickname;
    String content;
    String roomId;

    public MessageDto(String nickname, String content, String roomId) {
        this.nickname = nickname;
        this.content = content;
        this.roomId = roomId;
    }
}
```


- SimpMessagingTemplate 같은 경우 설정 파일에서 @EnableWebSocketMessageBroker를 통해 빈으로 등록되어있어 사용 가능합니다. 이 메시지 Template 같은 경우 받아온 메시지를 변형시켜서 특정 destination으로 보낼 수 있습니다. 저는 /app/message로 들어온 메시지의 Body부분을 이용하여 /topic/room/1 로 메시지를 변경하여 보내는 로직에 사용하였습니다. 이렇게 보내게 되면 Brocker Channel로 메시지가 전달되고 메시지 브로커는 해당 메시지를 받아서 /topic/room/1 을 구독한 Client들에게 메시지를 BroadCasting 합니다.

<br>

- @SubscribeMapping은 오직 구독만을 위한 어노테이션으로 @MessageMapping과 똑같은 파라미터들을 지원하는데 Return value가 clientOutBoundChannel로 바로 보내져서 Brocker를 거치지 않습니다. @SendTo 나 @SendToUser 혹은 SimpMessageTemplate을 이용해서 brocker에게 보낼 수도 있습니다. 이때 파라미터로 사용된 @DestinationVariable은 @PathVariable과 똑같은 역할로 작동한다고 보시면 됩니다.

<br>

- @MessageMapping()같은 경우 Body로 메시지를 받아 정해진 형식으로 바꿔서 SimpMessageTemplate을 이용하여 /topic/room/{roomId}로 메시지를 보냅니다. 받아온 메시지를 메시지 브로커에게 전달하기 위해서는 @SendTo나 @SendToUser를 사용하거나 SimpMessagTemplate을 사용하여 brocker에게 메세지를 전달할 수 있습니다.

<br>

이제 작성한 코드를 테스트하기 위한 테스트 코드입니다. Server Side 테스트라 이 테스트 코드의 핵심은 STOMP 메시지 흐름에서 나온 3개의 채널에 중점을 두었습니다. (clientInBoundChannel, clientOutBoundChannel, brocker channl)

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = {
        StompTest.TestWebSocketConfig.class
})
public class StompTest {

    @Autowired
    private AbstractSubscribableChannel clientInboundChannel;

    @Autowired private AbstractSubscribableChannel clientOutboundChannel;

    @Autowired private AbstractSubscribableChannel brokerChannel;

    private TestChannelInterceptor clientOutboundChannelInterceptor;

    private TestChannelInterceptor brokerChannelInterceptor;


    @Before
    public void setUp() throws Exception {
        this.brokerChannelInterceptor = new TestChannelInterceptor();
        this.clientOutboundChannelInterceptor = new TestChannelInterceptor();
        this.brokerChannel.addInterceptor(this.brokerChannelInterceptor);
        this.clientOutboundChannel.addInterceptor(this.clientOutboundChannelInterceptor);
    }


    /** subscribe 테스트 */
    @Test
    public void enterChatRoom() throws Exception {

        StompHeaderAccessor headers = StompHeaderAccessor.create(StompCommand.SUBSCRIBE);
        headers.setSubscriptionId("0");
        headers.setDestination("/app/room/1");
        headers.setSessionId("0");
        headers.setUser(new TestPrincipal("Ahn"));
        headers.setSessionAttributes(new HashMap<>());
        Message<byte[]> message = MessageBuilder.createMessage(new byte[0], headers.getMessageHeaders());

        this.clientOutboundChannelInterceptor.setIncludedDestinations("/app/room/1");
        this.clientInboundChannel.send(message);

        Message<?> reply = this.clientOutboundChannelInterceptor.awaitMessage(5);
        assertNotNull(reply);

        StompHeaderAccessor replyHeaders = StompHeaderAccessor.wrap(reply);
        assertEquals("0", replyHeaders.getSessionId());
        assertEquals("0", replyHeaders.getSubscriptionId());
        assertEquals("/app/room/1", replyHeaders.getDestination());

        String json = new String((byte[]) reply.getPayload(), Charset.forName("UTF-8"));
        assertEquals(json, "Ahn님 환영합니다.");



    }

    @Test
    public void sendMessageTest()throws Exception{
        MessageDto messageDto = new MessageDto("male","안녕하세요","1");

        StompHeaderAccessor headers2 = StompHeaderAccessor.create(StompCommand.SEND);
        headers2.setSubscriptionId("0");
        headers2.setDestination("/app/message");
        headers2.setSessionId("1");
        headers2.setUser(new TestPrincipal("Ahn"));
        headers2.setSessionAttributes(new HashMap<>());
        byte[] payload = new ObjectMapper().writeValueAsBytes(messageDto);
        Message<byte[]> message2 = MessageBuilder.createMessage(payload, headers2.getMessageHeaders());


        this.brokerChannelInterceptor.setIncludedDestinations("/topic/room/1");
        this.clientInboundChannel.send(message2);
        Message<?> reply2 = this.brokerChannelInterceptor.awaitMessage(5);
        assertNotNull(reply2);

        StompHeaderAccessor replyHeaders2 = StompHeaderAccessor.wrap(reply2);
        assertEquals("/topic/room/1", replyHeaders2.getDestination());

        String json2 = new String((byte[]) reply2.getPayload(), Charset.forName("UTF-8"));
        assertEquals(json2,  "male : 안녕하세요");
    }

    @Configuration
    @EnableScheduling
    @ComponentScan(
            basePackages="com.study.testsocket",
            excludeFilters = @ComponentScan.Filter(type= FilterType.ANNOTATION, value = Configuration.class)
    )
    @EnableWebSocketMessageBroker
    static class TestWebSocketConfig implements WebSocketMessageBrokerConfigurer {

        @Autowired
        Environment env;

        @Override
        public void registerStompEndpoints(StompEndpointRegistry registry) {
            registry.addEndpoint("/ws-stomp").withSockJS();
        }

        @Override
        public void configureMessageBroker(MessageBrokerRegistry registry) {
            registry.enableSimpleBroker("/queue/", "/topic/");
            registry.setApplicationDestinationPrefixes("/app");
        }
    }

}
```
```java
public class TestChannelInterceptor implements ChannelInterceptor {

    private final BlockingQueue<Message<?>> messages = new ArrayBlockingQueue<>(100);

    private final List<String> destinationPatterns = new ArrayList<>();

    private final PathMatcher matcher = new AntPathMatcher();


    public void setIncludedDestinations(String... patterns) {
        this.destinationPatterns.addAll(Arrays.asList(patterns));
    }

    /**
     * @return the next received message or {@code null} if the specified time elapses
     */
    public Message<?> awaitMessage(long timeoutInSeconds) throws InterruptedException {
        return this.messages.poll(timeoutInSeconds, TimeUnit.SECONDS);
    }

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        if (this.destinationPatterns.isEmpty()) {
            this.messages.add(message);
        }
        else {
            StompHeaderAccessor headers = StompHeaderAccessor.wrap(message);
            if (headers.getDestination() != null) {
                for (String pattern : this.destinationPatterns) {
                    if (this.matcher.match(pattern, headers.getDestination())) {
                        this.messages.add(message);
                        break;
                    }
                }
            }
        }
        return message;
    }

}
public class TestPrincipal implements Principal {

    private final String name;

    public TestPrincipal(String name){
        this.name = name;
    }

    @Override
    public String getName() {
        return this.name;
    }
}
```
```java
public class TestPrincipal implements Principal {

    private final String name;

    public TestPrincipal(String name){
        this.name = name;
    }

    @Override
    public String getName() {
        return this.name;
    }
}
```

테스트 코드는 Spring Context를 가져와서 WebSocket 설정파일만 설정 파일로 등록한뒤 테스트 하였습니다. <br>

테스트 코드 첫번째 시나리오는 clientInBoundChannel로 Subscribe 메시지를 보내면 clientOutBoundChannel로 Controller에 설정한 메시지가 전달 되어야 하는 메시지 흐름을 이용하여 Subscribe 메시지가 잘 처리가 되는지 확인하였습니다. 따라서 우리에게 필요한 것은 channel을 intercepter해서 필요한 내용을 뽑아내는 것입니다. 그 부분이 ChannelInterceptor를 구현한 TestChannelInterceptor이고 이 TestChannelInterceptor를 이용하여 특정 uri로 향하는 메시지를 확인하고 서버 측의 메시지 전달 로직에 이상이 없음을 테스트하였습니다. <br>

두 번째 시나리오는 clientInBoundChannel로 Send 메시지를 보내면 Controller의 해당하는 MessageMapping()메서드가 메시지를 처리하고 brocker channel로 보내는 메시지 흐름에 기반하여 테스트 하였습니다. SimpleBrocker에게 메시지가 도착하고 나서부터는 따로 코드를 작성한 부분이 없어 brocker channel까지만 제대로 메시지가 도착한다면 메시지 전달에 이상이 없을 것이라고 가정하고 brocker channel을 interceptor하여 우리가 짠 메시지가 잘 담겨서 전달되고 있는지를 확인하였습니다.

제가 작성한 테스트코드는 Spring Stomp Testing에 올라와 있는 깃허브 코드를 많이 참고하여 작성하였습니다. [링크](https://docs.spring.io/spring-framework/reference/web/websocket/stomp/testing.html)


<br>

## 5. 마무리 하며...
---

<br>

 처음으로 아무것도 모르는 기술스택을 프론트단이 없이 서버단으로만 코드를 짜고 테스트코드를 돌려본 부분이 아직 너무 생소하지만 앞으로 계속 이런 방향으로 코딩 습관을 바꾸어 나가야겠다는 생각이 들었습니다. 직접 사용해보진 않았지만 테스트 코드를 통해 프론트엔드와의 협업에서 코드가 잘 작동하리라 확신을 가질 수 있었고 테스트 코드 작성을 어떻게 시작해야 할까를 조금이나마 생각해 볼 수 있는 시간이었습니다. 또한 프레임 워크의 테스트 코드를 작성하려면 해당 프레임 워크의 작동 원리와 api를 자세하게 알고 있어야 한다는 것을 경험해 볼 수 있었습니다. 
 
 이번에 작성한 테스트 코드는 공식 문서에 나와있는 코드에 많은 영향을 받은 코드입니다. 저는 이번 기회에 이렇게 테스트 코드도 다른 사람들은 어떻게 작성하는지 참고하여 난 이렇게 짜야겠다로 발전해 나가야 겠다는 생각을 갖게 되었습니다.
