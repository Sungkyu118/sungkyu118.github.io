# SpringBoot SEO Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** SpringBoot 핵심 포스트 10개에 개별 SEO 설명문, 공통 대표 이미지, 시리즈 내부 링크 블록을 적용해 검색 결과 설명력과 글 간 이동 구조를 강화한다.

**Architecture:** 각 포스트는 front matter에 `description`과 `image`를 추가하고, 제목 아래에는 시리즈 학습 흐름을 보여주는 짧은 안내 블록을 넣는다. 수정은 10개 글을 두 묶음으로 나눠 안정적으로 적용하고, 마지막에 Jekyll 빌드와 생성 HTML을 검사해 메타 태그와 링크 출력이 의도대로 나오는지 확인한다.

**Tech Stack:** Jekyll, Markdown front matter, Liquid 기반 SEO 템플릿, GitHub Pages

---

### Task 1: 첫 번째 포스트 묶음에 SEO 메타와 시리즈 링크 적용

**Files:**
- Modify: `_posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown`

- [ ] **Step 1: 현재 front matter와 제목 아래 구조를 확인한다**

Run:

```powershell
Get-Content '_posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown' -Encoding UTF8 | Select-Object -First 20
```

Expected: 각 파일에 `description`과 `image`가 아직 없고, 제목 아래에 시리즈 안내 블록도 없는 상태를 확인한다.

- [ ] **Step 2: 각 글의 front matter에 직접 작성한 설명문과 공통 이미지를 넣는다**

Apply these values:

```yaml
2026-05-17-springboot-mvc-01-setup-intellij.markdown
description: "Spring Boot에서 IntelliJ와 Gradle로 첫 프로젝트를 만들고 Java 17 설정, Spring Initializr 선택, 실행 확인까지 한 번에 따라갈 수 있도록 단계별로 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-02-hello-controller-debug.markdown
description: "Spring Boot에서 첫 Controller를 만들고 /hello 요청을 처리하는 흐름을 브라우저 호출과 IntelliJ Debug 예제로 함께 확인합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-03-request-validation-exception.markdown
description: "Spring Boot에서 @RequestBody와 Validation을 함께 사용하는 방법, 검증 실패 시 자주 만나는 에러, 예외 응답을 정리하는 흐름을 예제로 설명합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown
description: "Spring Boot에서 JPA와 H2를 연결해 Entity와 Repository를 처음 만드는 과정, application.yml 설정, 저장과 조회 흐름을 실습 중심으로 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown
description: "Spring Boot CRUD API를 Controller, Service, Repository 구조로 끝까지 구현하면서 DTO 분리, 상태코드, 404 예외 처리까지 한 번에 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"
```

- [ ] **Step 3: 다섯 글의 제목 아래에 시리즈 안내 블록을 넣는다**

Insert blocks immediately after the first `#` heading in each file:

```markdown
2026-05-17-springboot-mvc-01-setup-intellij.markdown
> 이 글에서는 Spring Boot 프로젝트를 처음 만들고 실행하는 흐름을 실습 기준으로 정리합니다.
>
> 이전 글: 없음
> 다음 글: [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)

2026-05-17-springboot-mvc-02-hello-controller-debug.markdown
> 이 글에서는 첫 번째 GET API를 만들고 요청이 Controller까지 도달하는 흐름을 눈으로 확인합니다.
>
> 이전 글: [SpringBoot 입문 1: IntelliJ + Java + Gradle로 프로젝트 생성부터 실행까지](/springboot/mvc-setup-intellij)
> 다음 글: [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
> - [SpringBoot 입문 16: 로깅 기본(SLF4J)과 logback 설정](/springboot/mvc-logging-logback)

2026-05-17-springboot-mvc-03-request-validation-exception.markdown
> 이 글에서는 요청 DTO를 받고 검증한 뒤, 실패를 일관된 에러 응답으로 바꾸는 기본 패턴을 익힙니다.
>
> 이전 글: [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> 다음 글: [SpringBoot 입문 6: REST 기본(@RequestParam/@PathVariable)과 HTTP 상태코드](/springboot/mvc-rest-basics-params-status)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
> - [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown
> 이 글에서는 Spring Boot 프로젝트에 JPA와 H2를 붙여 첫 번째 Entity와 Repository를 만드는 과정을 따라갑니다.
>
> 이전 글: [SpringBoot 입문 7: Service 계층과 의존성 주입(DI) 기초](/springboot/mvc-service-layer-di)
> 다음 글: [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
> - [SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해](/springboot/mvc-transaction-basics)

2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown
> 이 글에서는 실전에서 가장 자주 만드는 CRUD API를 계층 구조에 맞춰 끝까지 완성하는 흐름을 정리합니다.
>
> 이전 글: [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)
> 다음 글: [SpringBoot 입문 10: 목록 API의 기본 (Pagination/Sorting)](/springboot/mvc-pagination-sorting)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> - [SpringBoot 입문 15: 예외 처리 고급(에러코드, NotFound, 공통 응답 포맷)](/springboot/mvc-exception-handling-advanced)
```

