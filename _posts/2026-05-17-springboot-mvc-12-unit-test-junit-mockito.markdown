---
layout: post
title: "SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기"
date: 2026-05-17 02:00:00 +0900
category: SpringBoot
permalink: /springboot/mvc-unit-test-junit-mockito
---

# SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기

초보자가 테스트를 시작할 때 가장 좋은 출발은 "Service 단위 테스트"입니다.

목표:

- JUnit5 기본 형태 익히기
- Mockito로 Repository를 가짜로 만들어 Service 로직만 검증하기

## 1) 테스트 의존성 확인

보통 Spring Boot 프로젝트에는 기본으로 들어있습니다.

```gradle
testImplementation 'org.springframework.boot:spring-boot-starter-test'
```

## 2) 테스트 대상 서비스(예시)

```java
@Service
public class UserNameService {
  private final UserRepository userRepository;

  public UserNameService(UserRepository userRepository) {
    this.userRepository = userRepository;
  }

  public String getUpperName(Long id) {
    UserEntity u = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("USER_NOT_FOUND"));
    return u.getName().toUpperCase();
  }
}
```

## 3) Mockito로 단위 테스트 작성

```java
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

import java.util.Optional;
import org.junit.jupiter.api.Test;

class UserNameServiceTest {

  @Test
  void uppercases_name() {
    UserRepository repo = mock(UserRepository.class);
    when(repo.findById(1L)).thenReturn(Optional.of(new UserEntity("sungkyu")));

    UserNameService service = new UserNameService(repo);

    assertThat(service.getUpperName(1L)).isEqualTo("SUNGKYU");
    verify(repo).findById(1L);
  }
}
```

포인트:

- Spring 컨테이너 없이 "순수 자바 테스트"로 빠르게 돌릴 수 있음
- 실패 원인이 명확함(환경/설정 영향 적음)

## 4) 흔한 실수

- 테스트가 느려서 안 하게 됨: 처음엔 단위 테스트부터(빠름)
- SpringBootTest를 무조건 쓰기: 단위 테스트는 컨테이너 없이 시작해도 충분

## 정리

- 단위 테스트는 Service 로직 검증에 특히 유용
- Mockito로 의존성을 가짜로 만들면 테스트가 빠르고 안정적이다

