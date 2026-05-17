---
layout: post
title: "SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해"
date: 2026-05-17 01:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-transaction-basics
---

# SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해

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

