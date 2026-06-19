# SpringBoot 핵심 포스트 SEO 보강 설계

## 1. 목적

이 설계의 목적은 블로그의 SpringBoot 카테고리에서 검색 유입 가능성이 높은 핵심 포스트 10개를 먼저 수동 최적화하고, 나머지 SpringBoot 포스트에도 같은 패턴을 확장할 수 있는 기준을 만드는 것이다.

이번 작업은 단순히 메타 태그를 채우는 수준에서 끝나지 않는다. 검색 결과에서 클릭을 유도할 수 있는 설명문, 시리즈 맥락이 드러나는 내부 링크, 대표 이미지 공통 정책을 함께 적용해 검색엔진과 사용자 모두에게 "정리된 시리즈"로 인식되도록 만드는 것이 목표다.

## 2. 현재 상태 요약

- 사이트 전역 기준 SEO 태그는 이미 동작한다.
- `lang`, `description`, `canonical`, `JSON-LD`, 기본 `og:image`, `twitter:image`는 사이트 차원에서 출력된다.
- 그러나 개별 SpringBoot 포스트에는 대부분 직접 작성한 `description`이 없다.
- 개별 포스트의 대표 이미지도 없다.
- 시리즈 간 내부 링크는 충분히 구조화되어 있지 않다.
- 따라서 검색 결과에서 글별 차별성이 약하고, 사용자가 한 글을 본 뒤 다음 글로 이동할 흐름도 약하다.

## 3. 이번 작업 범위

이번 단계에서는 SpringBoot 전체 포스트를 전면 수정하지 않는다. 검색 수요가 크고 시리즈 대표성이 높은 아래 10개 포스트를 먼저 수동 최적화한다.

1. `_posts/2026-05-17-springboot-mvc-01-setup-intellij.markdown`
2. `_posts/2026-05-17-springboot-mvc-02-hello-controller-debug.markdown`
3. `_posts/2026-05-17-springboot-mvc-03-request-validation-exception.markdown`
4. `_posts/2026-05-17-springboot-mvc-08-jpa-h2-first-db.markdown`
5. `_posts/2026-05-17-springboot-mvc-09-crud-api-end-to-end.markdown`
6. `_posts/2026-05-17-springboot-mvc-12-unit-test-junit-mockito.markdown`
7. `_posts/2026-05-17-springboot-mvc-18-openapi-swagger-springdoc.markdown`
8. `_posts/2026-05-17-springboot-mvc-20-security-basic.markdown`
9. `_posts/2026-05-17-springboot-mvc-23-dockerize-basics.markdown`
10. `_posts/2026-05-17-springboot-mvc-33-github-actions-ci.markdown`

선정 기준은 다음과 같다.

- 입문자가 많이 검색하는 주제인지
- 시리즈의 대표 글 역할을 하는지
- 이후 다른 글로 내부 링크 허브가 되기 쉬운지
- 경쟁 키워드에서 검색 의도 대응력이 중요한지

## 4. 적용 전략

### 4.1 개별 `description` 수동 작성

각 대상 포스트 front matter에 직접 작성한 `description`을 추가한다.

원칙은 다음과 같다.

- 본문 첫 문장을 기계적으로 잘라 쓰지 않는다.
- 사용자가 실제로 검색할 표현을 포함한다.
- "무엇을 배우는 글인지"가 바로 드러나야 한다.
- 너무 추상적인 개념 설명보다, 실습 포인트와 자주 만나는 문제를 같이 드러낸다.
- 길이는 검색 결과 요약으로 무너지지 않도록 지나치게 길지 않게 유지하되, 정보량은 충분히 확보한다.

예시 방향:

- 프로젝트 생성 글: IntelliJ, Java, Gradle, 실행 확인
- Validation 글: `@RequestBody`, `@Valid`, 예외 처리, 자주 나는 에러
- CRUD 글: Controller, Service, Repository, 전체 흐름
- Swagger 글: springdoc-openapi 설정, 문서 자동화, 확인 방법

### 4.2 공통 대표 이미지 적용

이번 단계에서는 SpringBoot 시리즈 공통 대표 이미지를 우선 적용한다.

정책은 다음과 같다.

- 외부 링크 이미지에 의존하지 않는다.
- 현재 저장소에서 안정적으로 서빙되는 로컬 이미지 경로를 사용한다.
- 우선 10개 포스트에 동일한 `image` 값을 넣어 "이미지 없는 포스트" 상태를 해소한다.
- 이후 성과가 좋은 글이나 중요 글은 개별 대표 이미지로 분화할 수 있게 front matter 구조는 유지한다.

초기 공통 이미지 후보는 현재 SEO 기본값으로도 사용 중인 로컬 파비콘 계열 이미지다. 이번 단계에서는 디자인 완성도보다 "미리보기 이미지 누락 방지"와 "시리즈 통일성"을 우선한다.

### 4.3 글 상단 시리즈 링크 블록 추가

각 대상 포스트의 제목 아래, 본문 초반에 짧은 시리즈 안내 블록을 추가한다.

