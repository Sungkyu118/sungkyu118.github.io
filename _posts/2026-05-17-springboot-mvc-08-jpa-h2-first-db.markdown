---
layout: post
title: "SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)"
date: 2026-05-17 01:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-jpa-h2-first-db
description: "Spring Boot에서 JPA와 H2를 연결해 Entity와 Repository를 처음 만드는 과정, application.yml 설정, 저장과 조회 흐름을 실습 중심으로 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)

> 이 글에서는 Spring Boot 프로젝트에 JPA와 H2를 붙여 첫 번째 Entity와 Repository를 만드는 과정을 따라갑니다.
>
> 이전 글: [SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초](/springboot/mvc-service-layer-di)
> 다음 글: [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
> - [SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해](/springboot/mvc-transaction-basics)

이 글은 초보자가 DB 개발을 시작할 수 있도록, 가장 쉬운 조합으로 진행합니다.

- JPA: 객체(Entity)로 테이블을 다룬다
- H2: 로컬에서 가볍게 쓰는 인메모리 DB(또는 파일 DB)

목표:

- 프로젝트에 JPA를 붙인다
- 엔티티 1개를 만든다
- Repository로 저장/조회한다

## 1) 의존성 추가

`build.gradle`:

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  runtimeOnly 'com.h2database:h2'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

Gradle Sync가 끝나야 import가 됩니다.

## 2) application.yml 설정

`src/main/resources/application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        format_sql: true
  h2:
    console:
      enabled: true
      path: /h2-console

logging:
  level:
    org.hibernate.SQL: DEBUG
```

초보자 설명:

- `ddl-auto: update`는 개발 편의 옵션입니다(운영에서는 보통 다른 전략 사용).
- H2 콘솔은 로컬 개발에서 DB 상태 확인에 유용합니다.

## 3) Entity 만들기

```java
package com.sungkyu.demo;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;

@Entity
public class UserEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;

  protected UserEntity() {}

  public UserEntity(String name) {
    this.name = name;
  }

  public Long getId() { return id; }
  public String getName() { return name; }
}
```

## 4) Repository 만들기

```java
package com.sungkyu.demo;

import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<UserEntity, Long> {
}
```

## 5) 저장/조회 테스트(간단한 서비스)

```java
package com.sungkyu.demo;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserService {
  private final UserRepository userRepository;

  public UserService(UserRepository userRepository) {
    this.userRepository = userRepository;
  }

  @Transactional
  public Long create(String name) {
    UserEntity saved = userRepository.save(new UserEntity(name));
    return saved.getId();
  }
}
```

## 6) H2 콘솔로 확인

서버 실행 후:

- `http://localhost:8080/h2-console`
- JDBC URL: `jdbc:h2:mem:testdb`

## 정리

- JPA 의존성 추가 + datasource 설정
- Entity/Repository로 DB 접근 시작
- 트랜잭션은 Service 계층에서 관리

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

데이터베이스를 처음 붙이실 때는 MySQL이나 PostgreSQL부터 바로 가는 것보다 H2와 함께 JPA의 기본 흐름을 먼저 익히는 편이 훨씬 수월합니다. 메모리 DB라서 준비가 간단하고, Entity와 Repository가 어떤 역할을 하는지 빠르게 확인해볼 수 있기 때문입니다.

특히 초반에는 SQL보다도 "객체를 저장했는데 왜 테이블이 생기지?", "findById는 어디서 온 메서드지?", "엔티티는 왜 기본 생성자가 필요하지?" 같은 질문이 먼저 나옵니다. H2는 이런 개념 질문을 작은 실습으로 빠르게 확인해보기에 아주 좋은 도구입니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 Entity 매핑, Repository 인터페이스, H2 메모리 DB 설정이 함께 돌아가는 첫 persistence 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.user;

import jakarta.persistence.*;

@Entity
public class UserEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;

  protected UserEntity() {
  }

  public UserEntity(String name) {
    this.name = name;
  }

  public Long getId() {
    return id;
  }

  public String getName() {
    return name;
  }
}
```
```java
package com.example.demo.user;

import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<UserEntity, Long> {
}
```

위 예시는 JPA와 H2을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 엔티티에 기본 생성자가 없으면 JPA가 객체를 만들지 못해 런타임 오류가 발생할 수 있습니다.
- H2 메모리 DB는 애플리케이션이 내려가면 데이터가 사라지므로, 재시작 후 데이터가 안 보인다고 해서 저장이 안 된 것으로 오해하실 수 있습니다.
- Entity를 요청 DTO처럼 사용하기 시작하면 나중에 API와 DB 모델이 서로 강하게 묶여 수정이 어려워질 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 테이블 하나, 엔티티 하나, 레포지토리 하나로 가장 단순한 저장 흐름을 먼저 익혀보시는 것이 좋습니다.
- 로그에 찍히는 SQL을 같이 보시면 JPA가 어떤 작업을 대신하고 있는지 이해하는 데 큰 도움이 됩니다.
- 요청 DTO와 Entity를 분리하는 습관은 CRUD가 커질수록 더 중요해지니 초반부터 의식해두시면 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
