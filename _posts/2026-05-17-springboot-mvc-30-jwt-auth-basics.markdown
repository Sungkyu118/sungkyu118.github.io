---
layout: post
title: "SpringBoot 입문 30: JWT 인증 기본 흐름"
date: 2026-05-17 22:20:00 +0900
category: SpringBoot
permalink: /springboot/mvc-jwt-auth-basics
description: "Spring Boot에서 JWT 인증이 동작하는 기본 흐름, Access Token과 Refresh Token 역할, 토큰 검증 시 자주 헷갈리는 포인트를 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 30: JWT 인증 기본 흐름

> 이 글에서는 세션 대신 JWT를 쓸 때 로그인, 토큰 발급, 검증까지 어떤 순서로 흐르는지 이해합니다.
>
> 이전 글: [SpringBoot 입문 29: QueryDSL로 동적 조회 만들기](/springboot/mvc-querydsl-intro)
> 다음 글: [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 28: 스케줄러로 배치 작업 시작하기](/springboot/mvc-scheduler-batch-basics)
> - [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql)


세션 대신 토큰 기반 인증을 도입할 때 가장 먼저 이해해야 하는 흐름을 정리한다.

## 핵심 포인트
- 로그인 성공 시 Access Token 발급
- 요청 필터에서 토큰 검증 후 인증 객체 구성
- 만료, 재발급(Refresh Token) 정책 분리

## 정리
JWT는 상태 없는 인증에 유리하지만, 토큰 보관 위치와 만료 정책을 안전하게 설계해야 한다.

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

JWT는 세션 기반 인증과 달리 서버가 인증 상태를 별도로 저장하지 않고도 인증 정보를 전달할 수 있게 해주는 대표적인 방식입니다. 마이크로서비스나 모바일 앱 연동처럼 상태 없는 인증이 필요한 경우 많이 쓰입니다. 다만 편리한 만큼 토큰 만료, 저장 위치, 재발급 정책을 함께 이해하셔야 안전하게 사용할 수 있습니다.

처음 JWT를 배우실 때는 토큰 생성 코드보다 흐름 자체를 먼저 이해하시는 것이 좋습니다. 로그인 성공 -> 액세스 토큰 발급 -> 요청 헤더에 토큰 포함 -> 필터에서 검증 -> 인증 객체 저장이라는 순서가 머릿속에 잡히면, 구현 세부 사항은 훨씬 따라가기 쉬워집니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 토큰 발급과 검증 흐름, Authorization 헤더 파싱, SecurityContext에 인증 정보를 넣는 과정 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
@Component
public class JwtTokenProvider {

  public String resolveToken(HttpServletRequest request) {
    String bearer = request.getHeader("Authorization");
    if (bearer != null && bearer.startsWith("Bearer ")) {
      return bearer.substring(7);
    }
    return null;
  }
}
```
```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
  private final JwtTokenProvider jwtTokenProvider;

  public JwtAuthenticationFilter(JwtTokenProvider jwtTokenProvider) {
    this.jwtTokenProvider = jwtTokenProvider;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain filterChain) throws ServletException, IOException {
    String token = jwtTokenProvider.resolveToken(request);
    // token validation and authentication set
    filterChain.doFilter(request, response);
  }
}
```

위 예시는 JWT 인증을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- Authorization 헤더에서 `Bearer ` 접두어를 정확히 처리하지 않으면 토큰 검증 전에 null이나 잘못된 문자열로 흐를 수 있습니다.
- 토큰 만료 시간을 너무 길게 잡으면 보안상 부담이 커지고, 너무 짧게 잡으면 사용자 경험이 불편해질 수 있습니다.
- 토큰을 어디에 저장할지에 대한 고민 없이 바로 구현하면 XSS나 탈취 위험에 대한 감각이 약해질 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 처음에는 Access Token 흐름부터 확실히 이해하시고, Refresh Token은 그다음 단계로 나누어 학습하시는 편이 좋습니다.
- JWT는 HTTPS 전제 하에서 써야 의미가 있으므로 전송 구간 보안도 같이 생각하셔야 합니다.
- 클레임에는 정말 필요한 최소 정보만 넣는 습관을 들이시면 토큰이 훨씬 단순해집니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic), [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics), [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 30: JWT 인증 기본 흐름 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->


## 추가로 연습해보시면 좋습니다

JWT 인증 주제는 문법을 한 번 읽는 것만으로 끝내기보다, 작은 요구사항을 붙여가며 반복 연습할수록 이해가 훨씬 깊어집니다. 예를 들어 응답 메시지를 바꿔보거나, 예외 상황을 일부러 만들어보거나, 설정 값을 변경해본 뒤 결과가 어떻게 달라지는지 확인해보시면 '아는 것 같은 상태'에서 '설명할 수 있는 상태'로 빠르게 넘어가실 수 있습니다.

또한 이 단계에서는 정답 코드를 외우기보다, 왜 이런 구조를 선택했는지를 설명해보시는 연습이 정말 중요합니다. 스스로 소리 내어 '이 어노테이션은 왜 붙였는지', '이 설정은 어느 계층에 영향을 주는지', '이 에러는 왜 발생했는지'를 정리해보시면 학습 밀도가 훨씬 올라갑니다. 친절한 입문 글일수록 바로 따라 할 수 있어야 하고, 동시에 왜 그렇게 하는지도 이해할 수 있어야 한다는 점을 계속 기억해주시면 좋겠습니다.

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic)
- [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
- [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
