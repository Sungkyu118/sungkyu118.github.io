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

