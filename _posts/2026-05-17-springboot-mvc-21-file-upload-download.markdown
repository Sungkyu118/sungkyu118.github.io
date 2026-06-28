---
layout: post
title: "SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)"
date: 2026-05-17 03:30:00 +0900
category: SpringBoot
permalink: /springboot/mvc-file-upload-download
description: "Spring Boot에서 Multipart 기반 파일 업로드와 다운로드를 구현하는 기본 흐름, 요청 설정과 주의사항을 입문자 관점으로 정리합니다."
image: "/assets/img/og/springboot-series-cover.svg"
---


# SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)

> 이 글에서는 파일 업로드와 다운로드 API를 만들 때 Multipart 요청과 응답 헤더를 어떻게 다뤄야 하는지 살펴봅니다.
>
> 이전 글: [SpringBoot 입문 20: Spring Security 아주 기초(인증/인가 감 잡기)](/springboot/mvc-security-basic)
> 다음 글: [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> - [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)


초보자 프로젝트에서도 파일 업로드는 자주 필요합니다(프로필 이미지 등).

목표:

- Multipart 업로드 API 만들기
- 응답으로 저장 경로/URL을 내려주기

## 1) 업로드 컨트롤러

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.UUID;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

@RestController
public class UploadController {

  @PostMapping("/upload")
  public ResponseEntity<String> upload(@RequestPart MultipartFile file) throws IOException {
    String name = UUID.randomUUID() + "-" + file.getOriginalFilename();
    Path target = Path.of("uploads").resolve(name);
    Files.createDirectories(target.getParent());
    Files.write(target, file.getBytes());
    return ResponseEntity.ok("saved: " + target.toString());
  }
}
```

## 2) 주의점

- 운영에서는 로컬 디스크 저장 대신 S3 같은 스토리지를 쓰는 경우가 많음
- 업로드 파일 타입/크기 제한을 꼭 걸어야 함

## 정리

- Multipart는 `MultipartFile`로 받는다
- 초보자 단계에서는 로컬 저장으로 시작하되, 운영에서는 스토리지 전략이 필요하다

<!-- codex-springboot-detail:start -->

## 여기서 한 단계 더 이해해보겠습니다

파일 처리는 단순한 CRUD보다 고려할 것이 많습니다. 용량 제한, 저장 경로, 파일명 충돌, 다운로드 헤더, 보안 이슈까지 함께 생각해야 하기 때문입니다. 그래서 입문 단계에서는 일단 Multipart 요청을 받고 파일을 저장하거나 읽는 기본 흐름을 확실히 이해하는 것이 중요합니다.

특히 브라우저나 Postman으로 테스트할 때는 일반 JSON 요청과 달리 multipart/form-data 구조를 사용하므로, 요청 형식 자체를 익혀두셔야 합니다. 다운로드도 마찬가지로 단순 문자열 응답이 아니라 헤더와 바이트 흐름을 같이 다뤄야 해서 처음에는 낯설 수 있습니다.

이 주제에서 특히 중요하게 보셔야 하는 부분은 `MultipartFile` 처리, 저장 파일명 관리, 다운로드 응답 헤더 구성 입니다. 처음 읽으실 때는 모든 문장을 외우려고 하기보다, 코드가 어느 계층에 놓이고 어떤 순서로 실행되는지를 천천히 따라가보시면 좋습니다. Spring Boot 학습은 각각의 기술을 따로 외우는 공부라기보다, 여러 조각이 실제 요청 흐름 안에서 어떻게 맞물리는지 이해하는 과정에 가깝습니다.

## 예시 코드로 흐름을 잡아보겠습니다

```java
package com.example.demo.file;

import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.nio.file.Files;
import java.nio.file.Path;

@RestController
@RequestMapping("/files")
public class FileController {

  @PostMapping("/upload")
  public String upload(@RequestParam MultipartFile file) throws Exception {
    Path uploadDir = Path.of("uploads");
    Files.createDirectories(uploadDir);

    Path savedPath = uploadDir.resolve(file.getOriginalFilename());
    file.transferTo(savedPath);
    return savedPath.toString();
  }

  @GetMapping("/download/{name}")
  public ResponseEntity<Resource> download(@PathVariable String name) throws Exception {
    Path filePath = Path.of("uploads").resolve(name);
    Resource resource = new UrlResource(filePath.toUri());

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + name + "\"")
        .body(resource);
  }
}
```

위 예시는 파일 업로드와 다운로드을 가장 작게 체험할 수 있는 흐름으로 구성했습니다. 처음에는 코드 한 줄 한 줄의 문법보다, 선언 위치와 실행 시점을 중심으로 읽어보시는 편이 훨씬 이해가 잘 되실 것입니다. 예를 들어 설정 클래스인지, Controller인지, Service인지에 따라 책임이 달라지고, 그 책임이 곧 Spring Boot 구조를 읽는 기준이 됩니다.

또 하나 중요한 점은 '지금 당장 돌아가는 코드'와 '나중에도 유지보수하기 쉬운 코드'는 조금 다를 수 있다는 사실입니다. 그래서 예제를 실행해보신 뒤에는 변수명, 메서드명, 경로, 설정 값을 직접 한두 번 바꿔보시면서 어떤 부분이 고정 규칙이고 어떤 부분이 프로젝트 상황에 따라 달라지는지 구분해보시면 정말 큰 도움이 됩니다.

## 자주 하는 실수와 에러도 같이 보겠습니다

- 사용자가 올린 원본 파일명을 그대로 저장하면 중복이나 경로 조작 문제를 일으킬 수 있습니다.
- 파일 용량 제한을 두지 않으면 너무 큰 업로드가 들어와 메모리나 디스크 문제가 생길 수 있습니다.
- 다운로드 응답에 Content-Disposition 헤더를 빼먹으면 브라우저가 기대와 다르게 처리할 수 있습니다.

이런 실수들은 대부분 개념을 몰라서라기보다, 실행 순서나 설정 위치를 한 번에 다 보지 못해서 생깁니다. 그래서 에러가 나면 조급하게 코드를 많이 바꾸기보다, 로그와 요청 흐름을 기준으로 '어디까지는 정상이고 어디서부터 어긋났는지'를 차분히 확인해보시는 편이 좋습니다.

## 실무에서는 이렇게 적용하시면 좋습니다

- 실무에서는 UUID 같은 안전한 저장 파일명을 별도로 만들고, 원본 이름은 메타데이터로 관리하시는 편이 좋습니다.
- 업로드 허용 확장자와 MIME 타입을 검증해두시면 보안상 훨씬 안전합니다.
- 파일 저장 위치를 로컬 디스크로 시작하더라도 나중에 S3 같은 외부 스토리지로 바꿀 수 있게 추상화를 고민해보시면 좋습니다.

처음부터 완벽한 구조를 만들려고 하실 필요는 없습니다. 다만 작은 예제라도 책임 분리, 예외 처리, 설정 관리, 로그 확인 같은 기본 습관을 함께 가져가시면 나중에 프로젝트가 커졌을 때 훨씬 덜 흔들리게 됩니다. 특히 Spring Boot는 편의 기능이 많아서 '왜 이렇게 동작하는지'를 놓치기 쉬우므로, 동작 원리를 설명할 수 있을 정도로 천천히 정리해두시는 것이 좋습니다.


<!-- codex-springboot-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status), [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics), [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format) 글도 함께 읽어보시면 좋습니다. 앞단의 준비 과정과 다음 단계의 확장 흐름이 자연스럽게 이어져서, 지금 배우는 내용이 프로젝트 안에서 어디에 연결되는지 더 분명하게 감을 잡으실 수 있습니다.

<!-- codex-springboot-inline-links:end -->
## 마무리 정리

SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart) 글을 공부하실 때는 한 번 읽고 끝내기보다, 직접 실행해보고 일부러 에러도 만들어보시고, 설정이나 코드 값을 조금씩 바꿔보시는 방식으로 접근해보시길 권합니다. 그렇게 하시면 단순 요약 글보다 훨씬 오래 기억에 남고, 다음 주제인 데이터 접근, 보안, 테스트, 운영 자동화로 넘어갈 때도 이해가 자연스럽게 이어집니다. 지금 단계에서 가장 중요한 것은 완벽함보다 흐름을 잡는 것이니, 천천히 따라오셔도 충분합니다.

<!-- codex-springboot-detail:end -->

<!-- codex-springboot-links:start -->

## 이어서 읽어보시면 좋습니다

- [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
- [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
- [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

지금 글과 연결되는 흐름으로 골라두었으니, 바로 다음 실습이나 비교 학습을 이어가실 때 참고해보시면 좋겠습니다.

<!-- codex-springboot-links:end -->
