---
title: "파일 업로드 (Stream, MultipartFile, Pre-signed url, Multipart Upload)"
date: 2025-2-20 17:28:00 +09:00
categories: [스프링부트, Upload]
tags:
  [
    SpringBoot,
    스프링부트,
    File Upload,
    파일 업로드,
    AWS,
    S3,
    Pre Signed url,
    Multipart upload,
    Stream,
    MultipartFile
  ]
---

> 진행했던 프로젝트에서 파일 업로드를 발전시킨 과정에 대해서 써보려고 합니다. 파일은 어떻게 전송되고, Spring Boot에서 어떻게 받아와서 어떤 식으로 저장하는지에 대해서 고려했던 점들과 Locust와 Prometheus + Grafana를 이용하여 파일 업로드 기능의 성능을 모니터링 해보려고 합니다.

<br>

## 1. 파일이 Client에서 서버로 전송되는 과정

<br>

일반적으로 클라이언트측에서 서버로 파일을 전송할 때, `Content-Type: multipart/form-data`를 헤더에 붙여주고 이미지의 바이너리 데이터를 multipart 형식으로 데이터를 구성하여 전송합니다. 일반적인 JSON의 경우 단일 문자열로 하나의 단일 본문에 포함되지만, multipart 요청은 본문이 여러 파트로 나뉘어 파일의 바이너리 데이터, 텍스트 데이터 등 다양한 데이터를 포함할 수 있습니다.

예를 들어, HTTP 헤더가 아래와 같이 있을 때, boundary는 요청 본문 내의 각 파트를 구분하는데 사용됩니다. boundary를 기준으로 json과 image가 본문 내용에 들어가 있는걸 볼 수 있습니다. 마지막 바운더리는 뒤에 --가 붙어 종료됨을 나타냅니다.
```markdown
POST /upload HTTP/1.1
Host: example.com
Content-Type: multipart/form-data; boundary=----Boundary123

------Boundary123
Content-Disposition: form-data; name="meta"
Content-Type: application/json

{
  "title": "이미지",
  "description": "테스트 이미지입니다."
}
------Boundary123
Content-Disposition: form-data; name="file"; filename="image.png"
Content-Type: image/png

[binary data of image.png]
------Boundary123--
```

Spring 서버는 이런 형식의 HTTP 요청을 아래와 같이 Controller를 구성하여 요청을 받습니다.

```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(
        @RequestPart("meta") ImageMeta meta,  
        @RequestPart("file") MultipartFile file) { 
                ...
}
```

<br>

@RequestPart 어노테이션은 mulitpart 요청의 특정 part를 지정해서 받을 때 사용할 수 있습니다.
예를 들어, 위 요청 예시와 같이 JSON과 이미지를 함께보내는 요청의 경우 boundary를 기준으로 각 part가 분리되어있는데, Spring의 MultipartResolver가 요청을 파싱하고 각 part에 붙어있는 ContentType을 바탕으로 파라미터에 매핑됩니다. 

<br>

```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(
        @RequestParam("description") String description,
        @RequestParam("file") MultipartFile file) {...}
```

단일 파일이나 text와 파일이 들어간 요청의 경우 위와 같이 @RequestParam으로도 받을 수 있습니다. 하지만 @RequestParam은 단순 문자열만 받을 수 있고 JSON이 전달되면 객체로 변환하는 기능은 제공하지 않습니다.

파일도 물론 JSON 형식으로 보낼 수 있습니다. 하지만 바이너리 데이터를 Base64 인코딩을 통해 텍스트로 변환해서 전달해야합니다. 이는 원본보다 크기가 약 33% 커지므로 전송 비용이 mulitpart 방식보다 증가한다고 볼 수 있습니다. 

<br>

## 2. Stream 방식 vs MultipartFile 방식

<br>

Spring에서 파일을 받아올 때 사용할 수 있는 방식은 2가지 있습니다. 

첫 번째는 `Stream 방식`으로 HttpServletRequest.getInputStream()을 통해 raw 데이터 스트림을 이용하는 방식입니다. 파일 전체를 메모리에 로딩하지 않고 버퍼의 크기(8kb or 16kb)만큼 입력스트림으로 데이터를 읽어 출력스트림을 통해 저장합니다. 따라서, 대용량 파일 업로드시 메모리에 가해지는 부담을 줄일 수 있습니다. 이미지 저장소로 S3를 사용한다면 stream을 그대로 전달하여 저장할 수 있다는 장점이 있습니다.

