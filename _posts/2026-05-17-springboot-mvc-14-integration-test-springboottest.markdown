---
layout: post
title: "SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기"
date: 2026-05-17 02:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-integration-test-springboottest
description: "Spring Boot에서 @SpringBootTest로 전체 흐름을 검증하는 통합 테스트 기본, 단위 테스트와의 차이, 느려지는 이유와 사용 기준을 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기

> 이 글에서는 애플리케이션을 실제로 띄우는 통합 테스트가 언제 필요한지와 비용을 함께 이해합니다.
>
> 이전 글: [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)
> 다음 글: [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
> - [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)


단위 테스트/컨트롤러 테스트가 "부분"을 검증한다면, 통합 테스트는 "전체가 연결되는지" 확인합니다.

초보자에게 추천하는 최소 통합 테스트 목표:

- 앱 컨텍스트가 뜨는지
- 주요 엔드포인트가 200을 내는지
- DB가 붙는 흐름이 동작하는지

## 1) 가장 기본적인 컨텍스트 테스트

```java
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class ContextLoadsTest {
  @Test
  void contextLoads() {}
}
```

이게 깨지면 "설정/빈 구성" 문제가 있다는 뜻입니다.

## 2) TestRestTemplate로 실제 HTTP 호출(선택)

초보자 단계에서는 MockMvc가 더 흔하지만, 실제 포트로 띄우고 호출하는 테스트도 가능합니다.

## 3) 통합 테스트가 느려지는 이유

- Spring 컨텍스트를 띄우는 비용
- DB/외부 의존 연결

그래서 테스트 전략은 보통:

- 단위 테스트 많이(빠름)
- 통합 테스트는 핵심 흐름만(느림)

## 정리

- `@SpringBootTest`는 전체 연결 확인용
- 느리기 때문에 핵심 케이스만 넣는 게 실전 전략이다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

통합 테스트는 Controller, Service, Repository, 설정이 실제로 함께 잘 동작하는지 확인할 때 필요합니다. 단위 테스트만으로는 각 조각이 개별적으로 맞는지만 볼 수 있기 때문에, 빈 연결이나 트랜잭션, DB 저장 흐름 같은 실제 통합 문제는 놓칠 수 있습니다.

예를 들어 회원 생성 API는 Service 테스트에서는 통과했는데, 실제 실행해보니 설정 프로필이 달라 DB 연결이 안 되거나, Validation 메시지 구조가 기대와 다르게 나오는 경우가 있습니다. 이런 문제를 미리 잡아주는 것이 통합 테스트의 역할입니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 실제 Spring 컨텍스트를 띄우고, 여러 계층이 같이 연결된 상태에서 핵심 흐름을 검증하는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
@SpringBootTest
@Transactional
class UserIntegrationTest {

  @Autowired
  private UserService userService;

  @Autowired
  private UserRepository userRepository;

  @Test
  void createAndRead_shouldWorkEndToEnd() {
    UserResponse created = userService.create("sungkyu");
    UserEntity found = userRepository.findById(created.id()).orElseThrow();

    assertThat(found.getName()).isEqualTo("sungkyu");
  }
}
```

위 예시는 통합 테스트을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 통합 테스트는 무겁기 때문에 모든 시나리오를 여기에 몰아넣으면 실행 시간이 급격히 늘어날 수 있습니다.
- 테스트 데이터 정리를 하지 않으면 테스트 순서에 따라 결과가 바뀌는 flaky 테스트가 되기 쉽습니다.
- 단위 테스트처럼 Mock으로 모두 대체해버리면 통합 테스트의 의미가 약해집니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 정말 중요한 대표 흐름 몇 개만 통합 테스트로 두고, 세부 규칙은 단위 테스트로 분산하시는 편이 효율적입니다.
- 테스트 전용 프로필과 테스트용 DB를 분리해두시면 로컬 데이터와 섞이는 일을 줄일 수 있습니다.
- 문제가 생겼을 때는 어떤 계층 연결에서 깨졌는지 추적하는 연습까지 함께 해보시면 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito), [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db), [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->


## 추가로 연습해보시면 좋습니다

통합 테스트 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
- [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)
- [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
