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

