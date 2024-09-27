---
title: "Spring 트랜잭션 rollBackOnly"
date: 2024-09-24 22:00:00 +09:00
categories: [스프링,JPA]
tags:
  [
    스프링,
    JPA,
    Transactional,
    트랜잭션,
    rollBackOnly
  ]
---

<br>

> Spring data JPA의 save() 메서드가 실패할 때 우리는 어떻게 할까? 저장을 해야하는데 필수 조건이 맞지 않아 실패한다면 우리는 던저진 에러를 잡아 처리한다. 만약, save() 메서드가 실패했을 경우 임시 데이터로 다시 저장하는 로직을 개발한다면 어떻게 처리하면 될까? 이에 대해 학습해보았다.

<br>

## 1. 테스트 코드로 순수하게 접근하기

save()메서드가 실패했을 때 예외처리로 해결할 수 있다고 처음에는 생각했다. 따라서 Service 클래스에서 @Transactional을 붙인 메서드 내에서 save()가 실패했을 때 try catch로 예외를 잡아 임시 데이터를 저장하는 코드를 작성하여 테스트 해보았다.

```java
@Entity
@NoArgsConstructor
@Getter
@ToString(of = {"id", "name", "amount"})
public class Fee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    @Column(nullable = false)
    private String name;

    private String amount;

    public Fee(String name, String amount) {
        this.name = name;
        this.amount = amount;
    }

    public Fee(FeeReqDto dto) {
        this.amount = dto.getAmount();
        this.name = dto.getName();
    }
}

@Transactional
public void saveBoard(FeeReqDto dto){
    try {
        feeRepository.save(new Fee(dto));
    }
    catch(Exception e){
        feeRepository.save(new Fee("undefiend",dto.getAmount()));
    }
}


--------------------------------------------------------------------

테스트 코드

@SpringBootTest
public class TransactionTest {

    @Autowired
    FeeService feeService;

    @Test
    void Test1(){
        FeeReqDto feeReqDto = new FeeReqDto(null,"1000");

        feeService.saveBoard(feeReqDto);
    }
}
```

코드는 단순하다. null 옵션이 false인 필드에 null값을 저장하게 해서 에러를 발생시킨 후, 해당 에러를 잡아서 새로운 Fee("undefiend",dto.getAmount()) 데이터로 저장하도록 작성하고 테스트해 보았다.

