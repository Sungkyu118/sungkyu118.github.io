---
layout: post
title: "SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)"
date: 2026-05-17 03:10:00 +0900
category: SpringBoot
permalink: /springboot/mvc-cors-basics
description: "Spring Boot에서 CORS가 왜 필요한지, 전역 설정과 특정 경로 허용 방법, 프론트엔드 연동 시 자주 만나는 문제를 쉽게 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---

# SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)

> 이 글에서는 브라우저의 CORS 오류가 왜 나는지와 Spring MVC에서 전역으로 풀어주는 방식을 설명합니다.
>
> 이전 글: [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)
> 다음 글: [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
> - [SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)](/springboot/mvc-file-upload-download)


프론트(예: React)와 백엔드(SpringBoot)를 붙이면 가장 흔히 만나는 에러가 CORS입니다.

목표:

- CORS가 뭔지 한 문장으로 설명한다
- Spring MVC에서 전역 CORS 설정을 적용한다
- "됐는데도 안 되는" 흔한 케이스를 안다

## 1) CORS가 뭐냐

브라우저가 "다른 출처(origin)"로 요청할 때, 서버가 허용해줘야 하는 보안 정책입니다.

## 2) 전역 설정(WebMvcConfigurer)

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebCorsConfig implements WebMvcConfigurer {
  @Override
  public void addCorsMappings(CorsRegistry registry) {
    registry.addMapping("/**")
        .allowedOrigins("http://localhost:3000")
        .allowedMethods("GET", "POST", "PUT", "DELETE")
        .allowCredentials(true);
  }
}
```

`addCorsMappings`는 Spring Framework의 표준 확장 포인트입니다. citeturn0search3

## 3) 그래도 안 된다면(가장 흔한 원인)

- Spring Security를 쓰는 경우: Security 쪽 CORS 설정이 별도로 필요할 수 있음
- allowedOrigins에 `*`를 쓰면서 credentials를 true로 둔 경우(브라우저 정책)

## 정리

- CORS는 브라우저 정책
- MVC에서는 WebMvcConfigurer로 전역 설정을 줄 수 있다

