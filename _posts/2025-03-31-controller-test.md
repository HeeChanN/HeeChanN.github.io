---
title: "JUnit Controller + Security 테스트"
date: 2025-03-31 3:00:00 +09:00
categories: [스프링부트,JUnit]
tags:
  [
    스프링부트,
    JUnit,
    AssertJ,
    Controller 테스트,
    Security
  ]
---

> Spring Security를 사용하게 되면, Controller는 인증/인가와 밀접한 관계를 가지게 됩니다. 따라서 각 Controller마다 인증/인가 관련 테스트 코드를 통합해서 작성하기보다는, Security 설정을 불러와 Controller와 함께 테스트하는 방식이 테스트 관리면에서 효율적이라고 판단했습니다. 이 글에서는 Controller와 Security 설정을 함께 테스트하는 코드를 작성하기 위해 학습한 내용을 정리해보려 합니다.

<br>

## 1. Controller 테스트 단계적 설정

<br>

큰 구조는 다음과 같이 구성하였고 아래 코드에서 Controller마다 필요한 service 빈에 대해서만 수정하여 적용하였습니다.

```java
@WebMvcTest(EntityController.class)
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Import({ControllerTestSecurityBeans.class, SecurityConfig.class})
public class EntityControllerTest {
    @Autowired
    @Autowired
    private MockMvc mockMvc;

    @Autowired private ObjectMapper objectMapper;

    @MockBean
    private EntityService entityService;

    @MockBean 
    private JwtProvider jwtProvider;
}
```

<br>

### `@WebMvcTest(EntityController.class)`
이 어노테이션은 특정 Controller에 대한 단위 테스트 환경을 구축할 수 있도록 도와줍니다. 테스트 시, 전체 애플리케이션 컨텍스트를 로드하는 것이 아니라 Web Layer만 최소한으로 구성하여 빠르게 테스트할 수 있습니다.

@WebMvcTest를 사용하면 다음과 같은 컴포넌트들이 스프링 컨텍스트에 포함됩니다:
 
 `@Controller`, `@ControllerAdvice`, `@JsonComponent`, `Converter`, `Filter`, `HandlerInterceptor`

<br>

### `@AutoConfigureMockMvc 와 mockMvc`
Spring Boot는 기본적으로 내장 Tomcat과 같은 서블릿 기반의 요청 처리 방식을 따릅니다. Controller 테스트를 위해 실제 네트워크 요청을 발생시키는 대신, MockMvc를 사용하여 HTTP 요청과 응답을 모의(Mock)할 수 있습니다.

MockMvc는 실제 서블릿 컨테이너 없이도 필터 체인, DispatcherServlet, 그리고 Controller를 호출할 수 있게 해주며, 테스트 환경에서 가상의 MockHttpServletRequest와 MockHttpServletResponse를 생성하여 전체 요청 흐름을 검증할 수 있습니다.

@AutoConfigureMockMvc는 MockMvc 객체를 자동으로 생성하고 주입해주는 어노테이션입니다. 만약 수동으로 구성하고 싶다면 아래처럼 설정할 수 있습니다:
```java
mockMvc = MockMvcBuilders.standaloneSetup(myController)
                .apply(springSecurity()) // Security 필터 적용 (원한다면)
                .setControllerAdvice(new GlobalExceptionHandler()) // 전역 예외 처리
                .setMessageConverters(new MappingJackson2HttpMessageConverter()) // 메시지 컨버터
                .build();

```

자동 설정 시, 아래와 같은 옵션을 사용할 수 있습니다:
```java
@AutoConfigureMockMvc(
    addFilters = false, //기본값 true
    secure = false, //기본값 true
    printOnlyOnFailure = false //기본값 true
)
```

<br>

### @Import({ControllerTestSecurityBeans.class, SecurityConfig.class})

<br>