하지만, multipart방식과는 다르게 http 본문에 boundary가 없어 하나의 파일만 전송할 수 있습니다. 또한 chunk로 데이터를 읽게되므로 파일 데이터를 모두 읽을 때까지 해당 요청을 맡은 쓰레드는 다른 요청을 처리할 수 없게 됩니다. 만약, 사용자가 큰 파일을 업로드하게 된다면 데이터를 chunk로 읽어서 저장하는 시간만큼 다른 사용자가 대기해야하는 상황이 발생할 수 있습니다.

따라서, stream방식을 사용하려고 한다면 서비스에서 저장되는 파일들의 평균 크기가 매우 중요하다고 생각합니다.

다음은 `MultipartFile 방식`입니다. 해당 방식은 Spring Boot 설정파일에서 다음과 같은 설정을 원하는 값들로 채워서 사용할 수 있습니다.

<br>

```yml
spring:
  servlet:
    multipart:
      enabled: true
      max-file-size: 10MB
      max-request-size: 10MB
      location: C:/Users/ahc70/Desktop/tmp
```

`max-file-size` 옵션은 파일 하나의 최대 크기를 말하고 `max-request-size`는 mulitpart 형식으로 들어오는 본문의 최대 크기를 말합니다.

예를 들어, Client에서 Server로 multipart 형식으로 전달할 때, 본문에 사진 N개를 보내면 N * size가 max-request-size보다 작아야합니다.
제가 만들었던 프로젝트에서는 단일 파일 업로드만 지원했기 때문에 두 사이즈 값을 동일하게 두었습니다. 

`location`의 경우 WAS가 Client로부터 전달 받은 바이너리 데이터를
임시 디렉토리에 저장할 때 사용합니다. 프로젝트에서 S3에 파일을 저장할 경우 InputStream으로 디스크에 저장된 파일을 읽어서 업로드 하게 되는데, 해당 로직 마지막에 close()를
작성하지 않으면 디스크 누수가 발생할 수 있어 이를를 확인하며 사용해야합니다.

또한, MultipartFile의 경우 기본적으로 디스크에 파일을 한번 저장하고 S3에 전송하기 때문에,
요청 하나당 파일 크기와 디스크 용량을 고려해서 사용해야합니다.

두 방법에 대해 실제 테스트를 진행하기 전에 비교해 보았을 때, MultipartFile 방식이 더 빠르게 요청을 처리할 것이라고 생각했습니다. 또한
큰 데이터를 처리할 수록 더 빠를 것이라고 생각했습니다., 따라서 MultipartFile 방식이 Stream보다 제 프로젝트에서는 좋은 성능을 낼 것이다라고 가설을 세우고 
Locust와 Prometheus + Grafana를 이용하여 모니터링을 해보았습니다.

먼저, local환경에서 50MB의 파일데이터를 동시에 5번 전송하는 테스트를 진행했을 때
Stream 방식의 경우 Heap 사용량을 추적하고 싶었지만 유의미한 차이를 보이지 않았고 Locust상의 ResponseTime은 평균적으로 54000ms가 나왔습니다.

