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

