---
layout: post
title: "SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해"
date: 2026-05-17 01:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-transaction-basics
description: "Spring Boot의 @Transactional이 언제 필요한지, 롤백과 프록시 때문에 초보자가 자주 헷갈리는 지점을 중심으로 트랜잭션 기본을 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해

> 이 글에서는 트랜잭션이 실제로 어떤 범위를 묶고 언제 롤백되는지 입문자 기준으로 이해합니다.
>
> 이전 글: [SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)](/springboot/mvc-pagination-sorting)
> 다음 글: [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)
> - [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)


초보자가 DB 작업을 하기 시작하면 바로 부딪히는 개념이 트랜잭션입니다.

이 글의 목표:

- `@Transactional`을 어디에 붙여야 하는지 안다
- readOnly를 언제 쓰는지 이해한다
- 흔한 오해(컨트롤러에 붙이기, self-invocation) 를 피한다

## 1) 트랜잭션은 "한 묶음"의 DB 작업

예: 회원 가입을 하면서

- user 테이블 insert
- 포인트 테이블 insert

이 두 작업이 하나의 트랜잭션이면, 중간에 실패 시 둘 다 롤백됩니다.

## 2) 어디에 붙이나: Service 계층

권장:

- Controller: 트랜잭션 X
- Service: 트랜잭션 O

```java
@Service
public class SignupService {
  @Transactional
  public void signup(...) { ... }
}
```

## 3) readOnly 트랜잭션

조회만 하는 로직에는:

```java
@Transactional(readOnly = true)
public UserResponse get(Long id) { ... }
```

초보자 관점에서의 이점:

- 의도를 명확히 함(쓰기 없음)

## 4) 흔한 오해/실수

### (1) 컨트롤러에 붙이기

컨트롤러에 붙이면 코드가 섞이고, 테스트/확장 시 복잡해집니다. Service에 붙이세요.

### (2) 같은 클래스 내부 호출(self-invocation)

Spring의 트랜잭션은 보통 프록시 기반이라, 같은 클래스 내부에서 `this.method()`로 호출하면 트랜잭션이 적용되지 않는 케이스가 있습니다. 구조를 나누거나 호출 흐름을 점검해야 합니다.

## 정리

- 트랜잭션은 Service 계층에
- 조회는 readOnly로 의도를 표현
- 프록시 기반 동작을 이해하면 디버깅이 쉬워진다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

트랜잭션은 데이터가 여러 단계로 바뀌는 작업을 하나의 묶음으로 다루기 위한 핵심 개념입니다. 돈을 이체하거나, 주문을 저장하면서 재고를 줄이고, 로그를 남기고, 상태를 변경하는 식의 작업에서 중간에 일부만 반영되면 큰 문제가 생길 수 있습니다. 이때 트랜잭션을 제대로 이해해두셔야 데이터 정합성을 지킬 수 있습니다.

많은 입문자분들이 `@Transactional`을 붙이면 모든 것이 자동으로 안전해진다고 생각하시는데, 실제로는 어디에 붙였는지, 어떤 예외가 발생했는지, 같은 클래스 안에서 호출했는지에 따라 동작이 달라질 수 있습니다. 그래서 문법보다 경계(boundary) 감각을 먼저 잡는 것이 중요합니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 트랜잭션 경계를 Service 메서드에 두고, 여러 DB 작업이 하나로 묶이는 흐름과 롤백 조건을 이해하는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.account;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class TransferService {
  private final AccountRepository accountRepository;

  public TransferService(AccountRepository accountRepository) {
    this.accountRepository = accountRepository;
  }

  @Transactional
  public void transfer(Long fromId, Long toId, int amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.withdraw(amount);
    to.deposit(amount);

    if (amount > 1000000) {
      throw new IllegalArgumentException("이체 한도를 초과했습니다.");
    }
  }
}
```

위 예시는 트랜잭션을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- `@Transactional`을 private 메서드에 붙이면 프록시가 적용되지 않아 기대한 롤백이 일어나지 않을 수 있습니다.
- 예외를 try-catch로 잡고 아무 처리 없이 삼켜버리면 트랜잭션이 커밋되어 일부 데이터만 반영될 수 있습니다.
- 같은 클래스 내부에서 메서드를 자기 자신이 호출하는 구조에서는 프록시를 우회해 트랜잭션이 적용되지 않는 경우가 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 트랜잭션은 보통 하나의 유스케이스를 담당하는 Service 메서드에 두시는 것이 가장 이해하기 쉽습니다.
- DB 작업과 무관한 외부 API 호출이나 무거운 파일 작업은 트랜잭션 밖으로 빼는 설계를 고민해보시는 것이 좋습니다.
- 롤백 규칙은 직접 예외를 발생시켜보면서 몸으로 확인해보셔야 오래 기억에 남습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
