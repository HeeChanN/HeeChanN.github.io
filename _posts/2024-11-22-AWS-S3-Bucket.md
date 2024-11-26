---
title: "S3 Bucket 생성 및 각 설정에 대한 이해"
date: 2024-11-22 18:00:00 +09:00
categories: [AWS,S3]
tags:
  [
    AWS,
    S3,
    이미지 업로드,
    Bucket,
    버킷,
    정적 파일 관리,
    버킷 차단 설정,
    객체 소유권
  ]
---


<br>

> 처음 버킷을 사용할 때 급하게 구현을 진행하느라 각 설정 옵션에 대한 이해 없이 버킷을 사용하고 있었습니다. 매번 버킷을 생성할 때마다 제가 생성하는 옵션에 대한 이해없이 생성해서 보안적으로도 이해도가 낮고 또 매번 블로그를 참고하여 생성해서 확신을 갖고 사용할 수 없었습니다. 왜 매번 이렇게 버킷 설정 옵션에 대해서 찾아보게 되고 또 각 옵션에 대해 이해가 가지 않았는지 생각해 보았는데, 그 이유로 S3버킷에서 사용하는 용어에 대한 정리가 안 되어있다는 점을 파악할 수 있었습니다. 따라서, 이번 기회에 S3에서 사용하는 용어정리를 바탕으로 버킷을 생성하는 옵션들에 대해 근거를 만들어 보려고 합니다.

<br>

기본적으로 AWS 계정을 가지고 있다고 가정하고 시작하겠습니다. 또한 애플리케이션에서 접근한다면 따로 S3 버킷에 접근하는 IAM을 만들고 해당 IAM의 access-key와 secret-key를 발급받아 사용해야합니다.


## 1. 처음 S3 버킷 생성시 겪었던 문제점들

<br>