블록의 역할은 다음과 같다.

- 사용자가 현재 글의 위치를 이해하게 한다.
- 이전 글과 다음 글로 이동시키는 학습 흐름을 만든다.
- 관련 개념 글로 내부 링크를 확장한다.
- 검색엔진에 시리즈 구조 신호를 준다.

기본 형식:

- 이 글이 다루는 학습 목표 1문장
- 이전 글 링크 1개
- 다음 글 링크 1개
- 함께 보면 좋은 관련 글 2개

이 블록은 지나치게 길지 않아야 하며, 본문을 가리지 않는 수준으로 짧고 일관되게 유지한다.

### 4.4 톤과 문구 통일

설명문과 링크 안내문은 모두 "사람이 실습하면서 배우는 글"이라는 방향으로 통일한다.

피해야 할 방식:

- 너무 짧은 키워드 나열형 설명
- "Spring Boot 소개"처럼 정보가 거의 없는 문구
- 검색 결과에서 무엇을 얻는 글인지 드러나지 않는 제목 복붙형 설명

지향하는 방식:

- 무엇을 설정하는지
- 어디서 자주 막히는지
- 예제와 함께 무엇을 해결하는지

## 5. 구현 형식

각 대상 포스트에는 아래 두 종류의 변경이 들어간다.

### 5.1 front matter

추가 또는 보강 대상:

- `description`
- `image`

예시:

```yaml
description: "Spring Boot에서 IntelliJ로 프로젝트를 만들고 Java, Gradle, 기본 의존성을 설정한 뒤 첫 실행까지 확인하는 과정을 실습 중심으로 정리합니다."
image: "/assets/img/favicons/android-chrome-512x512.png"
```

### 5.2 본문 상단 시리즈 블록

예시 형식:

```markdown
> 이 글에서는 Spring Boot 프로젝트를 처음 만들고 실행하는 흐름을 실습 기준으로 정리합니다.
> 이전 글: 없음
> 다음 글: [SpringBoot 입문 2: 첫 Controller 만들기 + Debug로 흐름 이해하기](/springboot/mvc-hello-controller-debug)
> 함께 보면 좋은 글:
> - [SpringBoot 입문 4: Gradle(build.gradle) 기본과 실행/빌드 흐름](/springboot/mvc-gradle-build-basics)
> - [SpringBoot 입문 5: application.yml과 설정 관리(Profiles) 기본](/springboot/mvc-application-yml-profiles)
```

실제 적용 시에는 각 글의 주제에 맞춰 이전/다음/관련 링크를 개별적으로 맞춘다.

## 6. 작업 순서

1. 대상 10개 포스트의 현재 front matter와 본문 구조 확인
2. 각 글의 검색 의도에 맞는 `description` 직접 작성
3. 공통 `image` 삽입
4. 글 상단 시리즈 링크 블록 삽입
5. 생성 HTML에서 메타 출력 확인
6. `bundle exec jekyll build` 검증
7. 이상 없으면 커밋 및 푸시

## 7. 검증 기준

작업 완료로 보기 위한 최소 기준은 아래와 같다.

- 10개 포스트 모두 `description`이 front matter에 존재한다.
- 10개 포스트 모두 `image`가 front matter에 존재한다.
- 생성된 HTML에서 각 포스트의 `description`이 직접 작성한 값으로 출력된다.
- 생성된 HTML에서 `og:image`와 `twitter:image`가 출력된다.
- 내부 링크가 빌드 결과 기준으로 깨지지 않는다.
- `bundle exec jekyll build`가 성공한다.

## 8. 이번 단계에서 하지 않는 것

이번 설계는 아래 작업까지 한 번에 포함하지 않는다.

- SpringBoot 35개 전체 전면 수동 최적화
- 포스트별 개별 대표 이미지 디자인 제작
- Search Console 실데이터 기반 CTR 분석 자동화
- 제목 자체 전면 개편

이 항목들은 이번 10개 최적화가 끝난 뒤 후속 작업으로 이어간다.

## 9. 후속 확장 방향

이번 작업이 끝나면 같은 패턴을 기준으로 다음 단계 확장이 가능하다.

- 나머지 SpringBoot 포스트 25개 확대 적용
- Redis 핵심 글 동일 방식 적용
- Flutter 핵심 글 동일 방식 적용
- 중요 포스트 개별 대표 이미지 분화
- Search Console 반응을 보고 메타 문구 추가 조정

## 10. 성공 기준

이번 설계의 성공은 "사이트 전역 SEO가 있다" 수준이 아니라, SpringBoot 핵심 포스트가 개별 검색 결과에서도 더 설명력 있게 보이고, 글과 글 사이의 이동 흐름이 더 분명해지는 상태를 만드는 것이다.

즉, 이번 단계의 핵심 산출물은 다음 세 가지다.

- 검색 의도에 맞는 개별 설명문
- 대표 이미지 누락 없는 포스트 메타
- 시리즈 구조가 드러나는 내부 링크
