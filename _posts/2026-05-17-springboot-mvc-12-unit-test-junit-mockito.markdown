---
layout: post
title: "SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기"
date: 2026-05-17 02:00:00 +0900
category: SpringBoot
permalink: /springboot/mvc-unit-test-junit-mockito
description: "Spring Boot Service 로직을 JUnit5와 Mockito로 검증하는 방법, Repository를 mock으로 대체하는 이유, 단위 테스트를 시작할 때 자주 막히는 지점을 함께 설명합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기

> 이 글에서는 Service 로직을 단위 테스트로 분리해서 검증하는 가장 기본적인 패턴을 익힙니다.
>
> 이전 글: [SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해](/springboot/mvc-transaction-basics)
> 다음 글: [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest)
> - [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)

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

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

단위 테스트는 "내 메서드가 의도한 규칙을 제대로 지키는지"를 빠르고 가볍게 확인하는 도구입니다. Spring 컨텍스트를 전부 띄우지 않아도 되기 때문에 실행 속도가 빠르고, 비즈니스 규칙이 깨졌는지 즉시 알려주는 장점이 있습니다. 특히 Service 계층 로직을 검증할 때 가장 효과를 크게 느끼실 수 있습니다.

처음에는 테스트 코드를 작성하는 것이 번거롭게 느껴질 수 있지만, 수정할 때마다 직접 브라우저를 눌러보는 것보다 훨씬 안정적입니다. 나중에 로직이 바뀌어도 기존 규칙이 유지되는지 자동으로 알려주기 때문에, 결국 개발 속도를 올려주는 쪽에 가깝습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 Mockito로 외부 의존성을 가짜로 만들고, 순수하게 Service 규칙만 검증하는 흐름 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

  @Mock
  private UserRepository userRepository;

  @InjectMocks
  private UserService userService;

  @Test
  void createUser_shouldSaveAndReturnName() {
    UserEntity saved = new UserEntity("sungkyu");
    ReflectionTestUtils.setField(saved, "id", 1L);

    given(userRepository.save(any(UserEntity.class))).willReturn(saved);

    UserResponse response = userService.create("sungkyu");

    assertThat(response.id()).isEqualTo(1L);
    assertThat(response.name()).isEqualTo("sungkyu");
    then(userRepository).should().save(any(UserEntity.class));
  }
}
```

위 예시는 단위 테스트을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 단위 테스트인데도 `@SpringBootTest`를 붙여 전체 컨텍스트를 띄우면 실행 속도가 느려지고, 무엇을 검증하는지 흐려질 수 있습니다.
- Mock 객체가 어떤 값을 리턴해야 하는지 정의하지 않으면 null 때문에 실패하는데, 이를 실제 서비스 로직 오류로 오해하실 수 있습니다.
- 검증 포인트가 너무 많으면 테스트가 읽기 어려워지고, 실패했을 때 무엇이 깨졌는지 파악하기 어려워집니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 한 테스트는 한 규칙만 검증한다고 생각하시면 구조를 잡기 쉽습니다.
- given-when-then 흐름으로 코드를 정리하면 테스트 의도가 훨씬 분명해집니다.
- 단위 테스트는 빠르게 많이 돌릴 수 있어야 하므로, DB나 외부 API까지 끌고 오지 않는 방향을 우선 생각해보시는 것이 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.

## 마무리 정리

SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->
