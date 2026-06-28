---
layout: post
title: "SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본"
date: 2026-05-17 00:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-application-yml-profiles
description: "Spring Boot의 application.yml 구조, 환경별 profile 분리, 설정값 관리 기본을 예제와 함께 정리해 설정 파일 때문에 자주 막히는 지점을 줄여줍니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본

> 이 글에서는 application.yml을 읽는 법과 profile로 설정을 나누는 기본 감각을 익힙니다.
>
> 이전 글: [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
> 다음 글: [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> - [SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초](/springboot/mvc-service-layer-di)


Spring Boot를 쓰면 설정의 중심은 `application.yml`(또는 properties)입니다. 초보자에게 중요한 건 이 3가지예요.

1. 포트/로그 레벨 같은 기본 설정
2. dev/prod처럼 환경별로 값을 다르게 주는 방법(Profile)
3. 설정을 코드로 안전하게 꺼내 쓰는 방법

## 1) application.yml 위치

기본 위치:

- `src/main/resources/application.yml`

## 2) 자주 쓰는 설정 3개

### (1) 서버 포트

```yaml
server:
  port: 8081
```

### (2) 로그 레벨

```yaml
logging:
  level:
    root: INFO
    com.sungkyu: DEBUG
```

### (3) JSON 응답 포맷(선택)

기본으로도 충분하지만, 타임존/날짜 등을 다루면 Jackson 설정이 필요할 수 있습니다.

## 3) Profile로 환경 분리하기

운영에서는 보통 아래처럼 나눕니다.

- local/dev: 개발 편의
- prod: 운영 안전

파일을 분리합니다.

- `application.yml` (공통)
- `application-local.yml`
- `application-prod.yml`

예: `application.yml`

```yaml
spring:
  profiles:
    active: local
```

예: `application-local.yml`

```yaml
server:
  port: 8080

logging:
  level:
    root: DEBUG
```

예: `application-prod.yml`

```yaml
server:
  port: 8080

logging:
  level:
    root: INFO
```

## 4) 실행 시 Profile 바꾸기

로컬에서 prod로 띄워보고 싶다면:

```bash
java -jar app.jar --spring.profiles.active=prod
```

IntelliJ Run Configuration에서 VM 옵션/Program arguments로도 설정할 수 있습니다.

## 5) 설정 값을 코드에서 안전하게 쓰기

초보자에게 추천하는 방식은 `@ConfigurationProperties`입니다.

```java
package com.sungkyu.demo;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app")
public record AppProperties(String name) {}
```

`application.yml`:

```yaml
app:
  name: demo
```

그리고 설정 클래스를 등록합니다(초보자용으로 가장 간단한 방식):

```java
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig {}
```

## 정리

- `application.yml`은 설정의 중심
- Profile로 local/prod 값을 분리
- 설정은 `@ConfigurationProperties`로 안전하게 주입받기

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

설정 파일은 초반에는 단순해 보여도 프로젝트가 커질수록 굉장히 중요해집니다. 로컬에서는 H2를 쓰고, 개발 서버에서는 MySQL을 쓰고, 운영에서는 별도 비밀키와 외부 주소를 쓰는 식으로 환경이 달라지기 때문입니다. 이때 프로필을 잘 나누어두면 같은 코드로도 환경별 설정을 안정적으로 분리하실 수 있습니다.

많은 분들이 처음에는 모든 설정을 `application.yml` 하나에 몰아넣다가, 어느 순간 비밀번호와 URL, 포트, 로깅 레벨이 뒤섞여 관리가 어려워집니다. 프로필은 이런 혼란을 줄이고 "지금 어떤 환경으로 켰는지"를 명확하게 만들어주는 장치라고 생각하시면 좋습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 기본 설정과 환경별 설정을 분리하고, 활성 프로필에 따라 어떤 값이 최종 적용되는지 이해하는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```yaml
# application.yml
spring:
  application:
    name: demo
  profiles:
    active: local

server:
  port: 8080
```
```yaml
# application-local.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password:
  h2:
    console:
      enabled: true
```
```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
    username: demo_user
    password: demo_pass
```

위 예시는 application.yml과 프로필을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- YAML은 들여쓰기에 매우 민감해서 공백 두 칸이 어긋나도 설정이 읽히지 않거나 전혀 다른 구조로 해석될 수 있습니다.
- 운영 비밀번호를 로컬 설정 파일에 그대로 넣고 커밋해버리는 사고가 생각보다 자주 발생합니다.
- 활성 프로필이 무엇인지 확인하지 않고 실행하면 "왜 H2가 아니라 MySQL로 붙지?" 같은 혼란을 겪기 쉽습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 공통으로 쓰는 값만 기본 `application.yml`에 두고, 환경별 값은 프로필 파일로 분리하시는 편이 좋습니다.
- 비밀번호나 토큰은 가능하면 환경 변수로 빼고, 설정 파일에는 `${ENV_NAME}` 형태로 참조하시는 습관을 들이시면 안전합니다.
- 실행 옵션으로 `--spring.profiles.active=dev`를 직접 줘보면서 값이 바뀌는지 확인해보시면 이해가 빨라집니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드)](/springboot/mvc-01-1-setup-intellij-practice), [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics), [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 1-1: IntelliJ로 첫 프로젝트 만들고 실행까지 (완전 실습 가이드)](/springboot/mvc-01-1-setup-intellij-practice)
- [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
- [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