@WebMvcTest는 경량화된 Web Layer만 로딩하기 때문에, 인증/인가 테스트를 위해 Security 설정이 별도로 스프링 컨텍스트에 등록되어야 합니다. 이를 위해 @Import를 통해 SecurityConfig와 테스트용 Security 관련 Bean들을 포함하는 ControllerTestSecurityBeans 클래스를 명시적으로 등록해주었습니다.

Security 설정에서 필요한 주요 Bean은 다음과 같습니다:

```java
SecurityConfig.class

private final CorsConfigurationSource corsConfig;
private final AuthorizationConfig authorizationConfig;
private final JwtAuthenticationFilterConfig jwtAuthenticationFilterConfig;
private final SecurityExceptionFilterConfig securityExceptionFilterConfig;
```

이 중, 실제 인증/인가 동작에 필요한 Bean들은 ControllerTestSecurityBeans 클래스에서 직접 @Bean으로 등록하였고, CorsConfigurationSource는 테스트에 영향을 주지 않도록 @MockBean으로 대체하였습니다.

또한 JwtAuthenticationFilter의 동작 자체는 테스트 대상이 아니므로, 해당 필터에서 사용하는 JwtProvider 역시 @MockBean으로 등록하여 단위 테스트의 범위를 명확히 구분하였습니다.

```java
@TestConfiguration
public class ControllerTestSecurityBeans {

    @MockBean
    @Qualifier("customCorsConfigurationSource")
    private CorsConfigurationSource corsConfigurationSource;

    @Bean
    public JwtAuthenticationFilterConfig jwtAuthenticationFilterConfig(JwtProvider jwtProvider) {
        return new JwtAuthenticationFilterConfig(jwtProvider);
    }
    @Bean
    public AuthorizationConfig authorizationConfig() {
        return new AuthorizationConfig();
    }

    @Bean
    public SecurityExceptionFilterConfig securityExceptionFilterConfig(HandlerExceptionResolver handlerExceptionResolver) {
        return new SecurityExceptionFilterConfig(handlerExceptionResolver);
    }
}
```


<br>

## 2. Client의 요청이 처리되는 구조 파악 (with Security)

<br>


![image](/assets/img/post/controller_test/1.png)

위 그림은 클라이언트 요청이 처리되는 과정을 간단히 나타낸 흐름도입니다. 사용자의 요청이 들어오면 Tomcat은 HttpServletRequest와 HttpServletResponse 객체를 생성하고, 이를 서블릿 필터 체인에 전달합니다. 각 필터를 거친 후 DispatcherServlet에 도달하게 되고, 최종적으로 요청에 해당하는 Controller의 URI로 매핑됩니다.

이 과정에서 인증과 인가를 담당하는 Spring Security Filter Chain은 서블릿 필터 체인 내 하나의 필터로 동작합니다. Controller 테스트에서도 이 인증/인가 과정을 포함해 테스트하고자 로그를 활용해 동작 순서를 확인해본 결과, 아래와 같은 흐름으로 동작함을 확인했습니다. (테스트 환경: Spring Boot 2.7)

![image](/assets/img/post/controller_test/2.png)

Spring Security에서 인증/인가의 핵심은 SecurityContext에 저장된 사용자 정보 (Authentication)입니다.
요청이 들어오면 다음과 같은 순서로 인증 및 인가 처리가 진행됩니다:

1. `SecurityContextPersistenceFilter`<br>
요청마다 새로운 SecurityContext를 생성합니다.

<br>

2. `JwtAuthenticationFilter`<br>
요청에 토큰이 포함되어 있다면, 해당 토큰을 검증하여 사용자 정보를 추출하고 Authentication 객체로 만들어 SecurityContext에 저장합니다.<br>
토큰이 없다면 SecurityContext는 비어 있게 됩니다.

3. `AnonymousAuthenticationFilter`<br>
SecurityContext에 인증 정보가 없는 경우, 익명 사용자(AnonymousAuthentication)를 생성하여 SecurityContext에 저장합니다.