테스트는 성공할 것이라고 생각했지만 아래와 같은 오류로 실패하였다.
![image](https://github.com/user-attachments/assets/a1eb21dc-0812-4307-adeb-587985d688ad)

먼저 @Transactional의 default TxType은 Required이다. 이 옵션은 전파에 대한 옵션으로 트랜잭션이 없다면 현재 Scope에 물리적 트랜잭션을 설정한다. 만약, 더 큰 Scope에 이미 정의된 트랜잭션이 있다면 이 트랜잭션에 참여한다.

따라서 위 코드에서 우리는 saveBoard()라는 메서드가 실행될 때 물리적 트랜잭션이 설정되고 이후에 트랜잭션들은 별다른 옵션이 없다면 이 트랜잭션에 포함될 것이라는 것을 알 수 있다.

트랜잭션은 트랜잭션이 실패할 경우 roll back을 시키기 위해 내부에 rollBackOnly 옵션이 존재한다. 트랜잭션은 이 rollBackOnly 옵션을 마지막에 확인하고 트랜잭션을 commit할지 roll back할지 결정한다.

이 때, 내부에 @Transactional이 붙어있는 메서드 내부에서 로직을 실행하는 도중 Exception이 터지면 트랜잭션 rollBackOnly 옵션을 true로 설정한다.

이후 외부 트랜잭션은 정상적으로 에러를 잡아 처리하더라도 메서드가 끝난후 rollBackOnly옵션이 true라면 해당 트랜잭션에서 처리한 모든 것들을 roll back한다.


> 그럼 왜 save()메서드에서도 이런 일이 발생할까?
{: .prompt-warning }

https://github.com/spring-projects/spring-data-jpa/blob/main/spring-data-jpa/src/main/java/org/springframework/data/jpa/repository/support/SimpleJpaRepository.java

save() 메서드의 경우도 링크에 들어가서 보면 @Transactional이 붙어있는걸 확인할 수 있다. 따라서 우리가 이 save()메서드를 호출하기 전에 @Transactinal을 붙여주었다면 이 save()의 트랜잭션의 경우 외부 트랜잭션에 포함되게 된다.

<br>

## 2. 전파옵션을 REQUIRES_NEW로 해결하며 겪은 이슈

<br>

@Transactional 어노테이션에는 전파옵션을 지정할 수 있다.
`@Transactional(propagation = Propagation.REQUIRES_NEW)` 이렇게 붙이게 되면 트랜잭션이 하나로 묶이지 않고 따로 관리되어서 rollbackOnly 옵션이 전파되지 않는다.

하지만 나는 이 방법으로 해결하려고 했을 때, 또 한번 문제를 발견할 수 있었다. 아래는 내가 해결하려고 짠 코드이다.

```java
@RequiredArgsConstructor
public class FeeService {
    private final FeeManager feeManager;
    private final FeeRepository feeRepository;

    @Transactional 
    public void saveBoard(FeeReqDto dto) {
        try {
            saveInnerClass(new Fee(dto));
        } catch (Exception e) {
            this.saveInnerClass(new Fee("null",dto.getAmount()));
        }
    }


    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveInnerClass(Fee fee) {
        feeRepository.save(fee);
    }

}

----------------------------------------------------------------------------------------

@Test
void Test1(){
    FeeReqDto feeReqDto = new FeeReqDto(null,"1000");

    feeService.saveBoard(feeReqDto);
}
```

이렇게 작성해서 테스트를 돌렸을 때 옵션으로 REQUIRES_NEW를 주었는데도 불구하고 동일한 트랜잭션에서 작동하여 똑같은 에러가 나오고 있었다.

문제의 원인을 살펴보니 `@Transactional AOP`와 관련이 있었다.
해당 어노테이션을 붙인 메서드가 존재하는 Service를 빈으로 등록할 때 스프링 부트는 실제 우리가 짠 코드로 이루어진 객체를 등록하지않고 트랜잭션과 관련된 코드가 추가된 프록시 객체를 빈으로 등록한다. 따라서 우리가 이 Serivce를 주입받아서 사용할 때는 프록시 객체를 호출하여 해당로직을 실행하게 된다. 

이 때, 호출된 로직을 실행하다가 내부 메서드 호출이 일어나게되면 프록시 객체로 접근하는 것이 아니라 실제 객체로 접근하기 때문에 트랜잭션 옵션이 적용되지 않고 호출되어서 동일한 트랜잭션에 포함이 되고 있었다.

> 프록시 객체에서 내부 메서드 호출이 일어나게 되면 발생하는 문제였다.
{: .prompt-warning }

-> 따라서, 이런 경우를 대비해 save()메서드를 다른 클래스로 분리하여 코드를 작성한다면 최종적으로 아래처럼 작성하였다.

```java
@Service
@RequiredArgsConstructor
public class FeeManager {

    private final FeeRepository feeRepository;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void save(Fee fee){
        feeRepository.save(fee);
    }
}


---------------------------------------------------------------------------------------

@Transactional
    public void saveBoardWithOuterClass(FeeReqDto dto) {
        try{
            feeManager.save(new Fee(dto));
        }
        catch(Exception e){
            feeManager.save(new Fee("null",dto.getAmount()));
        }
    }
```


이렇게 작성하게 되면 feeService 객체가 feeManager의 프록시 객체를 호출하기 때문에 feeManager의 save()메서드에 `@Transactional(propagation = Propagation.REQUIRES_NEW)`가 적용된 메서드를 호출하게되어 정상적으로 원하는 서비스 로직이 발생하게 된다.

## 3. 결론

@Transactional의 기본옵션에 따라서 원하지 않는 결과가 나올 수 있는 것을 확인해 볼 수 있었다. 또, AOP로 작동하는 @Transactional에서 내부 호출이 발생하면 프록시 객체가 아니라 실제 객체로 호출하는 것도 확인해 볼 수 있었다. 

핵심
- Transactional의 전파지연이 default옵션을 설정하였을 때, rollBackOnly로 인해 원하지 않는 상황이 발생할 수 있다.

- 프록시 객체를 사용하고 있을 때 내부 메서드 호출이 일어나면 프록시 객체 메서드를 호출하는 것이 아니라 실제 객체를 호출한다.

