---
layout: post
category: java
---

스프링에서 **`@Transactional`**을 사용하여 중첩 트랜잭션을 구현하려면 **Propagation.NESTED** 속성을 사용하고, **DataSourceTransactionManager**를 명시적으로 설정해야 한다. 이는 JPA의 기본 트랜잭션 매니저인 **`JpaTransactionManager`**가 NESTED 전파를 지원하지 않기 때문이다.

```java
@Configuration
@EnableTransactionManagement
public class DbConfig {
    @Bean
    public PlatformTransactionManager datasourceTransactionManager(DataSource dataSource) {
        DataSourceTransactionManager manager = new DataSourceTransactionManager();
        manager.setDataSource(dataSource);
        manager.setNestedTransactionAllowed(true);  // 필수 설정
        return manager;
    }
}

```

이렇게 설정 후, 외부 트랜잭션과 내부 트랜잭션을 분리하고 예외 발생 시 부분 롤백을 구현

```java
// 외부 서비스 (부모 트랜잭션)
@Service
public class MemberAppService {
    @Transactional(transactionManager = "datasourceTransactionManager")
    public void createMember() {
        List<String> names = List.of("최길동", "홍길동", "고길동");
        List<String> duplicateNames = new ArrayList<>();
        
        for (String name : names) {
            try {
                memberService.createMemberByName(name);  // NESTED 트랜잭션 호출
            } catch (Exception e) {
                duplicateNames.add(name);
            }
        }
    }
}

// 내부 서비스 (자식 트랜잭션)
@Service
public class MemberService {
    @Transactional(
        transactionManager = "datasourceTransactionManager",
        propagation = Propagation.NESTED  // NESTED 전파 설정
    )
    public void createMemberByName(String name) {
        jdbcTemplate.update("INSERT INTO Member (name) VALUES (?)", name);
        if (name.equals("홍길동")) throw new RuntimeException("의도된 예외");
    }
}

```

동작방식은 다음과 같다

- 부모 트랜잭션이 시작되면 데이터베이스에 **세이브 포인트**가 생성
- 자식 트랜잭션에서 예외가 발생하면 해당 세이브 포인트까지 롤백, 부모 트랜잭션은 나머지 작업을 진행

만약 위의 코드에서 `홍길동` 삽입 시에 예외가 발생한다면, `홍길동` 에 대한 INSERT 작업은 롤백되고, 다른 작업은 정상적으로 커밋된다.

> **유의할 점은 자식 트랜잭션에서 예외를 터뜨려야 전체 트랜잭션이 롤백되지 않는다.**
> 

## `@Transactional`에서 `REQUIRES_NEW`를 사용하면 새로운 DB 세션을 사용할까?

```java
// 부모 트랜잭션
@Transactional
public void parentMethod() {
    parentRepository.save(entityA);  // 커넥션 1 사용
    childService.childMethod();      // 새로운 커넥션 2 사용
}

// 자식 트랜잭션 (REQUIRES_NEW)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void childMethod() {
    childRepository.save(entityB);   // 커넥션 2 사용
}

```

`Propagation.REQUIRES_NEW` 는 기존 트랜잭션이 존재하면 해당 트랜잭션을 보류시키고, 새로운 커넥션을 데이터소스 풀에서 할당받아 사용

## `Propagation.REQUIRES_NEW`**와 **`Propagation.NESTED`의 차이점

![](https://velog.velcdn.com/images/leehjhjhj/post/018ed3c1-182b-4554-ac30-2d860d41f417/image.png)

### Trade Off

- 커넥션 사용량
    - **`REQUIRES_NEW`**는 트랜잭션마다 커넥션을 할당하므로 동시 요청이 많을 경우 **풀 고갈 위험**
- Lock 경합
    - **`NESTED`**는 동일 커넥션을 공유하므로 **행 단위 Lock 유지 시간 증가** 가능성

### 각각 어디에 써야할까?

- REQUIRES_NEW
    - 감사 로그 기록: 주문 처리 실패 시 로그는 반드시 저장해야 할 때
    - 외부 API 호출: 결제 서비스와의 통신
- NESTED
    - 배치 처리: 100개 데이터 처리 중 일부 실패 시 부분 재시도