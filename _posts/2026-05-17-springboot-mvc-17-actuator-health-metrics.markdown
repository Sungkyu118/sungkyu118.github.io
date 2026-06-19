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

