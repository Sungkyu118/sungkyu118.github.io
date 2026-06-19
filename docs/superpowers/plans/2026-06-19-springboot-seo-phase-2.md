# SpringBoot SEO Phase 2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 남은 SpringBoot 25개 포스트에 SEO 메타와 시리즈 링크를 확장 적용하고, 핵심 4개 글에는 개별 대표 이미지를 분화한다.

**Architecture:** SpringBoot 시리즈 전체에 공통 커버 이미지를 도입하고, 핵심 글 4개는 주제별 SVG 커버를 별도로 만든다. 남은 25개 글에는 `description`과 `image`를 추가하고, 제목 아래에는 시리즈 또는 관련 글 흐름이 드러나는 안내 블록을 넣는다. 마지막에는 Jekyll 빌드와 생성 HTML을 검사해 메타 태그와 이미지 경로가 실제로 출력되는지 검증한다.

**Tech Stack:** Jekyll, Markdown front matter, SVG social cover assets, GitHub Pages

---

### Task 1: SpringBoot 공통 커버와 핵심 4개 개별 커버를 만든다

**Files:**
- Create: `assets/img/og/springboot-series-cover.svg`
- Create: `assets/img/og/springboot-crud-cover.svg`
- Create: `assets/img/og/springboot-swagger-cover.svg`
- Create: `assets/img/og/springboot-security-cover.svg`
- Create: `assets/img/og/springboot-docker-cover.svg`

- [ ] **Step 1: 1200x630 기준의 SpringBoot 공통 커버를 추가한다**
- [ ] **Step 2: CRUD, Swagger, Security, Docker용 개별 SVG 커버 4개를 추가한다**
- [ ] **Step 3: 파일 경로와 이름이 front matter에서 재사용하기 쉬운지 확인한다**

### Task 2: 남은 SpringBoot 25개 포스트에 메타 설명과 상단 링크 블록을 추가한다

**Files:**
- Modify: `2024-02-15-aop.markdown`
- Modify: `2026-05-17-springboot-mvc-04-gradle-build-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-05-application-yml-profiles.markdown`
- Modify: `2026-05-17-springboot-mvc-06-rest-basics-params-status.markdown`
- Modify: `2026-05-17-springboot-mvc-07-service-layer-di.markdown`
- Modify: `2026-05-17-springboot-mvc-10-pagination-sorting.markdown`
- Modify: `2026-05-17-springboot-mvc-11-transaction-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-13-web-test-mockmvc.markdown`
- Modify: `2026-05-17-springboot-mvc-14-integration-test-springboottest.markdown`
- Modify: `2026-05-17-springboot-mvc-15-exception-handling-advanced.markdown`
- Modify: `2026-05-17-springboot-mvc-16-logging-logback.markdown`
- Modify: `2026-05-17-springboot-mvc-17-actuator-health-metrics.markdown`
- Modify: `2026-05-17-springboot-mvc-19-cors-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-21-file-upload-download.markdown`
- Modify: `2026-05-17-springboot-mvc-22-build-jar-run.markdown`
- Modify: `2026-05-17-springboot-mvc-24-docker-compose-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-25-spring-data-jpa-n1.markdown`
- Modify: `2026-05-17-springboot-mvc-26-redis-cache-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-27-async-event-listener.markdown`
- Modify: `2026-05-17-springboot-mvc-28-scheduler-batch-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-29-querydsl-intro.markdown`
- Modify: `2026-05-17-springboot-mvc-30-jwt-auth-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-31-global-response-format.markdown`
- Modify: `2026-05-17-springboot-mvc-32-testcontainers-mysql.markdown`
- Modify: `2026-05-18-springboot-mvc-01-1-setup-intellij-practice.markdown`

- [ ] **Step 1: 각 글에 개별 `description`을 추가한다**
- [ ] **Step 2: 기본 이미지는 `springboot-series-cover.svg`로 지정한다**
- [ ] **Step 3: 제목 바로 아래에 시리즈 안내 또는 관련 글 블록을 삽입한다**
- [ ] **Step 4: AOP와 1-1 실습 글은 시리즈 외 확장 글 성격에 맞게 관련 링크 중심 블록으로 조정한다**

### Task 3: 이미 최적화한 10개 글의 이미지 정책을 업데이트한다

**Files:**
- Modify: `2026-05-17-springboot-mvc-01-setup-intellij.markdown`
- Modify: `2026-05-17-springboot-mvc-02-hello-controller-debug.markdown`
- Modify: `2026-05-17-springboot-mvc-03-request-validation-exception.markdown`
- Modify: `2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown`
- Modify: `2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown`
- Modify: `2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown`
- Modify: `2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown`
- Modify: `2026-05-17-springboot-mvc-20-security-basic.markdown`
- Modify: `2026-05-17-springboot-mvc-23-dockerize-basics.markdown`
- Modify: `2026-05-17-springboot-mvc-33-github-actions-ci.markdown`

- [ ] **Step 1: 공통 이미지 사용 글은 `springboot-series-cover.svg`로 교체한다**
- [ ] **Step 2: CRUD, Swagger, Security, Docker 4개 글은 개별 SVG 커버 경로와 `alt`를 갖는 이미지 객체로 바꾼다**
- [ ] **Step 3: 기존 description과 시리즈 블록은 유지한다**

### Task 4: 빌드 검증 후 저장소에 반영한다

**Files:**
- Modify: `docs/superpowers/plans/2026-06-19-springboot-seo-phase-2.md`
- Modify: `assets/img/og/*.svg`
- Modify: `_posts/*.markdown` 중 SpringBoot 카테고리 대상 파일

- [ ] **Step 1: `bundle exec jekyll build`로 빌드를 검증한다**
- [ ] **Step 2: 생성된 `_site/springboot/*.html`에서 `description`, `og:image`, `twitter:image`, 내부 링크 출력 여부를 점검한다**
- [ ] **Step 3: 변경 파일 범위를 `git status --short`와 `git diff`로 확인한다**
- [ ] **Step 4: 변경사항을 커밋하고 `origin/main`에 푸시한다**