버킷을 만들 때 제일 먼저 수행하는 것은 버킷 이름을 설정하는 것입니다. 버킷 이름 같은 경우 현재 다른 사람들이 생성한 버킷이름과 중복되지 않는다면 쉽게 설정할 수 있습니다. 아래 사진과 같이 버킷 이름 항목에 작성하면 됩니다.
![bucket image](https://github.com/user-attachments/assets/a84c5817-70a9-4dc4-9cf8-2e264e1b2a6f)

<br>

문제점은 다음 항목 부터 나옵니다. 객체 소유권을 설정하는 것인데 생성 창에 나와있는 설명은 S3를 학습하지 않은 사용자에게 매우 불친절하게 느껴질 수 밖에 없습니다. 권장은 ACL 비활성화인데 ACL활성화와 구체적으로 차이점이 뭔지도 모르겠고 일단 기본적으로 용어에 대한 정리가 안되어 있어서 이해하기 쉽지 않았습니다. 객체의 소유권은 계정 소유권이라고 치고 그럼 엑세스 제어 목록 (ACL)은 무엇이고 지칭하는 대상 (버킷, 객체)들은 뭘 말하는 거지?라는 생각을 하게 되었습니다.
![객체 소유권](https://github.com/user-attachments/assets/c10bc428-fa2e-4eab-a11e-623920bfb123)

<br>

객체 소유권뿐만 아니라 버킷의 퍼블릭 액세스 차단 설정과 관련해서는 매번 이해도 없이 설정하고 매번 설정을 바꾸는데 이유도 모르고 사용하고 있었습니다. 여기서도 각 옵션에 대한 설명들이 와닿지 않았고 그 이유는 ACL과 Public Access 차단 그리고 정책같은 것들에 대한 개념이 잡혀있지 않아서라고 저는 판단하였습니다.
![Public Access 차단 설정](https://github.com/user-attachments/assets/a0aeadfb-4fcb-42e1-b32c-cebe5e6d791c)

<br>

이런 이해도 부족의 이유는 용어 정리 부족이라고 판단하였고 따라서 이번 기회에 용어정리와 함께 각 옵션 조합을 설정했을 때 어떻게 작동하는지 가정해보고 실제 사용해보며 S3에 대한 이해도를 높여보려고 합니다.

<br>

## 2. S3 버킷 용어 정리

<br>

먼저 저는 AWS 공식 문서를 읽어가며 학습을 진행하였습니다. 해당 링크는 아래와 같습니다.

[AWS 문서](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/access-control-block-public-access.html?icmpid=docs_amazons3_console)

가장 먼저 파악한 것은 버킷은 `기본적으로 객체에 대한 Public 접근을 허용하지 않는다`는 것입니다.
여기서 객체는 버킷에 저장한 이미지, 파일과 같은 정적파일을 말합니다. 즉, 아무 설정 없이 저장한 객체에 대해서 외부접근은 불가능하다고 볼 수 있습니다.

그럼 버킷에 저장한 객체에 대해서 접근하기 위해서 필요한 것들이 무엇일까요?

객체 단위부터 본다면 가장먼저 `ACL`이 있습니다. 

`ACL`은 객체를 저장할 때 객체에 부여할 수 있는 액세스 제어 목록입니다. `ACL`에는 private, public-read-write, authenticated-read, public-read등 이 있으며 예시를 본다면 좀 더 이해가 쉬울 수 있습니다.

```console
aws s3api put-object --bucket my-practice-bucket --key <업로드될 객체 키> --body <로컬 파일 경로> --acl public-read
```

<br>

위 명령어는 콘솔창에서 사용할 수 있는 명령어로 s3 버킷에 정적파일을 넣는 명령어입니다. 

이때 `버킷 이름`은 my-practice-bucket이고 업로드될 `객체 키`에는 버킷내의 어떤 경로로 저장될지를 적어주면 되고 (image/profile1.jpg) `body`에는 업로드할 파일의 로컬 위치를 적어주면 된다. 그 후 마지막에 `acl`을 설정할 수 있는데 위 명령어는 ACL 중 public-read를 준 예시입니다. 

이 방식으로 접근 제어를 하게 되면 객체 각각에 접근제어를 부여할 수 있다는 장점이 있습니다.

객체보다 한 단계 더 큰 단위는 `정책`입니다.

```json
{
    "Version": "2012-10-17",
    "Id": "Policy1732173753488",
    "Statement": [
        {
            "Sid": "Stmt17321737125246",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-practice-bucket/*"
        }
    ]
}
```

<br>

버킷 정책을 따로 작성해본 사람이라면 위 JSON구조가 무엇인지 바로 알 것입니다. 해당 정책같은 경우 버킷을 생성하고 권한탭 -> 버킷 정책으로 간 뒤 새로 생성하거나 편집할 수 있습니다. 또한 AWS에서 버킷 정책 생성기를 제공하고 있으므로 자신이 원하는 정책을 손쉽게 만들어서 사용해 볼 수 있습니다.

위 정책은 my-practice-bucket에 존재하는 모든 파일에 대해 모든 주체가 Get 접근을 허용한다는 정책입니다.

이 방식으로는 객체보다 더 큰 단위인 디렉토리 단위로 접근 제어를 수행할 수 있습니다.

이 두가지 접근 제어와 처음에 말한 `버킷의 기본 설정은 Public 접근을 허용하지 않는다`는 개념으로 `버킷의 퍼블릭 액세스 차단 설정`을 이해해 보겠습니다.

<br>

## 3. 버킷의 퍼블릭 액세스 차단 설정에 대한 이해

<br>

![퍼블릭 액세스 차단 설정](https://github.com/user-attachments/assets/a0aeadfb-4fcb-42e1-b32c-cebe5e6d791c)


이 옵션을 보면 항상 저는 항목과 설명으로 이해할 수 없었습니다. 하지만 `ACL`과 `객체` 그리고 `정책`에 대한 이해와 `버킷의 기본 설정은 public 접근을 허용하지 않는다`는 사실을 인지하고 해당 선택지들과 공식문서를 참고한다면 각 선택지가 어떤걸 말하고 있는지 파악할 수 있습니다.

먼저 1번과 2번 선택지입니다. 해당 선택지의 대상은 객체이고 객체에 직접 ACL을 부여하는 상황을 말합니다. 첫 번째는 객체를 저장할 때 ACL을 부여하는 요청을 차단하는 선택지이고 두 번째는 해당 요청으로 객체를 버킷에 저장을 하지만 객체에 부여된 ACL은 무시한다는 선택지입니다. 즉, 두 옵션은 객체에 직접 ACL로 public 접근을 하는 것을 차단하기 위한 선택지였던 것입니다.

3번과 4번은 사실 처음 S3를 사용한다면 제일 이해하기 어려운 말입니다. 하지만 버킷 생성 시점을 한정해서 말하면 정책으로 버킷의 접근제어를 수행하는 상황을 말하는 것이라고 이해하시면 됩니다.

버킷을 생성하고 액세스 포인트라는 것을 만들 경우 똑같이 `액세스 포인트의 퍼블릭 액세스 차단 설정`이라는 것을 진행하게 되는데, 이 때랑 동일하게 해당 UI를 AWS가 사용중이라 3번과 같이 `새 버킷 또는 액세스 지점 정책을 통해 부여된 버킷 및 객체에 대한 퍼블릭 액세스 차단`이라는 말이 나오게 된 것입니다.

3번과 4번의 차이는 1번과 2번의 차이와 동일하게 정책 설정을 완전히 못하게 하는 것과 정책을 설정해도 적용이 되지 않는 것의 차이가 있습니다.

각 선택지에 대해 이해했다면 이미지를 저장하고 해당 이미지를 랜더링하는 예시로 어떤 선택지를 골라야하는지 따져보겠습니다.

버킷은 아무 조건이 없다면 public 접근을 할 수 없습니다. 하지만 저는 버킷에 이미지를 저장해서 Client 화면에 랜더링하고 싶다는 요구사항이 존재했고 객체에 pulbic으로 접근해야했습니다. 순수하게 S3 버킷에 업로드하고 접근한다는 조건에서는 이 요구사항을 지키기 위해 모든 퍼블릭 액세스 차단 설정으로 진행할 수 없습니다. 이런 상황에서 선택사항은 다음과 같습니다.

- 첫 번째는 정책을 생성하지 않고 애플리케이션단에서 이미지 생성시마다 public-access를 부여하는 방식입니다. 모든 이미지에 대해서 직접 객체에 ACL을 부여하는 방식으로 만약 서비스에서 접근제한을 두어야하는 이미지라면 ACL로 모든걸 조절해야합니다. 이 때는 아래와 같이 사용해 볼 수 있습니다. 이 방식은 public 접근을 허용하는 정책을 생성조차 못하게 하고 객체에 ACL을 부여하여 관리하는 방식입니다.<br><br>
![객체 해제](https://github.com/user-attachments/assets/27717a15-f857-4a2b-af43-c195d3e3e318)


- 두 번째는 정책으로 접근제한을 두는 방식입니다. 정책으로 접근제한을 진행할 경우 객체에 ACL을 부여하는 방식과 함께 사용할 수 있습니다. 이 때에는 가장 먼저 정책으로 해당 접근을 허용하거나 차단하고 두번쨰는 객체에 포함된 ACL로 차단/허용을 설정합니다. 따라서 총 2가지 설정이 존재할 수 있습니다. 전부 해제하고 사용하거나 정책 차단 설정을 해제하고 사용하는 방식이 존재합니다. 이렇게 사용하는 경우 버킷을 생성한 뒤 해당 버킷에 정책을 생성하여 접근 제어를 진행할 수 있습니다.<br><br>
![전부 해제](https://github.com/user-attachments/assets/e0ac4b5f-405c-4bd9-bd86-4bf1b812e581)
![정책 해제](https://github.com/user-attachments/assets/b97910d0-f7ba-4df6-a2b1-92ab3cd3149c)
![정책 생성](https://github.com/user-attachments/assets/09d1815c-60f9-40d7-90e5-ea5ba372d8de)

- 세 번째는 전부 차단하는 방식입니다. AWS에서 권장하는 방식으로 S3내의 객체에 대해서 직접적으로 public 접근할 수 없게 만드는 방법으로 pre signed url 방식을 이용할 때 설정해 볼 수 있는 방법입니다. 이 떄는 put, get 모두 요청마다 서명된 url을 만들고 그 url에 업로드 또는 다운로드하는 방식으로 진행하게 됩니다.
![전부 차단](https://github.com/user-attachments/assets/95be4d1e-0552-48f2-9041-5c729eaff1e9)

<br>

이미지를 저장하는데 S3를 사용할 경우 액세스 차단 설정에 관하여 알아보았는데, 이 설정에 추가적으로 객체 소유권에 대한 설정만 이해하고 사용한다면 버킷을 사용하는데 있어서 큰 문제없이 사용할 수 있습니다. 중간에 나온 액세스 포인트라는 개념은 AWS 공식문서에 더 자세하게 나와있으므로 해당 내용에 대해 학습하고 싶다면 아래 링크에서 추가적으로 학습을 진행하시면 됩니다. 간단하게 설명하면 접근제어를 버킷 정책과 같이 사용하면 좀 더 세밀하게 다룰 수 있는 방법입니다.

[AWS 문서](https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/access-points-vpc.html?icmpid=docs_amazons3_console)

<br>

## 4. 객체 소유권 설정에 대한 이해

<br>

![객체 소유권](https://github.com/user-attachments/assets/c10bc428-fa2e-4eab-a11e-623920bfb123)

버킷 소유권 옵션에는 3가지가 존재합니다.

1. ACL 비활성화 (버킷 소유권 : 버킷 소유자 적용)
2. ACL 활성화 (버킷 소유권 : 버킷 소유자 우선)
3. ACL 활성화 (버킷 소유권 : 객체 작성자 소우, ACL 활성화)

S3 객체 소유권은 버킷에 업로드되는 객체의 소유권을 제어하고 액세스 제어 목록 (ACL)을 활성화하거나 비활성화하는데 사용할 수 있는 S3 버킷 수준 설정입니다. 버킷의 경우 객체 소유권은 `버킷 소유자 적용` 설정으로 자동 설정되고 모든 ACL이 사용 중지 된다. ACL이 비활성화되면 버킷 소유자는 버킷의 모든 객체를 소유하고 `정책`을 통해 해당 객체에 대한 액세스를 관리합니다.

여기서 `핵심`은 ACL을 비활성화했을 경우 버킷의 퍼블릭 액세스 차단 설정으로 객체에 ACL로 권한을 부여할 수 있다고 설정하더라도 해당 기능을 사용할 수 없습니다. 즉 ACL이 적용되어있더라도 ACL이 무시되므로 ACL의 권한은 적용되지 않습니다.

AWS 권장사항은 ACL 사용중지인데 나와있는 설명에 따르면 각 객체에 대한 개별 액세스를 제어해야하는 상황을 제외하고는 ACL이 필요없다고 보는 입장입니다.

만약 ACL을 활성화한다면 객체에 ACL을 부여할 수 있게 되며 서로다른 AWS 계정에서 해당 버켓에 접근할수 있게 됩니다. 
2번과 3번 선택지의 차이는 bucket-owner-full-control라는 ACL을 객체를 저장할 때 추가하는지의 여부로 객체에 대한 소유권을 버킷 소유자에게 둘지 혹은 객체 작성자에게 둘지를 결정할 수 있습니다. 다른 계정으로 버킷에 접근하는 것은 생각하지 않고 ACL을 부여할 수 있는지 없는지에 초점을 맞춰서 사용하려고 합니다.

위에서 설명했던 사용예시 3가지 방식에 각각 객체 소유권 설정을 맞춰본다면

`1번은` 객체에 ACL로 퍼블릭 액세스를 허용하는걸 막지 않고 정책은 막겠다는 옵션으로 해당 옵션을 사용할 경우 `객체 소유권은 ACL 활성화 됨`을 선택해서 사용해야합니다.

`2번은` 정책으로 접근제어를 진행하는 방식으로 객체에 대한 ACL로 public 접근 제한을 설정하지 않기에 `객체 소유권은 ACL 비활성화 됨`을 선택해서 사용할 수 있습니다.

`3번은` 모두 막는 방식으로 `객체 소유권은 ACL 비활성화 됨`을 선택해서 사용할 수 있습니다.

<br>

## 5. 정리

<br>

버킷을 생성할 때 항상 선택의 근거에 고민이었던 `객체 소유권 옵션`과 `버킷의 퍼블릭 액세스 차단 설정`에 대해서 자세하게 뜯어가며 학습해서 최종적으로 버킷을 어떤 용도로 사용할지에 따라 어떤 소유권 옵션을주고 어떤 차단설정을 줘서 사용할지 선택할 수 있는 근거를 만들어 보았습니다. 

다른 옵션들(버킷 버전 관리, 기본 암호화, 고급 설정)은 충분히 설명을 읽고 판단할 수 있다고 생각해서 생략하였습니다.

추가적으로 버킷을 애플리케이션에서 접근해서 사용하려면 IAM을 따로 생성해서 키를 발급받아 접근해서 사용해야합니다. 이 때,
이 IAM 유저는 기본적으로 S3에 접근할 수 있으며 이 유저들에 대해서 따로 정책이나 ACL로 막지 않는다면
모든 기능을 사용할 수있습니다.





