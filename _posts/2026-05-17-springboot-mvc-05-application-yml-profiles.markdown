---
layout: post
title: "SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본"
date: 2026-05-17 00:50:00 +0900
category: SpringBoot
permalink: /springboot/mvc-application-yml-profiles
---

# SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본

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

