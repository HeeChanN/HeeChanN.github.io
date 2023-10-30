---
title: "스프링부트로 카카오페이 api 이용하기"
date: 2023-10-30 20:53:00 +09:00
categories: [스프링부트,외부 API]
tags:
  [
    스프링부트,
    spring boot,
    카카오페이 api,
    RestTemplate
  ]
---


> [카카오페이 Api 문서](https://developers.kakao.com/docs/latest/ko/kakaopay/single-payment)
안녕하세요. 먼저 제 글을 보시기 전에 Api 문서를 보고 직접 해석해 가며 구현해 보면 좋을 것 같아서 앞단에 첨부 했습니다. 
뒤에서 나올 스프링 부트에서 보낼 **Request**와 **Response**는 모두 이 Api 문서를 참고하여 작성하였습니다!

![kakao](https://github.com/HeeChanN/HeeChanN/assets/88177732/16fe8e48-831d-46a4-8c09-b5569cce4a8e)

<br>


## 1. 카카오페이 API
---

<br>

먼저 공식 Api 문서에 이런 문구가 적혀있습니다.
![api](https://github.com/HeeChanN/HeeChanN/assets/88177732/f55aee14-9c81-4c27-a37a-b216e7fa7649)

이 말은 Spring boot에서 해당 기능을 이용하고자 한다면 서버측에서만 카카오 Api 와 소통해야 된다는 말입니다. 즉, React에서 직접 요청이 불가능하다는 말로도 해석 할 수 있습니다. 

따라서, 만약 React에서 해당 기능을 모두 구현한다면 서버에서 직접 요청이 불가능하고 서버에 적용 내용만을 전달하며 구현할 수 있을것이라고 예상해 볼 수 있습니다.

이 글에서 React는 결제할 정보 제공과 rediret url 연결만 시켜주고 서버에서 결제 관련 API 이용을 모두 처리하는 과정을 다룰 예정입니다.

<br>

## 2. 동작 과정
---

<br>

![kakao2](https://github.com/HeeChanN/HeeChanN/assets/88177732/d25ec143-cde5-4407-8d75-0eadd1123b6a)

구현하기 앞서 위의 그림을 간단하게 설명하겠습니다.

1. React 에서 먼저 결제정보 ( 상품이름, 결제금액 ) 을 서버로 넘깁니다. 

2. 서버는 해당 내용과 유저정보를 바탕으로 카카오페이 API에 보낼 Request를 만들어 https://kapi.kakao.com/v1/payment/ready 로 요청을 보냅니다. 이 때 **Request**는 위에서도 말했듯이 공식 문서에 자세하게 적혀있습니다.

3. 카카오페이 API에서 해당 요청을 검증하고 올바른 요청이라면 Response를 보냅니다.

4. 이 Response에는 next_redirect_pc_url이 담겨있는데 해당 url은 클라이언트를 결제단으로 이동시켜주는 url입니다. 

5. 따라서 이 url을 받은 React가 사용자를 redicet 시켜주는게 5번 과정입니다.

6. 기본 설정은 http://localhost:8080/payment/success&pg_token=토큰 이지만 유저 정보를 바탕으로 결제서비스를 진행하기 위해 저는 유저 id를 추가해 주었습니다. 
이 단계에서는 2번 단계와 마찬가지로 https://kapi.kakao.com/v1/payment/approve에 Request를 보내는 과정입니다.

7. Api 문서에 적힌 Response가 돌아옵니다.

8. 해당 정보를 바탕으로 서비스 로직을 구현하고 해당 내용을 React에게 Response를 전달하는 과정입니다.

<br>

## 3. 구현
---

<br>

먼저 진행하기 앞서 https://developers.kakao.com 에서 애플리케이션을 만들어야 합니다. 
해당 링크로 들어가 오른쪽 상단에 

내 애플리케이션-> 애플리케이션 추가하기 클릭 후

앱 이름과 사업자 명을 입력하시면 끝! 생성후 해당 앱에 들어가면 아래와 같이 4개의 정보가 나올텐데 카카오페이 Api에서는 Admin key만 이용하니 해당 Admin Key를 복사해 주세요!

![key](https://github.com/HeeChanN/HeeChanN/assets/88177732/5624e8b8-0fa2-4d89-b39c-d7115ecb84af)

<br>

> ### 1. application-pay.yml 파일 생성

위에서 만든 Admin key를 이용하여 application-pay.yml 파일에 아래와 같이 작성해줍니다. 

``` yml
pay:
  admin-key: 앱 admin key
```

Service단에 그냥 변수로 선언하고 사용해도 되지만 저는 프로젝트에 사용해야 하기에 yml파일을 따로 설정하여 만들었습니다.

<br>

> ### 2. 카카오 페이 API를 이용하기 위한 Request와 Response 

먼저, 이 단계의 끝에 만들어질 디렉토리를 소개하며 시작하겠습니다.
![](https://velog.velcdn.com/images/ahc700/post/8f3c900b-0970-4321-88b1-2487a26b890b/image.JPG)

총 2번의 Requset를 카카오페이 API에 보내고 받는 응답을 저장할 Response와 Request들로 차례로 코드와 함께 설명 하겠습니다.

<br>

- **PayInfoDto**

``` java
@Getter
public class PayInfoDto {
    private int price;
    private String itemName;
}
```
이 Dto는 이 결제시스템 동작과정 첫 시작부분에 사용되는 Dto이다. React에게서 받아오는 내용으로 가격과, 상품이름 받아오도록 설정하였습니다. 카카오페이 API에 Request로 보내고 싶은 정보를 더 추가해서 Custom 하셔서 사용하시면 됩니다.


<br>


- **MakePayRequest**

``` java
@Component
@RequiredArgsConstructor
public class MakePayRequest {

    public PayRequest getReadyRequest(Long id, PayInfoDto payInfoDto){
        LinkedMultiValueMap<String,String> map=new LinkedMultiValueMap<>();

        /** partner_user_id,partner_order_id는 결제 승인 요청에서도 동일해야함 */
        String memberId=id+"";

        String orderId="point"+id;
        // 가맹점 코드 테스트코드는 TC0ONETIME 이다.
        map.add("cid","TC0ONETIME");
        
        // partner_order_id는 유저 id와 상품명으로 정하였다.
        // 해당내용은 자유롭게 정하시면 됩니다.
        // 중요한점은 다음 결제 승인 정보를 얻을 때
        // 아래 partner_order_id,partner_user_id 가 동일해야 합니다.
        map.add("partner_order_id",orderId);
        map.add("partner_user_id","본인의 서비스명");
        
        // 리액트에서 받아온 payInfoDto로 결제 주문서의 item 이름을
        // 지어주는 과정입니다.
        map.add("item_name",payInfoDto.getItemName());
        
        //수량
        map.add("quantity","1");
       	
        //가격
       map.add("total_amount",payInfoDto.getPrice()+"");
       
       	//비과세 금액
        map.add("tax_free_amount", "0");
        
        // 아래 url은 사용자가 결제 url에서 결제를 성공, 실패, 취소시 
        // redirect할 url로 위에서 설명한 동작 과정에서 5번과 6번 사이 과정에서  
        // 나온 결과로 이동할 url을 설정해 주는 것입니다.
        map.add("approval_url", "http://localhost:8080/payment/success"+"/"+id); // 성공 시 redirect url
        map.add("cancel_url", "http://localhost:8080/payment/cancel"); // 취소 시 redirect url
        map.add("fail_url", "http://localhost:8080/payment/fail"); // 실패 시 redirect url

        return new PayRequest("https://kapi.kakao.com/v1/payment/ready",map);
    }

    public PayRequest getApproveRequest(String tid, Long id,String pgToken){
        LinkedMultiValueMap<String,String> map=new LinkedMultiValueMap<>();

        String orderId="point"+id;
        // 가맹점 코드 테스트코드는 TC0ONETIME 이다.
        map.add("cid", "TC0ONETIME");
        
        // getReadyRequest 에서 받아온 tid
        map.add("tid", tid);
        map.add("partner_order_id", orderId); // 주문명
        map.add("partner_user_id", "본인의 서비스명");
        
        // getReadyRequest에서 받아온 redirect url에 클라이언트가
        // 접속하여 결제를 성공시키면 아래의 url로 redirect 되는데
        //http://localhost:8080/payment/success"+"/"+id
        // 여기에 &pg_token= 토큰값 이 붙어서 redirect 된다.
        // 해당 내용을 뽑아 내서 사용하면 된다.
        map.add("pg_token", pgToken);

        return new PayRequest("https://kapi.kakao.com/v1/payment/approve",map);
    }
}
```

먼저 동작 과정에서 설명한 2번 단계로 **결제 고유번호 tid**와 클라이언트를 보낼 **next_redirect_pc_url** 을 얻기 위한 **getReadyRequest()** 메서드 입니다.
해당 메서드는 공식 API 문서의 Request를 바탕을 만들어서 해당 문서의 설명을 참고하는 것이 더 정확할 것이다. 
**getApproveRequest()** 메서드는 클라이언트의 결제 승인 정보를 받아오기 위한 메서드 입니다.
위의 코드에 주석을 자세하게 달아 놓았으니 해당 내용을 참고해주세요!

추가로 설명할 내용은 다른 블로그와 달리 저는 성공시 redirect url에 유저 아이디를 추가했다는 점입니다. **getApproveRequest()**에서 tid와 orderId를 이용해야 하는데 해당 내용을 카카오 API에서 success url로 redirect 시켜주었다면 따로 얻을 방법이 없어서 저는 tid를 member db에 저장하고 해당 member id를 이용하여 partner_order_id 와 tid를 추가해 주는 방법을 사용하였습니다.

<br>
<br>

- **PayRequest**

``` java
@Getter
@AllArgsConstructor
public class PayRequest {
    private String url;
    private LinkedMultiValueMap<String,String> map;
}
```

url은 요청을 보낼 카카오 API URL이고 map은 해당 요청에 담을 Request입니다. 


<br>
<br>

- **PayReadyResDto**

``` java
@Getter
public class PayReadyResDto {
    private String tid; // 결제 고유 번호
    private String next_redirect_pc_url; // pc 웹일 경우 받는 결제 페이지
    private String created_at;

}
```
**getReadyRequest()**를 카카오페이 API에 Request로 보내면 얻는 Response를 Java 객체로 변환하기 위한 Dto입니다.

저는 pc 웹을 만들어서 pc_url만 받았지만 공식문서의 URL을 참고하여 얻고 싶은 Response를 Custom 하시면 됩니다. tid는 필수로 얻으셔야하고 url도 하나는 무조건 얻으셔야 합니다!

<br>
<br>

- **PayApproveResDto , Amount**

``` java
public class PayApproveResDto {
    private Amount amount; // 결제 금액 정보
    private String item_name; // 상품명
    private String created_at; // 결제 요청 시간
    private String approved_at; // 결제 승인 시간

}
```

``` java
@Getter
public class Amount {
    private int total; // 총 결제 금액
    private int tax_free; // 비과세 금액
    private int tax; // 부가세 금액
}
```

**getApproveRequest()** 를 카카오페이 API에 Request로 보냈을 때 받는 Response를 위한 객체 Dto 입니다. 이것 역시 공식 문서에서 직접 Custom 하여 작성하시면 됩니다.


> ### 3. Controller, Service

- **Controller**

``` java
@RestController
@RequiredArgsConstructor
@RequestMapping("/payment")
public class KakaoPayController {

    private final KakaoPayService kakaoPayService;

    /** 결제 준비 redirect url 받기 --> 상품명과 가격을 같이 보내줘야함 */
    @GetMapping("/ready")
    public ResponseEntity<?> getRedirectUrl(@RequestBody PayInfoDto payInfoDto) {
        try {
            return ResponseEntity.status(HttpStatus.OK)
                    .body(kakaoPayService.getRedirectUrl(payInfoDto));
        }
        catch(Exception e){
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(new BaseResponse<>(HttpStatus.INTERNAL_SERVER_ERROR.value(), e.getMessage()));

        }
    }
    /**
     * 결제 성공 pid 를  받기 위해 request를 받고 pgToken은 rediret url에 뒤에 붙어오는걸 떼서 쓰기 위함
     */
    @GetMapping("/success/{id}")
    public ResponseEntity<?> afterGetRedirectUrl(@PathVariable("id")Long id,
                                                 @RequestParam("pg_token") String pgToken) {
        try {
            PayApproveResDto kakaoApprove = kakaoPayService.getApprove(pgToken,id);

            return ResponseEntity.status(HttpStatus.OK)
                    .body(kakaoApprove);
        }
        catch(Exception e){
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(new BaseResponse<>(HttpStatus.INTERNAL_SERVER_ERROR.value(),e.getMessage()));
        }
    }

    /**
     * 결제 진행 중 취소
     */
    @GetMapping("/cancel")
    public ResponseEntity<?> cancel() {
        return ResponseEntity.status(HttpStatus.EXPECTATION_FAILED)
                .body(new BaseResponse<>(HttpStatus.EXPECTATION_FAILED.value(),"사용자가 결제를 취소하였습니다."));
    }

    /**
     * 결제 실패
     */
    @GetMapping("/fail")
    public ResponseEntity<?> fail() {

        return ResponseEntity.status(HttpStatus.EXPECTATION_FAILED)
                .body(new BaseResponse<>(HttpStatus.EXPECTATION_FAILED.value(),"결제가 실패하였습니다."));

    }

}
```

<br>

- **Service**

``` java
@RequiredArgsConstructor
@Service
@Slf4j
public class KakaoPayService {
    private final MakePayRequest makePayRequest;
    private final MemberRepository memberRepository;

    @Value("${pay.admin-key}")
    private String adminKey;


    /** 카오페이 결제를 시작하기 위해 상세 정보를 카카오페이 서버에 전달하고 결제 고유 번호(TID)를 받는 단계입니다.
     * 어드민 키를 헤더에 담아 파라미터 값들과 함께 POST로 요청합니다.
     * 테스트  가맹점 코드로 'TC0ONETIME'를 사용 */
    @Transactional
    public PayReadyResDto getRedirectUrl(PayInfoDto payInfoDto)throws Exception{
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String name = authentication.getName();


        Member member=memberRepository.findByEmail(name)
                .orElseThrow(()-> new Exception("해당 유저가 존재하지 않습니다."));
                
        Long id=member.getId();


        HttpHeaders headers=new HttpHeaders();

        /** 요청 헤더 */
        String auth = "KakaoAK " + adminKey;
        headers.set("Content-type","application/x-www-form-urlencoded;charset=utf-8");
        headers.set("Authorization",auth);

        /** 요청 Body */
        PayRequest payRequest=makePayRequest.getReadyRequest(id,payInfoDto);

        /** Header와 Body 합쳐서 RestTemplate로 보내기 위한 밑작업 */
        HttpEntity<MultiValueMap<String, String>> urlRequest = new HttpEntity<>(payRequest.getMap(), headers);

        /** RestTemplate로 Response 받아와서 DTO로 변환후 return */
        RestTemplate rt = new RestTemplate();
        PayReadyResDto payReadyResDto = rt.postForObject(payRequest.getUrl(), urlRequest, PayReadyResDto.class);

        member.updateTid(payReadyResDto.getTid());


        return payReadyResDto;
    }

    @Transactional
    public PayApproveResDto getApprove(String pgToken, Long id)throws Exception{

        

        Member member=memberRepository.findById(id)
                .orElseThrow(()->new Exception("해당 유저가 존재하지 않습니다."));

        String tid=member.getTid();

        
        HttpHeaders headers=new HttpHeaders();
        String auth = "KakaoAK " + adminKey;

        /** 요청 헤더 */
        headers.set("Content-type","application/x-www-form-urlencoded;charset=utf-8");
        headers.set("Authorization",auth);

        /** 요청 Body */
        PayRequest payRequest=makePayRequest.getApproveRequest(tid,id,pgToken);

        
        /** Header와 Body 합쳐서 RestTemplate로 보내기 위한 밑작업 */
        HttpEntity<MultiValueMap<String, String>> requestEntity = new HttpEntity<>(payRequest.getMap(), headers);

        // 요청 보내기
        RestTemplate rt = new RestTemplate();
        PayApproveResDto payApproveResDto = rt.postForObject(payRequest.getUrl(), requestEntity, PayApproveResDto.class);

      

        return payApproveResDto;


    }


}
```
 

<br>

Controller의 구성은 React에서 상품이름과 상품 가격을 전달 받을 **/payment/ready** 와 해당 과정에서 설정한 url로 결제 승인 과정에서 redirect 시키므로 각각의 **/success, /cancel, /fail**을 만들어 주었다.
Return에서 예외처리를 위해 **ResponseEntity<?>**를 이용하였고 자신의 팀원들과 상의하여 Response는 만들어 주시면 됩니다! 저는 따로 만든 BaseResponse와 Dto 그대로 Response를 보내주었습니다.


<br>


Service에서는 **RestTemplate**를 이용하여 카카오페이 API에 Request를 보내고** postForObject**를 이용하여 바로 객체화 시켜주었다. **adminKey**는 구현 앞단에서 설정한 yml파일에서 가져온 것이고 앞서 설명하였듯이 저는 tid를 member 엔티티에 저장하여 이후에 /success에서 해당 tid를 이용하기 위해 MemberRepository를 추가로 사용하였습니다.

저는 따로 유저의 결제 승인 정보를 받아온 뒤 서비스 Logic을 구현하지 않았지만 결제 내역을 바탕으로 각자의 서비스 Logic을 구현 하시면 됩니다!

<br>

        
``` java
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String name = authentication.getName();
```

이 부분은 JWT를 이용하여 프로젝트를 구현하여서 Header에 담긴 인증 정보로 Member의 이메일을 가져오는 과정입니다. 이 이메일로 Member를 db에서 가져오고 수정하고 저장합니다. 각자의 상황에 맞게 사용하시면 됩니다. **중요한 점**은 해당 요청을 보낸 Member의 정보를 이용해야 된다는 점입니다. 

<br>

## 4. PostMan을 이용한 테스트

아래는 PostMan에서 특정 유저로 회원가입후 로그인한뒤 결제 서비스를 이용하는 테스트 입니다.

![postman](https://github.com/HeeChanN/HeeChanN/assets/88177732/2df2a819-3210-4644-8ec1-d1c6d46cc60c)


/payment/ready 로 가격과 상품 이름을 전달하면 위에서 설정한 PayReadyResDto가 응답으로 나오는걸 확인 하실 수 있습니다.

여기서 해당 URL에 접속시 아래와 같이 화면이 뜨고 
![pay](https://github.com/HeeChanN/HeeChanN/assets/88177732/bd45b8ff-0aa2-4dbb-b22d-29466746f1d1)

휴대폰을 이용하여 결제를 진행하면 설정한 Response가 뜨는걸 확인할 수 있습니다.

![web](https://github.com/HeeChanN/HeeChanN/assets/88177732/d7f9384d-c82c-4210-a4f6-c01602d7fe63)

<br>

## 5. React와 통신

<br>

Spring boot로 잘 작동하는걸 확인한 후 팀원과 통신으로 테스트를 하는 과정에서 처음 설명한 순서도에서 5번의 next_redirect_url_pc로 클라이언트를 연결해 주는 과정에서 react의 request는 끝나는걸 확인 할 수 있었다. 이후 부터는 카카오 결제 후 해당 결제를 성공,취소,실패에 따라 백엔드 Controller로 연결되는 구조였다. 
따라서 해당 Controller에서 Response를 보내면 http://백엔드서버:포트번호/success/{id} 로 Response가 전달되는 걸 확인 할 수 있었다. 이 Response는 애초에 React가 보낸 요청이 아니기에
( 카카오페이 api가 만든 url에서 redirect 되어서 /success/{id}로 왔기에 )
Response가 그냥 서버단에 json 그 자체로 표시되는 거라고 생각하였다. 

따라서 해커톤 개발에서 서비스가 끊기지 않고 실행되어야 됬기에 일단 정보를 넘겨주는 것보다 다시 마지막 컨트롤러단에서 리액트 서버로 리다이렉트 시켜주는 것이 맞다고 생각하여 아래와 같이 수정하였다.  
 

``` java
@GetMapping("/success/{id}")
    public void afterGetRedirectUrl(HttpServletResponse response, @PathVariable("id")Long id,
                                              @RequestParam("pg_token") String pgToken) {
        try {
            PayApproveResDto kakaoApprove = kakaoPayService.getApprove(pgToken,id);


            response.sendRedirect("http://localhost:3000/myPage/:"+id);
            

        }
        catch(Exception e){
			
        }

    }

```

너무 급하게 해결하였기에 나중에 다른 방법을 찾아서 내용을 추가해야 겠다고 생각하였다. 위에처럼 구현시 아래 사진과 같이 실행이 된다. 

순서는 

1. 카카오페이에서 유저가 결제를 완료하면 백엔드 /payment/success/{id}로 redirect 된후


 
2. 해당 컨트롤러에서 결제 관련 서비스 로직을 구현한 후 유저를 다시 react 도메인 내 정보 페이지로 redirect 시키는 구조이다. 



<br>


##  6. 마치며..


카카오 페이 API를 스프링 부트로 이용하기 위해 위 과정을 진행하며 클라이언트가 결제 창에 들어가서 결제를 성공하고 Redirect로 localhost:8080/payment/success 로 이동하는 과정에서 어떻게 tid와 유저 정보를 이용할지 고민을 많이 하였습니다. 

다른 블로그를 보면 Model에 담아서 Session으로 이용하는 방법을 보았는데 저는 React와 통신을 하며 해결 해야 했기에 mvc 방법으로 해결할 수 없었습니다. 

따라서 http://localhost:8080/payment/success 를 request에 담을 때 애초에 member id를 PathVariable로 담자는 생각을 하였고 
"http://localhost:8080/payment/success"+"/"+id 이런 결과가 탄생하게 되었습니다. 

또한 React와 통신 과정에서 원래 방식으로 서비스를 구현할 때 어떻게 해결해야할지 고민을 정말 많이 했던 것 같다. 팀 프로젝트를 진행하니 혼자 개발할 때와는 다른 문제점을 정말 많이 느낄 수 있었는데 해결방법은 총 2가지라고 생각하였습니다.

첫 번째 방법은 팝업창을 이용하여 유저를 새로운 창으로 보내는 방법으로 React와 급하게 협업하느라 이 방법을 사용하지 못했다.
 
두 번째 방법은 우리 팀이 급하게 구현한 위와 같은 방법으로 백엔드 /payment/success/{id}에서 리다이렉트 하는 방법입니다. 그래도 팀 프로젝트를 진행하며 겪을 수 있는 문제점을 어떤 도움을 받지 않고 해결해 나갔다는 점에서 좋은 경험이 될 수 있었습니다.


하지만 오직 구현에 목적을 두고 만들었기에 어떤 문제점이 있을지 모르고 아직 배움이 부족하기에 다른 좋은 방법이나 문제점이 있다면 댓글로 알려 주시면 감사하겠습니다 ! 

추가적으로 카카오페이 API를 서버에서 이용하기 위해 구현하다 보며 느낀점은 그냥 React에서 카카오페이 API와 통신하며 나온 결과를 서버에 보내 해당 내용을 바탕으로 서비스 로직만 구현하는 방법이 더 효율적일 것 같다는 생각이 들었습니다. 다음 프로젝트에는 프론트 팀원에게 맡겨봐도 좋을 것 같다는 생각과 함께 이번 글을 마무리 하겠습니다.
 
부족한 글 끝까지 읽어주셔서 감사합니다!