![Image](https://github.com/user-attachments/assets/655de60f-94f2-4841-9521-361187d3c240)

MultipartFile의 경우 동일한 환경에서 테스틀 진행했을 때, ResponseTime만 비교해보았습니다.
결과는 아래 사진과 같이 나왔고 평균 22000ms가 걸렸습니다.

![Image](https://github.com/user-attachments/assets/cef4b018-a22b-4329-b8e8-186f8bf240cf)

결론적으로 5명이 각각 50MB의 파일을 업로드할 때 총 3배의 정도의 시간이 걸렸고 이는 파일의 크기가 커질수록 더 증가할 것이라는 결론을 내릴 수 있었습니다.
MultipartFile은 그만큼 일정 시간동안 디스크 사용량이 250MB이기 때문에 이 점을 고려해서 사용해야 한다고 생각합니다.

실제 EC2 서버에서는 어떤 상황을 보여줄지 궁금하여 t3.micro(2vCPU, 1g mem)의 사양에서 테스트 해보았습니다.
환경은 다음과 같이 설정하고 진행하였습니다.

![Image](https://github.com/user-attachments/assets/a665ee8e-fc51-47f1-8764-e06c7a5f5120)

노트북에서 요청을 보내고 ec2 내부의 Spring 애플리케이션은 S3와 VPC 엔드포인트를 이용해 통신하는 환경으로
테스트를 진행해 보았습니다. 테스트는 5개의 50MB파일을 동시에 업로드하는 것으로 살펴보았습니다.

Stream의 경우 

![Image](https://github.com/user-attachments/assets/75d1fd4e-d66e-493a-9ac0-0e6442406d8a)

![Image](https://github.com/user-attachments/assets/7c2a337a-8bdd-45db-afc2-09b2d43737bb)

![Image](https://github.com/user-attachments/assets/9a3e52f9-ff9d-4dca-88ec-42b054554409)

결과를 살펴보면, ResponeTime은 평균 36000ms가 나왔고 메모리의 경우 요청이 들어간 시점에 Heap사용량 변화를 살펴볼 수 있었습니다. 또한 해당 요청 시간에 쓰레드가 총 5개의 쓰레드가 Busy 상태임을 확인했습니다.

<br>

Mulitpart의 경우도 살펴보면

![Image](https://github.com/user-attachments/assets/b06fdaf6-051e-4caf-9755-2bd1e0efb981)

![Image](https://github.com/user-attachments/assets/df627446-df5d-4673-9aac-d9a83bf0ff86)

![Image](https://github.com/user-attachments/assets/d7c70a59-fc4c-4c77-b774-d40ccf969258)

ResponseTime은 평균 23000ms가 나왔고 디스크 사용량은 100MB정도 사용했습니다. 그리고 역시 해당 시간에 총 5개의 쓰레드가 Busy 상태임을 확인할 수 있었습니다. 

두 방식은 실제 모니터링을 해보기 전에 생각했던 상황대로 동작했습니다. 확실히 MultipartFile 방식이 ResponseTime 측면에서 Stream보다 좋은 선택이라고 볼 수 있었고 disk의 경우 250MB를 사용할 거라 예상했는데 100MB가 찍힌 이유는 매트릭 수집시점과 실제 요청이 처리되는 시점에 차이가 있어서 이런 결과가 나왔다고 판단하였습니다.

결론적으로, 적은 메모리 리소스(1g mem)로 업로드 기능을 구현한다면 메모리 사용보다 디스크를 사용하는 MulitpartFile 방식이 더 좋은 선택이라고 판단하였고 또한 동시에 요청이 들어오는 상황에서 ResponsTime도 더 우수하다고 판단하여 초기에 MultipartFile 방식으로 구현하여 사용하였습니다.

<br>

## 3. 이미지 파일이 서버를 거치면..

<br>

위 두 방식을 사용할 경우 무조건 이미지 바이너리 파일은 서버를 한 번 거쳐서 최종 저장소인 S3에 저장되는 구조입니다. 따라서 네트워크 I/O 비용이 다음과 같이 2번 발생하게 됩니다.

1. Client환경 -> Spring 서버
2. Srping 서버 -> S3 Bucket

VPC 엔드포인트를 사용한다면 어느정도 Latency를 줄여줄 수 있다고 판단하지만, 네트워크 I/O는 결국 2번 발생하고 또한 서버에 가해지는 부하는 여전히 존재한다고 판단하였습니다.

이런 상황 속에 도입할 수 있는 방법으로는 파일 업로드만 처리하는 서버로 분리하는 방식과 Client에서 S3에 직접 파일을 업로드하는 방식이 있었습니다.

파일 업로드 전용 서버를 이용한다면 파일 업로드에 들어가는 비용과 서비스에 들어가는 비용이 분리되어 서로 영향을 주지 않는 장점이 있었고 Client에서 S3 업로드하는 방식은 불필요하게 서버를 거치는 구조를 없앨 수 있었습니다.

두 가지 선택 중 금액을 고려했을 때, Client에서 S3에 업로드하는 방식을 선택하게 되었습니다. S3 비용을 고려할 때는 아래 AWS 문서를 바탕으로 결정하였습니다.

https://aws.amazon.com/ko/s3/pricing/?nc=sn&loc=4

<br>

## 4. Pre-Signed URL 방식

<br>

Pre Signed Url 방식은 서버에서 AWS sdk를 사용하여 특정 객체에 대한 pre signed url을 생성해서 Client에게 전달하는 방식으로 진행됩니다.
이 url은 지정된 유효 기간 동안 유효하며 해당 기간 내에 url을 전달받은 Client는 파일을 지정된 S3 주소로 직접 저장할 수 있습니다. 따라서, 파일은 서버를 거치지 않고 S3에 저장될 수 있어 네트워크 I/O 비용을 감소시킬 수 있습니다.

![Image](https://github.com/user-attachments/assets/03214fff-917c-45bd-b1b2-9d94a1fc06f2)

위 그림에서 pre signed url과 함께 전달되는 객체 키는 해당 이미지를 pre signed url로 저장을 성공하면 client 화면에 이미지를 바로 보여주기 위한 접근 url입니다. 5번 과정에서 이미지를 전송할 때 기본적인 pre sgined url 방식을 사용하면 put메서드에 binary 데이터를 전송해야합니다. Post로 업로드 하는 방식의 경우 multipart/form-date로 데이터를 전송하면 됩니다.

S3를 생성할 때, 정책을 통해 퍼블릭 액세스를 제어하도록 설정하였고 버킷의 특정 디렉토리에 대해서만 getObejct 권한을 모든 사용자에게 허용해서 구현하였습니다. 

이 방식을 사용했을 때 Response Time은 얼마나 줄지 파악하기 위해 똑같이 5개의 50MB 파일을 동시에 업로드하는 시나리오를 진행해 보았고 그 결과는 다음과 같습니다.

먼저 실제 서버에서 처리하는 작업인 pre signed url 발급 로직의 Response Time을 파악해 보았고 여기에 추가적으로 Client가 이미지 업로드를 완료하는 동작까지 수행한 시점의 Response Time을 측정해 보았습니다.

![Image](https://github.com/user-attachments/assets/54a540d2-cdce-4e6e-9c2a-d72d450b66c0)

먼저 Server에서 PresignedUrl을 전달할 때 다음과 같은 Response Time을 얻을 수 있었습니다. 이 지표를 보면 서버의 부하는 확실하게 감소됬음을 알 수 있습니다.

![Image](https://github.com/user-attachments/assets/5e825a8b-d37e-46ec-b328-232b6374c7fd)

실제 파일이 Client에서 S3로 업로드될 때 걸리는 시간은  MultipartFile방식과 비교했을 때 비슷할 수준의 시간이 걸렸습니다. 


50MB 정도 되는 크기의 데이터 혹은 이보다 큰 데이터를 업로드할 때는 S3의 Multipart Upload를 사용하는게 더 좋은 방식이라고 생각합니다. 이 방식은 파일을 여러 개의 Part로 나누어 각각 스트리밍 방식으로 전송합니다. 따라서 대용량 파일 업로드에 적합하지만 구현이 상대적으로 복잡하다는 단점이 있습니다.
아마, 이 Multipart Upload까지 도입한다면 Client측에서 S3에 업로드 하는 과정이 더 빨라질 것이라고 예상합니다.

제가 진행하는 프로젝트는 이미지 위주의 파일이 대다수이기 때문에 평균 크기가 1MB도 안되는 상황이었고 대용량 업로드를 요구하는 기능이 존재하지 않아 구현 복잡도가 높은 기술을 굳이 도입하지 않아도 된다고 판단하여 Pre-Signed URL 방식까지만 도입하였습니다.

<br>

## 5. 정리

<br>

최종적으로 `Pre-Signed URL 방식`을 결정하였는데 그 근거로는 다음과 같습니다.
먼저, 서비스에 업로드 되는 파일의 크기가 작은 상황에서 대용량 파일을 업로드하는데 특화된 Multipart Upload 방식은 `구현의 복잡도`를 고려하여 선택하지 않았습니다.

다음으로, 실제 Stream과 MultipartFile을 구현하여 테스트해보며 얻은 결과를 바탕으로 파일 업로드로 인해 `다른 서비스에 부하`가 갈 수 있다는 점과 파일이 총 2번 네트워크를 타고 움직이는 `불필요한 네트워크 I/O`가 발생한다는 점을 근거로 pre signed url을 선택하였습니다.

이번 글에서 업로드 기능에 대한 점진적 부하 테스트를 진행해 보진 못했습니다. 따라서, 이후에 실제 프로젝트에서 사용이 가장 많이되었던 api에 대한 부하테스트를 진행하며 이미지 업로드에 대한 부하테스트도 진행하려고 합니다. 

처음으로 Spring에서 Actuator를 바탕으로 Metric을 수집하여 원하는 리소스가 어떻게 사용되는지 Grafana로 지켜보는 과정을 진행해보았는데, 이런 좋은 모니터링 도구들이 오픈소스로 존재해 학습의 기회가 개방되어 있는 IT분야의 따뜻한 면모를 느낄 수 있었습니다.  


