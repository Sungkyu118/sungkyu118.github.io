---
layout: post
title: "SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)"
date: 2026-05-17 02:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-actuator-health-metrics
description: "Spring Boot Actuator로 헬스체크와 운영 지표를 여는 방법, 어떤 엔드포인트를 공개해야 하는지, 운영에서 주의할 점을 입문자 기준으로 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)

> 이 글에서는 Actuator로 서비스 상태와 운영 지표를 열면서도 보안상 어디까지 노출할지 판단하는 기준을 익힙니다.
>
> 이전 글: [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)
> 다음 글: [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
> - [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)


운영을 하려면 "서비스가 살아있는지"를 기계가 확인할 수 있어야 합니다. Spring Boot Actuator는 그걸 쉽게 만들어줍니다.

목표:

- `/actuator/health`로 헬스체크 제공
- 필요한 엔드포인트만 안전하게 노출

## 1) 의존성 추가

`build.gradle`:

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

## 2) 엔드포인트 노출 설정(중요)

기본적으로 Actuator는 보안상 제한되어 있습니다. 꼭 필요한 것만 노출하는 게 안전합니다.

`application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
```

관련 설명은 Spring Boot 문서에서도 "노출 전 보안 고려"를 강조합니다. citeturn0search0

## 3) 확인

- `GET /actuator/health`

## 4) 운영 팁

- health는 로드밸런서/쿠버네티스에서 자주 사용됩니다.
- 민감한 엔드포인트를 공개하면 정보 유출이 될 수 있습니다.

## 정리

- Actuator는 운영 준비의 기본
- 노출은 최소한으로, 보안이 먼저다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

애플리케이션이 잘 켜지는 것과, 운영에서 건강한 상태인지 확인하는 것은 다른 문제입니다. Actuator는 현재 서버가 살아 있는지, 디스크는 괜찮은지, 어떤 메트릭이 쌓이고 있는지 같은 운영 관점 정보를 확인할 수 있게 해주는 중요한 도구입니다.

초반에는 `/actuator/health` 하나만 열어도 배포 자동화나 모니터링 연동이 훨씬 쉬워집니다. 다만 편하다고 해서 모든 엔드포인트를 운영에 그대로 노출하면 위험할 수 있기 때문에, 무엇을 공개하고 무엇을 막을지 같이 생각하시는 습관이 필요합니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 Actuator 의존성 추가, 노출 엔드포인트 설정, 운영 환경에서의 보안 고려 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: when_authorized
```

위 예시는 Actuator을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 테스트 용도로 `*` 전체 노출을 켜둔 뒤 운영까지 그대로 가는 실수가 생각보다 자주 발생합니다.
- 관리 포트와 애플리케이션 포트를 혼동하면 헬스체크 URL을 잘못 붙이는 경우가 있습니다.
- 헬스 상세 정보를 누구나 볼 수 있게 열어두면 내부 인프라 정보가 과하게 노출될 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 `health`, `info` 정도만 최소로 열고, 필요한 메트릭만 점진적으로 추가하시는 편이 좋습니다.
- 운영 환경에서는 보안 설정과 함께 Actuator 엔드포인트 접근 정책을 반드시 점검하셔야 합니다.
- 배포 자동화와 연결될 URL이 무엇인지 직접 호출해서 한 번 검증해보시는 습관이 중요합니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles), [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback), [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->


## 추가로 연습해보시면 좋습니다

Actuator 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
- [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)
- [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
