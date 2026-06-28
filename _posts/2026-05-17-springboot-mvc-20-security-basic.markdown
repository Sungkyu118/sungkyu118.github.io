---
layout: post
title: "SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)"
date: 2026-05-17 03:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-security-basic
description: "Spring Security 입문자가 인증과 인가 개념을 처음 잡을 수 있도록 기본 흐름, 401과 403 차이, SecurityFilterChain 시작점을 쉽게 설명합니다."
image:
  path: "/assets/img/og/springboot-security-cover.svg"
  alt: "SpringBoot Security basics article cover"
---


# SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)

> 이 글에서는 Spring Security를 처음 붙였을 때 왜 401과 403이 나오는지부터, 어떤 경로를 열고 막을지 결정하는 기초 흐름을 이해합니다.
>
> 이전 글: [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> 다음 글: [SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)](/springboot/mvc-file-upload-download)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics)
> - [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)

Security는 초보자에게 어렵지만, "기본 흐름"만 잡아도 개발을 시작할 수 있습니다.

용어:

- 인증(Authentication): 너 누구야?
- 인가(Authorization): 너 이 기능 해도 돼?

## 1) 의존성 추가

```gradle
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-security'
}
```

추가하면 기본적으로 많은 엔드포인트가 보호되기 때문에, 처음엔 "왜 401/403이 뜨지?"를 경험하게 됩니다.

## 2) 가장 쉬운 시작: 특정 경로만 열기

초보자용으로는 `/hello`, `/swagger` 등만 허용하고 나머지는 막는 형태로 시작하면 감을 잡기 좋습니다.

## 3) 실전 팁

- 처음부터 JWT까지 들어가면 복잡도가 급증합니다.
- MVC 입문 시리즈에서는 "SecurityFilterChain 개념 + 허용/차단"만 우선 잡는 걸 추천합니다.

## 정리

- Security는 인증/인가를 담당한다
- 의존성 추가만으로도 기본 보호가 켜질 수 있다
- 처음엔 경로 기반으로 허용/차단부터 잡자

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

Spring Security는 처음 보면 필터가 많고 설정 클래스도 낯설어서 어렵게 느껴질 수 있지만, 사실 핵심은 "누가 접근할 수 있는지"와 "인증 정보를 어떻게 확인할지" 두 가지입니다. 기초를 잘 잡아두시면 이후 JWT, OAuth2, 권한 분리 같은 주제로 넘어갈 때 훨씬 덜 막히게 됩니다.

입문 단계에서는 모든 것을 한 번에 이해하려고 하기보다, 인증이 필요한 URL과 열어둘 URL을 분리하고, 인증 실패 시 어떤 응답이 나가는지 보는 연습부터 해보시는 것이 좋습니다. 이렇게 접근하면 필터 체인도 훨씬 덜 복잡하게 느껴집니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 `SecurityFilterChain`으로 접근 정책을 선언하고, 인증/인가를 단계적으로 나누어 이해하는 것 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

  @Bean
  public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        )
        .httpBasic(Customizer.withDefaults());
    return http.build();
  }

  @Bean
  public UserDetailsService userDetailsService() {
    return new InMemoryUserDetailsManager(
        User.withUsername("admin")
            .password("{noop}1234")
            .roles("ADMIN")
            .build()
    );
  }
}
```

위 예시는 Spring Security 기초을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 기본 보안이 켜져 있는데 permitAll을 지정하지 않으면 의도치 않게 모든 API가 401로 막힐 수 있습니다.
- CSRF와 인증 실패를 구분하지 않으면 POST 요청이 왜 막히는지 헷갈리기 쉽습니다.
- 인가 정책보다 먼저 커스텀 필터를 붙이기 시작하면 흐름 이해 없이 설정만 복잡해질 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 HTTP Basic이나 폼 로그인으로 인증 흐름을 먼저 체감한 뒤 JWT로 넘어가시는 것이 좋습니다.
- 헬스체크, Swagger, 로그인 API처럼 열어둘 경로는 의식적으로 목록을 관리해보시는 편이 좋습니다.
- Security는 "왜 막혔는지"를 콘솔 로그와 응답 코드로 같이 보는 습관을 들이셔야 빠르게 익숙해집니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