- [ ] **Step 4: 수정된 다섯 파일의 diff를 확인한다**

Run:

```powershell
git diff -- _posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown _posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown _posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown _posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown _posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown
```

Expected: front matter에 `description`, `image`가 들어가고 제목 아래에 시리즈 블록이 추가된 diff를 확인한다.

### Task 2: 두 번째 포스트 묶음에 SEO 메타와 시리즈 링크 적용

**Files:**
- Modify: `_posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-20-security-basic.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown`

- [ ] **Step 1: 현재 front matter와 도입부 구성을 확인한다**

Run:

```powershell
Get-Content '_posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-20-security-basic.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown' -Encoding UTF8 | Select-Object -First 20
Get-Content '_posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown' -Encoding UTF8 | Select-Object -First 20
```

Expected: 각 글의 핵심 주제와 제목 바로 아래 삽입 위치를 확인한다.

- [ ] **Step 2: 개별 검색 의도에 맞는 설명문과 공통 이미지를 넣는다**

Apply these values:

```yaml
2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown
description: "Spring Boot Service 로직을 JUnit5와 Mockito로 검증하는 방법, Repository를 mock으로 대체하는 이유, 단위 테스트를 시작할 때 자주 막히는 지점을 함께 설명합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown
description: "Spring Boot 3.x 프로젝트에 springdoc-openapi를 붙여 Swagger UI를 열고 API 문서를 자동화하는 방법, 운영 환경에서 주의할 점까지 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-20-security-basic.markdown
description: "Spring Security 입문자가 인증과 인가 개념을 처음 잡을 수 있도록 기본 흐름, 401과 403 차이, SecurityFilterChain 시작점을 쉽게 설명합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-23-dockerize-basics.markdown
description: "Spring Boot 애플리케이션을 Dockerfile로 이미지화하고 JAR 실행부터 컨테이너 기동까지 따라가며, 초보자가 자주 헷갈리는 Docker 기본 흐름을 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"

2026-05-17-springboot-mvc-33-github-actions-ci.markdown
description: "Spring Boot 프로젝트에 GitHub Actions CI를 붙여 push와 pull request마다 Gradle 빌드와 테스트를 자동 실행하는 방법을 입문자 기준으로 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"
```

- [ ] **Step 3: 제목 아래 시리즈 안내 블록을 추가한다**

Insert blocks immediately after the first `#` heading in each file:

```markdown
2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown
> 이 글에서는 Service 로직을 단위 테스트로 분리해서 검증하는 가장 기본적인 패턴을 익힙니다.
>
> 이전 글: [SpringBoot 입문 11: 트랜잭션(@Transactional) 기본과 흔한 오해](/springboot/mvc-transaction-basics)
> 다음 글: [SpringBoot 입문 13: MockMvc로 Controller 테스트하기(요청/응답 검증)](/springboot/mvc-web-test-mockmvc)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 14: 통합 테스트(@SpringBootTest)로 전체 흐름 확인하기](/springboot/mvc-integration-test-springboottest)
> - [SpringBoot 입문 9: CRUD API 끝까지 만들기 (Controller/Service/Repository)](/springboot/mvc-crud-api-end-to-end)

2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown
> 이 글에서는 Spring Boot API 문서를 자동으로 열어주는 Swagger UI를 붙이고, 운영에서 어디까지 공개할지 판단하는 기준까지 다룹니다.
>
> 이전 글: [SpringBoot 입문 17: Actuator로 헬스체크/운영 지표 열기(주의점 포함)](/springboot/mvc-actuator-health-metrics)
> 다음 글: [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 3: 요청 받기(@RequestBody) + Validation + 예외 처리](/springboot/mvc-request-validation-exception)
> - [SpringBoot 입문 31: 공통 응답 포맷과 예외 코드 표준화](/springboot/mvc-global-response-format)

2026-05-17-springboot-mvc-20-security-basic.markdown
> 이 글에서는 Spring Security를 처음 붙였을 때 왜 401과 403이 나오는지부터, 어떤 경로를 열고 막을지 결정하는 기초 흐름을 이해합니다.
>
> 이전 글: [SpringBoot 입문 19: CORS 기본과 전역 설정(WebMvcConfigurer)](/springboot/mvc-cors-basics)
> 다음 글: [SpringBoot 입문 21: 파일 업로드/다운로드 기본(Multipart)](/springboot/mvc-file-upload-download)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 30: JWT 인증 기본 흐름](/springboot/mvc-jwt-auth-basics)
> - [SpringBoot 입문 18: Swagger(OpenAPI) 문서 자동화(springdoc-openapi)](/springboot/mvc-openapi-swagger-springdoc)

2026-05-17-springboot-mvc-23-dockerize-basics.markdown
> 이 글에서는 Spring Boot 애플리케이션을 JAR로 만든 뒤 Docker 이미지와 컨테이너로 실행하는 가장 기본적인 배포 전환 흐름을 익힙니다.
>
> 이전 글: [SpringBoot 입문 22: JAR 빌드(bootJar)하고 실행하기](/springboot/mvc-build-jar-run)
> 다음 글: [SpringBoot 입문 24: Docker Compose로 앱+DB 함께 실행](/springboot/mvc-docker-compose-basics)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 33: GitHub Actions로 CI 자동화](/springboot/mvc-github-actions-ci)
> - [SpringBoot 입문 8: JPA + H2로 DB 시작하기 (Entity/Repository)](/springboot/mvc-jpa-h2-first-db)

2026-05-17-springboot-mvc-33-github-actions-ci.markdown
> 이 글에서는 Spring Boot 프로젝트를 GitHub Actions로 자동 빌드하고 테스트해서, 푸시와 PR 단계에서 문제를 먼저 잡는 흐름을 설명합니다.
>
> 이전 글: [SpringBoot 입문 32: Testcontainers로 MySQL 통합 테스트](/springboot/mvc-testcontainers-mysql)
> 다음 글: 없음
> 함께 보면 좋은 글:
> - [SpringBoot 입문 12: 단위 테스트(JUnit5 + Mockito)로 Service 검증하기](/springboot/mvc-unit-test-junit-mockito)
> - [SpringBoot 입문 23: Docker로 실행하기(가장 단순한 Dockerfile)](/springboot/mvc-dockerize-basics)
```

- [ ] **Step 4: 수정 내용을 diff로 확인한다**

Run:

```powershell
git diff -- _posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown _posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown _posts/2026-05-17-springboot-mvc-20-security-basic.markdown _posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown _posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown
```

Expected: 다섯 파일 모두 `description`, `image`, 시리즈 안내 블록이 추가된 diff를 확인한다.

### Task 3: 빌드와 생성 HTML 검증 후 저장소 반영

**Files:**
- Modify: `_posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-20-security-basic.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown`
- Modify: `_posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown`

- [ ] **Step 1: Jekyll 빌드를 실행한다**

Run:

```powershell
bundle exec jekyll build
```

Expected: build succeeds and `_site` is regenerated without front matter parse errors.

- [ ] **Step 2: 생성 HTML에서 메타 태그와 링크를 확인한다**

Run:

```powershell
rg -n "description|og:image|twitter:image|keywords" _site/springboot
```

Expected: 대상 포스트 HTML에 직접 작성한 `description`과 이미지 메타가 출력된다.

- [ ] **Step 3: 전체 변경 상태를 확인한다**

Run:

```powershell
git status --short
git diff -- docs/superpowers/plans/2026-06-19-springboot-seo-optimization.md
git diff -- _posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown _posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown _posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown _posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown _posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown _posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown _posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown _posts/2026-05-17-springboot-mvc-20-security-basic.markdown _posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown _posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown
```

Expected: 계획 문서와 10개 포스트만 변경된 상태를 확인한다.

- [ ] **Step 4: 변경사항을 커밋하고 푸시한다**

Run:

```powershell
git add docs/superpowers/plans/2026-06-19-springboot-seo-optimization.md _posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown _posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown _posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown _posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown _posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown _posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown _posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown _posts/2026-05-17-springboot-mvc-20-security-basic.markdown _posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown _posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown
git commit -m "Optimize SpringBoot post SEO metadata"
git push origin main
```

Expected: commit succeeds and the branch is pushed to `origin/main`.
