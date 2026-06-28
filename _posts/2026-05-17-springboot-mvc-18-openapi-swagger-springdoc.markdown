---
layout: post
title: "SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)"
date: 2026-05-17 03:00:00 +0900
category: SpringBoot
permalink: /springboot/mvc-openapi-swagger-springdoc
description: "Spring Boot 3.x 프로젝트에 springdoc-openapi를 붙여 Swagger UI를 열고 API 문서를 자동화하는 방법, 운영 환경에서 주의할 점까지 정리합니다."
image:
  path: "/assets/img/og/springboot-swagger-cover.svg"
  alt: "SpringBoot Swagger and OpenAPI article cover"
---


# SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)

> 이 글에서는 Spring Boot API 문서를 자동으로 열어주는 Swagger UI를 붙이고, 운영에서 어디까지 공개할지 판단하는 기준까지 다룹니다.
>
> 이전 글: [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
> 다음 글: [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> - [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

API를 만들다 보면 다음 문제가 바로 생긴다.

- "이 API는 어떤 요청을 받고 어떤 응답을 주지?"

문서를 수동으로 관리하면 금방 코드와 어긋난다. Spring Boot 3.x에서는 `springdoc-openapi`를 많이 사용한다.

## 1) 의존성 추가(Gradle)

Spring MVC(= `spring-boot-starter-web`)라면 아래를 추가한다.

```gradle
dependencies {
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.4.0'
}
```

Spring Boot 3.x에서는 `springdoc-openapi-starter-webmvc-ui`가 일반적인 선택이다.

## 2) 실행 후 접속

기본 URL:

- `http://localhost:8080/swagger-ui.html`
- 또는 `http://localhost:8080/swagger-ui/index.html`

## 3) 실전 팁

- 운영(prod)에서는 Swagger UI를 막는 경우가 많다.
- local/dev에서만 열고, prod에서는 비활성화하는 전략이 안전하다(Profiles로 분리).

## 정리

- 문서를 자동화하면 개발 효율이 올라간다.
- springdoc으로 OpenAPI/Swagger UI를 쉽게 구성할 수 있다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

API 문서는 나중에 한 번에 정리하는 부가 작업처럼 보이지만, 실제 협업에서는 굉장히 큰 차이를 만듭니다. 요청 본문 구조, 필수 필드, 응답 예시를 문서로 같이 보여줄 수 있으면 프론트엔드와의 커뮤니케이션 비용이 크게 줄고, 테스트 도구 역할도 일부 대신할 수 있습니다.

특히 개인 학습 단계에서도 Swagger UI는 "지금 내 API가 정말 이렇게 동작하는가?"를 빠르게 확인하는 좋은 창구가 됩니다. 다만 너무 편하다고 운영 환경에 아무 제한 없이 열어두면 보안상 부담이 생길 수 있으므로, 개발용과 운영용 전략을 나누어 생각하시는 것이 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 springdoc 의존성, Swagger UI 경로, API 설명과 스키마를 코드에 가깝게 유지하는 방식 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```gradle
dependencies {
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.4.0'
}
```
```java
package com.example.demo.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class SwaggerConfig {

  @Bean
  public OpenAPI openAPI() {
    return new OpenAPI().info(
        new Info()
            .title("Demo API")
            .version("v1")
            .description("Spring Boot 실습용 API 문서입니다.")
    );
  }
}
```

위 예시는 Swagger와 OpenAPI을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 버전이 맞지 않는 springdoc 의존성을 넣으면 애플리케이션 시작 자체가 실패하거나 UI가 열리지 않을 수 있습니다.
- 운영 환경에서 Swagger를 그대로 열어두면 내부 API 구조가 과하게 노출될 수 있습니다.
- 실제 API는 바뀌었는데 설명 어노테이션을 갱신하지 않으면 문서와 동작이 어긋나기 시작합니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 문서는 기능이 안정된 뒤 한꺼번에 쓰기보다, 엔드포인트를 만들 때 같이 정리하시는 편이 더 정확합니다.
- 개발/테스트 환경에서만 문서를 열도록 프로필 기반 제어를 고민해보시면 좋습니다.
- 파일 업로드나 인증 헤더처럼 특수한 케이스는 Swagger에서 어떻게 보이는지 직접 확인해보시는 것이 중요합니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status), [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced), [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
- [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
- [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
