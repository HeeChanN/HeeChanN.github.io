---
title: "Spring Boot에서 외부 API 호출 (1) : RestTemplate"
date: 2025-04-12 3:00:00 +09:00
categories: [스프링부트,외부 API 호출]
tags:
  [
    스프링부트,
    RestTemplate,
    Apache HttpClient,
    외부 API 호출
  ]
---

> 스프링 부트를 이용하여 SMS API 호출 코드를 작성하며 학습하게된 RestTemplate에 관해 정리해보려고 합니다. RestTemplate의 기본 개념부터 RestTemplate + Apache HttpClient까지 WireShark를 통해 TCP 연결을 어떻게 재사용하는지 직접 확인해보려고합니다.

<br>

## 1. RestTemplate이란?

<br>

RestTemplate은 Spring Framework에서 제공하는 `동기식` HTTP 통신을 위한 클라이언트입니다. 서버 간 통신이나 외부 API 호출 시 쉽게 HTTP 요청을 보낼 수 있도록 도와주는 추상화된 API입니다.

기본적으로는 Java의 `HttpURLConnection`을 사용하여 동작하고 필요에 따라 `Apache HttpClient`, `OkHttp` 등을 내부에 주입해 사용할 수도 있습니다.

HTTP 메서드로는 GET, POST, PUT, DELETE, PATCH와 같은 주요 메서드를 지원해 주며 이런 메서드들은 내부적으로 HTTP 요청을 생성하고, 응답을 원하는 타입으로 자동 변환해 줍니다.

예시를 하나 들면 아래와 같이 작성하면 Http 응답을 User 객체로 변환까지 자동으로 처리됩니다.

```java
User user = restTemplate.getForObject("http://localhost:8081/user/1", User.class);
```

만약, 헤더를 커스터마이징 하고싶다면 HttpEntity<>을 이용하여 원하는 헤더를 넣어서 요청을 보낼 수 있습니다. 실제 HttpEntity의 생성자는 아래와 같이 구현되어있습니다.

```java
public HttpEntity(@Nullable T body, @Nullable MultiValueMap<String, String> headers) {
    this.body = body;
    this.headers = HttpHeaders.readOnlyHttpHeaders((MultiValueMap)(headers != null ? headers : new HttpHeaders()));
}
```

따라서, 실제 사용을 해본다면 아래와 같이 사용해볼 수 있습니다.

```java
HttpHeaders headers = new HttpHeaders();
headers.set("Authorization", "Bearer token");
HttpEntity<String> entity = new HttpEntity<>(headers);

ResponseEntity<String> response = restTemplate.exchange(
    "http://localhost:8081/api",
    HttpMethod.GET,
    entity,
    String.class
);
```

예시에서 사용한 exchange는 HttpMethod.GET처럼 파라미터 안에 메서드를 추가하여 GET뿐만아니라 POST, DELETE 등을 모두 사용할 수 있습니다. 또한 응답 헤더와 응답 상태코드까지 확인할 수 있어 요청에 대한 응답의 상세한 정보를 확인하고 싶다면 exchange()메서드를 활용하면 됩니다.

<br>

## 2. RestTemplate 기본설정 + WireShark

<br>

RestTemplate을 기본설정으로 사용할 때 어떻게 서버와 연결을 맺고 통신하는지 확인하기 위해 WireShark를 통해 모니터링 해보았습니다.

먼저 저는 아래와 같이 기본설정으로 사용해보았습니다. 

```java
@Bean(name = "simpleRestTemplate")
public RestTemplate smsRestTemplate() {
    return new RestTemplate();
}
```

위와 같이 설정하고 restTemplate으로 쓰레드 여러개로 동시에 10개의 요청을 보냈을 때 아래와 같이 10개의 TCP 커넥션이 맺어지는 것을 확인해볼 수 있었습니다.

![Image](https://github.com/user-attachments/assets/a480acbb-a1a6-4c85-bd3c-456512be9402)

이를 통해 톰캣의 max 쓰레드 풀 크기가 200일 때 커넥션이 200개까지도 생성될 수 있다고 예상해볼 수 있었습니다.

이후, 10개의 쓰레드를 동시에 모두 실행시키는 것이 아니라 Tread.sleep(1000)을 통해 1초 간격으로 실행했을 때는 TCP 커넥션을 재사용하는 것을 확인해 볼 수 있었습니다.

![Image](https://github.com/user-attachments/assets/5ba2af68-ed8a-4f6b-8e54-924408244bf0)

이는, Http 1.1의 Keep Alive로 인해 TCP 커넥션이 재사용되고 있었고 실제 sleep 간격을 늘려가며 확인해 보았을 때 총 5~6초정도 커넥션을 종료하지 않고 유지하는 것을 확인할 수 있었습니다.

<br>

기본 RestTemplate 설정으로만 사용할 경우 기본적으로 커넥션 유지시간을 5초에서 제어할 수 없으며 커넥션의 개수를 제어하려면 쓰레드 풀의 쓰레드 개수를 줄여야한다는 불편함이 존재했습니다.

이런 불편함을 제거하기 위해 사용하는 것이 바로 RestTemplate을 설정할 때 `Apache HttpClient의 커넥션 풀` 설정을 추가하는 것이라고 생각했습니다.

<br>

## 3. RestTemplate + Apache HttpClient

<br>

먼저 설정은 아래와 같이 진행했습니다.

```java
@Bean(name = "pooledRestTemplate")
public RestTemplate pooledRestTemplate() {

    PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(30);
    cm.setDefaultMaxPerRoute(30);

    HttpClient httpClient = HttpClients.custom()
            .setConnectionManager(cm)
            .evictIdleConnections(30, TimeUnit.SECONDS)
            .build();

    HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);

    factory.setConnectTimeout(5000);
    factory.setReadTimeout(5000);
    factory.setConnectionRequestTimeout(3000);

    return new RestTemplate(factory);
}
```

세부적인 설정에대한 설명보다 이전 RestTemplate 설정과 비교해서 설명하면, 가장 큰 차이는 커넥션 풀을 생성했다는 것입니다. 총 30개의 커넥션을 생성할 수 있고 각 도메인마다 총 30개의 연결을 사용할 수 있도록 설정하였습니다.`(cm.setMaxTotal(30), cm.setDefaultMaxPerRoute(30))`

또한, 이전에는 5초면 커넥션이 사라졌는데 위 설정을 사용하게 되면 커넥션 풀에서 유휴 커넥션이 총 30초까지 대기하게됩니다. `(evictIdleConnections(30, TimeUnit.SECONDS))`

결국 HttpClient 설정을 추가하면서 얻을 수 있는 장점은 연결에 대한 제어권이 개발자에게 있다는 점이라고 생각할 수 있었습니다.

<br>

## 4. 마무리하며

<br>

Spring에서 사용할 수 있는 RestTemplate은 학습곡선이 낮기 때문에 자주 사용했었는데 이번 학습을 통해 실제 사용할 때 어떤 부분을 주의해서 사용해야할지 더 자세하게 알 수 있었습니다. 최근에는 RestTemplate보다 WebClient를 더 많이 사용한다고 해서 이후에는 WebClient를 학습하고 RestTemplate으로 작성했던 코드를 WebClient로 변경하는 작업을 진행해보려고 합니다.
