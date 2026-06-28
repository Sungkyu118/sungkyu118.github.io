---
layout: post
title: "SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-testcontainers-mysql
description: "Spring Boot에서 Testcontainers로 MySQL 통합 테스트 환경을 만드는 방법, 로컬 DB 의존도를 줄이는 이유와 기본 구성을 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트

> 이 글에서는 실제 MySQL과 가까운 테스트 환경을 Testcontainers로 만드는 이유와 기본 사용법을 다룹니다.
>
> 이전 글: [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)
> 다음 글: [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics)
> - [SpringBoot 입문 29: QueryDSL로 동적 조회 만들기](/springboot/mvc-querydsl-intro)


로컬 DB 의존성을 줄이고 신뢰도 높은 통합 테스트를 구성해보자.

## 핵심 포인트
- 테스트 실행 시 컨테이너 DB 자동 기동
- 운영과 유사한 DB 환경에서 검증 가능
- CI 환경에서도 재현성이 높음

## 정리
Testcontainers를 적용하면 환경 차이로 인한 테스트 실패를 크게 줄일 수 있다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

통합 테스트에서 실제 MySQL과 비슷한 환경을 재현하고 싶을 때 Testcontainers는 아주 강력한 선택지입니다. 로컬에 DB를 따로 깔아두지 않아도 되고, 테스트가 시작될 때 필요한 컨테이너를 띄워 실제와 유사한 환경을 만들 수 있어서 신뢰도 높은 테스트를 구성하기 좋습니다.

입문자분들은 보통 H2로 충분하지 않나 생각하실 수 있는데, 실제 MySQL과 동작 차이가 있는 SQL, 인덱스, 타입 매핑 문제가 존재할 수 있습니다. 그래서 중요한 저장소 로직이나 쿼리는 실제 DB 계열과 가까운 환경에서 한 번쯤 검증해보는 것이 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 테스트 코드 안에서 MySQL 컨테이너를 띄우고, 동적으로 datasource 속성을 연결하는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
@Testcontainers
@SpringBootTest
class UserRepositoryContainerTest {

  @Container
  static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
      .withDatabaseName("demo")
      .withUsername("test")
      .withPassword("test");

  @DynamicPropertySource
  static void overrideProps(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", mysql::getJdbcUrl);
    registry.add("spring.datasource.username", mysql::getUsername);
    registry.add("spring.datasource.password", mysql::getPassword);
  }
}
```

위 예시는 Testcontainers을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- Docker 데몬이 꺼져 있으면 Testcontainers 테스트는 시작조차 하지 못할 수 있습니다.
- 컨테이너 필드를 static으로 두지 않거나 라이프사이클을 잘못 이해하면 테스트마다 불필요하게 많이 뜰 수 있습니다.
- 단위 테스트까지 모두 Testcontainers로 돌리면 전체 테스트 속도가 크게 느려질 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 정말 실제 DB와의 차이가 중요한 테스트에만 선별적으로 적용하시는 것이 좋습니다.
- 로컬 개발 환경에서 Docker가 정상인지 먼저 확인해두면 시행착오를 많이 줄일 수 있습니다.
- H2와 Testcontainers를 경쟁 관계로 보기보다, 목적이 다른 두 도구로 구분해서 사용하시면 훨씬 편합니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

## 추가로 연습해보시면 좋습니다

Testcontainers 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

## 추가로 연습해보시면 좋습니다

Testcontainers 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.
