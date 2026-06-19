---
layout: post
title: "SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)"
date: 2026-05-17 01:30:00 +0900
category: SpringBoot
permalink: /springboot/mvc-crud-api-end-to-end
description: "Spring Boot CRUD API를 Controller, Service, Repository 구조로 끝까지 구현하면서 DTO 분리, 상태코드, 404 예외 처리까지 한 번에 정리합니다."
image:
  path: "/assets/img/og/springboot-crud-cover.svg"
  alt: "SpringBoot CRUD API article cover"
---

# SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)

> 이 글에서는 실전에서 가장 자주 만드는 CRUD API를 계층 구조에 맞춰 끝까지 완성하는 흐름을 정리합니다.
>
> 이전 글: [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)
> 다음 글: [SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)](/springboot/mvc-pagination-sorting)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> - [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)

이번 글은 초보자가 "진짜 API 개발"을 시작할 수 있도록 CRUD를 끝까지 한 번 완주합니다.

목표:

- Create/Read/Update/Delete 엔드포인트 만들기
- DTO와 Entity를 구분하기
- 상태코드와 예외(404) 처리

## 1) DTO 정의

```java
package com.sungkyu.demo;

import jakarta.validation.constraints.NotBlank;

public record CreateUserRequest(@NotBlank String name) {}
public record UpdateUserRequest(@NotBlank String name) {}
public record UserResponse(Long id, String name) {}
```

## 2) Service 구현

```java
package com.sungkyu.demo;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class UserCrudService {
  private final UserRepository userRepository;

  public UserCrudService(UserRepository userRepository) {
    this.userRepository = userRepository;
  }

  @Transactional
  public UserResponse create(String name) {
    UserEntity saved = userRepository.save(new UserEntity(name));
    return new UserResponse(saved.getId(), saved.getName());
  }

  @Transactional(readOnly = true)
  public UserResponse get(Long id) {
    UserEntity e = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("USER_NOT_FOUND"));
    return new UserResponse(e.getId(), e.getName());
  }

  @Transactional
  public UserResponse update(Long id, String name) {
    UserEntity e = userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("USER_NOT_FOUND"));
    // 단순화를 위해 setter 대신 새 엔티티 만들지 않고, 필드만 수정하는 형태로 가정
    // (실전에서는 setter 또는 도메인 메서드로 갱신)
    e = new UserEntity(name); // 초보자용 예시(실전에서는 주의)
    UserEntity saved = userRepository.save(e);
    return new UserResponse(saved.getId(), saved.getName());
  }

  @Transactional
  public void delete(Long id) {
    if (!userRepository.existsById(id)) {
      throw new NotFoundException("USER_NOT_FOUND");
    }
    userRepository.deleteById(id);
  }
}
```

주의: 위 업데이트는 초보자용으로 "흐름"만 보여주는 예시라서, 실제로는 Entity에 변경 메서드/ setter를 두고 영속성 컨텍스트에서 update되는 방식을 이해하는 게 중요합니다. 다음 글에서 이 부분을 더 안전하게 바꿔봅니다.

## 3) Controller 구현

```java
package com.sungkyu.demo;

import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/users")
public class UserCrudController {
  private final UserCrudService service;

  public UserCrudController(UserCrudService service) {
    this.service = service;
  }

  @PostMapping
  public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req) {
    return ResponseEntity.status(201).body(service.create(req.name()));
  }

  @GetMapping("/{id}")
  public UserResponse get(@PathVariable Long id) {
    return service.get(id);
  }

  @PutMapping("/{id}")
  public UserResponse update(@PathVariable Long id, @Valid @RequestBody UpdateUserRequest req) {
    return service.update(id, req.name());
  }

  @DeleteMapping("/{id}")
  public ResponseEntity<Void> delete(@PathVariable Long id) {
    service.delete(id);
    return ResponseEntity.noContent().build();
  }
}
```

## 4) 404/에러 처리

```java
package com.sungkyu.demo;

public class NotFoundException extends RuntimeException {
  public NotFoundException(String message) {
    super(message);
  }
}
```

그리고 `@RestControllerAdvice`에서 404 응답 포맷을 통일합니다(입문 3편의 구조 확장).

## 정리

- CRUD는 Controller/Service/Repository로 계층을 나눠서 만든다
- DTO로 외부 입력/출력을 통제한다
- 예외/상태코드를 일관되게 유지하면 프론트/클라이언트가 편해진다

