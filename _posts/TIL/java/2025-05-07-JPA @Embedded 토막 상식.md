---
layout: post
category: java
---

JPA에서 `@Embedded` 어노테이션은 엔티티 클래스 내에 다른 값 타입을 포함시키는 데 사용됩니다. 이는 객체지향적 설계를 데이터베이스 모델링에 적용할 수 있게 해주는 중요한 기능입니다.

## @Embedded 어노테이션 개념

`@Embedded`는 값 타입을 엔티티에 내장하는 것을 의미합니다. 관련된 필드들을 하나의 클래스로 그룹화하여 재사용하고 코드의 가독성과 유지보수성을 높일 수 있습니다.

## 사용 방법

- 먼저 임베디드 타입을 정의합니다.

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;

    // 게터, 세터, 생성자 등
}

```

- 엔티티 클래스에서 임베디드 타입을 사용합니다.

```java
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @Embedded
    private Address address;

    // 게터, 세터, 생성자 등
}

```

## 데이터베이스 매핑

임베디드 타입을 사용하면 데이터베이스 테이블에서는 해당 필드들이 엔티티 테이블에 직접 포함됩니다. 위 예제에서 USER 테이블은 다음과 같은 컬럼을 갖게 됩니다.

- ID
- NAME
- STREET
- CITY
- ZIPCODE

## 컬럼명 커스터마이징

`@AttributeOverride`를 사용하여 임베디드 타입의 컬럼명을 재정의할 수 있습니다.

```java
@Entity
public class Company {
    @Id
    @GeneratedValue
    private Long id;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "company_street")),
        @AttributeOverride(name = "city", column = @Column(name = "company_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "company_zipcode"))
    })
    private Address address;
}

```

## 여러 임베디드 타입 사용

동일한 임베디드 타입을 여러 번 사용할 수도 있습니다

```java
@Entity
public class Person {
    @Id
    @GeneratedValue
    private Long id;

    @Embedded
    private Address homeAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "work_street")),
        @AttributeOverride(name = "city", column = @Column(name = "work_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "work_zipcode"))
    })
    private Address workAddress;
}
```