4. `FilterSecurityInterceptor`<br>
현재 요청의 URI에 대해 권한이 필요한지 검사합니다. 
인증되지 않은 사용자의 경우 `인증 예외`가 발생하고, 권한이 부족한 사용자의 경우 `인가 예외`가 발생합니다.
 
5. `ExceptionTranslationFilter`<br>
발생한 예외를 아래처럼 등록한 핸들러에서 처리합니다:
```java
http.exceptionHandling()
        .accessDeniedHandler(new CustomAccessDeniedHandler())
        .authenticationEntryPoint(new CustomAuthenticationEntryPoint());
```

6. `SecurityExceptionFilter`<br>
커스텀 예외 필터에서 예외를 한 번 더 감싸서 GlobalExceptionHandler로 전달하고, 공통 예외 응답으로 변환합니다.

<br>

따라서 인증/인가 예외를 테스트하기 위해 필요한 것은 결국 `Authentication` 객체의 존재 여부이며, 저는 이를 기반으로 `@WithMockUser(username = "admin", roles = "ADMIN")` 어노테이션을 활용해 SecurityContext에 인증 정보를 강제로 삽입하여 테스트를 작성했습니다.

직접 Authentication 객체를 생성해서 주입할 수도 있습니다:

```java
CustomUserDetails customUserDetails = new CustomUserDetails(new Member());
UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
    customUserDetails, null,
    Collections.singleton(new SimpleGrantedAuthority("ROLE_ADMIN"))
);
SecurityContextHolder.getContext().setAuthentication(auth);
```

이와 같은 방식으로 AuthorizationConfig에 설정된 URI별 권한 검증, 인증 객체 등록, MockMvc를 활용한 요청 테스트까지 한 번에 구성할 수 있었습니다.

예시 코드는 아래와 같습니다.

```java
@Test
@WithMockUser(username = "admin", roles = "ADMIN")
void 어드민_축제기간_생성() throws Exception {
// given
doNothing().when(durationService).createDurations(any(), any());

// when & then
mockMvc.perform(post("/duration")
            .param("festivalId", "1")
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(durationCreateDtos)))
        .andExpect(status().isOk());
verify(durationService, times(1)).createDurations(any(), any());
}

@Test
void 일반유저_축제기간_생성_인증거부() throws Exception {
    // given
    doNothing().when(durationService).createDurations(any(), any());

    // when & then
    mockMvc.perform(post("/duration")
                    .param("festivalId", "1")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(durationCreateDtos)))
            .andExpect(status().isUnauthorized());
}

```

<br>

## 3. 마무리 하며

<br>

SecurityFilter의 동작 과정에에 추가적인 설명을 하자면, SecurityExceptionFilter는 직접 커스텀하여 추가한 필터로, 인증/인가 예외를 최초로 처리하는 역할을 합니다.
accessDeniedHandler와 authenticationEntryPoint에서 발생하는 예외를 커스텀 예외로 변환해 던지면, 이 예외를 SecurityExceptionFilter에서 받아 처리하고, 최종적으로 GlobalExceptionHandler를 통해 일관된 응답으로 매핑되도록 구성했습니다.

이와 같이 구성한 이유는, JwtAuthenticationFilter에서 토큰 검증 중 발생하는 예외를 세부적으로 커스터마이징하고자 했기 때문입니다. 기존 구조에서는 필터 단계에서 발생한 예외를 원하는 방식으로 핸들링하기 어려웠기에, 예외 처리 전용 필터를 추가하게 되었습니다. 결과적으로 예외를 통합적으로 관리할 수 있는 구조를 갖추게 되었고, 이는 유지보수 측면에서도 효과적인 접근이라고 판단했습니다.

이번 테스트 환경을 구성하면서, 오히려 테스트 코드 작성법보다 Spring Security 구조와 동작 방식에 대한 학습이 더 많았던 것 같습니다. 약간은 이상하게 느껴지기도 했지만, 원하는 테스트 환경을 만들기 위한 과정 속에서 많은 것을 배울 수 있었던 경험이었습니다.