---
layout: post
title: "SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드"
date: 2026-05-17 01:00:00 +0900
category: SpringBoot
permalink: /springboot/mvc-rest-basics-params-status
description: "Spring Boot에서 @RequestParam과 @PathVariable을 언제 쓰는지, HTTP 상태코드를 어떻게 내려야 하는지 REST API 기본 흐름을 예제로 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드

> 이 글에서는 REST API에서 query parameter와 path variable을 구분하고 상태코드를 설계하는 기본을 다룹니다.
>
> 이전 글: [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
> 다음 글: [SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초](/springboot/mvc-service-layer-di)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
> - [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)


이 글은 API를 만들 때 가장 자주 쓰는 2가지 입력 방식과, 응답 상태코드를 정리합니다.

- Query parameter: `@RequestParam`
- Path parameter: `@PathVariable`
- 상태코드: 200/201/204/400/404/500

## 1) @RequestParam: 쿼리스트링 받기

예: `/search?q=spring`

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SearchController {

  @GetMapping("/search")
  public String search(@RequestParam String q) {
    return "q=" + q;
  }
}
```

선택값이면:

```java
public String search(@RequestParam(required = false) String q) { ... }
```

## 2) @PathVariable: URL 경로에서 받기

예: `/users/42`

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class UserController2 {

  @GetMapping("/users/{id}")
  public String getUser(@PathVariable String id) {
    return "id=" + id;
  }
}
```

## 3) 상태코드를 명확하게 내려주기(ResponseEntity)

초보자는 일단 `ResponseEntity`만 익혀도 충분합니다.

```java
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class CreateController {

  @PostMapping("/items")
  public ResponseEntity<String> create() {
    return ResponseEntity.status(201).body("created");
  }
}
```

실무에서 자주 쓰는 패턴:

- 생성: 201 Created
- 삭제: 204 No Content
- 잘못된 요청: 400
- 없는 리소스: 404

## 4) 초보자 실수 모음

- `@RequestParam`인데 URL에 `?q=`를 안 붙임
- `@PathVariable`인데 `{}` 경로를 안 맞춤
- 상태코드를 200으로만 내려서 클라이언트가 케이스 분기하기 어려움

## 정리

- 입력은 `@RequestParam`/`@PathVariable`로 시작
- 응답은 `ResponseEntity`로 상태코드까지 명확히

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

REST API를 작성하실 때는 단순히 데이터를 돌려주는 것보다, 어떤 경로와 파라미터를 받고 어떤 상태코드로 응답할지를 분명하게 설계하는 것이 중요합니다. 그래야 프론트엔드나 다른 클라이언트가 API를 예측 가능하게 사용할 수 있고, 디버깅도 훨씬 쉬워집니다.

처음에는 `@RequestParam`과 `@PathVariable`의 차이가 아주 작아 보여도, 실제로는 URL 설계 철학과 연결됩니다. "리소스 자체를 식별하는 값인지", "조회 조건인지"를 구분하면서 설계해보시면 API가 훨씬 읽기 좋아지고 유지보수도 쉬워집니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 경로 변수와 쿼리 파라미터의 역할 차이, 그리고 성공/실패에 맞는 HTTP 상태코드를 분명하게 내려주는 습관 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.web;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserRestController {

  @GetMapping("/{id}")
  public ResponseEntity<String> getUser(@PathVariable Long id,
                                        @RequestParam(defaultValue = "false") boolean detail) {
    return ResponseEntity.ok("id=" + id + ", detail=" + detail);
  }

  @PostMapping
  public ResponseEntity<String> createUser(@RequestParam String name) {
    return ResponseEntity.status(201).body("created user=" + name);
  }
}
```

위 예시는 REST 파라미터와 상태코드을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- `@PathVariable` 이름과 URL 템플릿 이름이 맞지 않으면 바인딩 오류가 나거나 실행 시점에 예외가 발생할 수 있습니다.
- 리소스 생성인데도 항상 200 OK만 내려주면 클라이언트 입장에서 동작 구분이 어려워집니다.
- 숫자 타입 파라미터에 문자열이 들어오면 400 Bad Request가 발생하는데, 이를 서버 오류로 오해하시는 경우가 많습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 리소스 식별자는 PathVariable, 정렬/필터 조건은 RequestParam이라는 큰 기준으로 먼저 잡아보시면 좋습니다.
- 단순 문자열을 바로 리턴하는 예제에서 시작하더라도, 상태코드는 꼭 의식적으로 설정해보시는 습관을 들이시는 것이 좋습니다.
- Postman이나 curl로 200, 201, 400을 일부러 만들어보시면 HTTP 상태코드가 훨씬 몸에 익습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
