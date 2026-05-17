---
layout: post
title: "SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초"
date: 2026-05-17 01:10:00 +0900
category: SpringBoot
permalink: /springboot/mvc-service-layer-di
---

# SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초

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

