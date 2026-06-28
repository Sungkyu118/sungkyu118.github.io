---
layout: post
title: "SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초"
date: 2026-05-17 01:10:00 +0900
category: SpringBoot
permalink: /springboot/mvc-service-layer-di
description: "Spring Boot에서 Controller와 Service를 분리하는 이유, 의존성 주입이 왜 필요한지, 입문자가 계층 구조를 잡는 방법을 쉽게 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초

> 이 글에서는 Service 계층을 분리해 Controller가 너무 많은 책임을 갖지 않게 만드는 방법을 설명합니다.
>
> 이전 글: [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
> 다음 글: [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
> - [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)


초보자가 컨트롤러에 로직을 다 넣기 시작하면, 프로젝트가 조금만 커져도 바로 유지보수가 어려워집니다.

이 글은 다음을 목표로 합니다.

- 컨트롤러는 "HTTP 처리"만 하고
- 비즈니스 로직은 Service로 분리하는 습관을 만든다
- DI(의존성 주입)가 왜 필요한지 감을 잡는다

## 1) Controller와 Service 역할 분리

권장:

- Controller: 요청/응답, 파라미터 변환, 상태코드 결정
- Service: 비즈니스 로직(규칙), 트랜잭션, 외부 의존 호출

## 2) 생성자 주입(가장 추천)

```java
import org.springframework.stereotype.Service;

@Service
public class HelloService {
  public String hello(String name) {
    return "hello " + name;
  }
}
```

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController2 {
  private final HelloService helloService;

  public HelloController2(HelloService helloService) {
    this.helloService = helloService;
  }

  @GetMapping("/hello2")
  public String hello(@RequestParam String name) {
    return helloService.hello(name);
  }
}
```

생성자 주입의 장점:

- 테스트가 쉽다(가짜 구현을 넣기 쉬움)
- 필수 의존성이 명확하다

## 3) 초보자들이 자주 묻는 질문

### Q. 왜 new로 만들면 안 되나요?

`new HelloService()`로 직접 만들면, Spring이 관리하는 기능(프록시, 트랜잭션, 설정 주입 등)을 놓칠 수 있습니다. 가장 중요한 건 "테스트/확장"이 어려워진다는 점입니다.

## 정리

- 로직은 Service로 분리하는 습관이 프로젝트를 살린다
- 의존성은 생성자 주입이 가장 깔끔하다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Controller에 모든 로직을 몰아넣기 시작하면 초반에는 빨라 보여도 금방 읽기 어렵고 테스트하기도 어려워집니다. Service 계층은 "요청을 받는 부분"과 "비즈니스 규칙을 수행하는 부분"을 나눠서 코드를 더 이해하기 쉽게 만드는 역할을 합니다. 그리고 DI는 이 계층들을 느슨하게 연결해주는 핵심 원리입니다.

예를 들어 회원 생성 요청을 받았을 때, Controller는 입력을 받고 응답을 만드는 데 집중하고, 실제 저장 규칙이나 중복 검사 같은 비즈니스 처리는 Service로 넘기는 식으로 역할을 나누면 이후 기능이 커져도 구조가 덜 흔들립니다. 이 감각은 프로젝트가 커질수록 정말 중요해집니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 생성자 주입으로 의존성을 연결하고, Controller는 얇게 유지하면서 Service가 핵심 규칙을 담당하도록 나누는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.service;

import org.springframework.stereotype.Service;

@Service
public class UserService {

  public String createUser(String name) {
    return "created:" + name;
  }
}
```
```java
package com.example.demo.web;

import com.example.demo.service.UserService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserController {
  private final UserService userService;

  public UserController(UserService userService) {
    this.userService = userService;
  }

  @PostMapping
  public String create(@RequestParam String name) {
    return userService.createUser(name);
  }
}
```

위 예시는 Service 계층과 DI을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 필드 주입으로 빠르게 시작하면 테스트 코드에서 객체를 직접 만들기 어렵고, 나중에 리팩터링할 때 의존관계가 잘 보이지 않습니다.
- Service가 단순 전달만 하고 실제 규칙이 다시 Controller로 흘러나오면 계층을 나눈 의미가 약해집니다.
- 두 Service가 서로를 주입하는 구조가 생기면 순환 참조 문제가 생길 수 있으니 책임 분리를 다시 점검하셔야 합니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 생성자 주입을 기본값으로 삼으시면 테스트성과 가독성이 함께 좋아집니다.
- Controller는 입력과 출력, Service는 규칙과 흐름이라는 구도를 계속 의식하시면 좋습니다.
- 처음에는 메서드 하나만 분리해도 충분하니, 억지로 모든 클래스를 세분화하려고 하실 필요는 없습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
