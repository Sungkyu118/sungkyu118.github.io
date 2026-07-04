---
layout: post
title: "Vue 입문 64: Composable 안에서 watch를 사용할 때 주의할 점"
date: 2026-07-05 07:04:00 +0900
category: Vue
permalink: /vue/intro-64-watch-inside-composable-cautions
description: "Vue Composable 내부에서 watch를 사용할 때 감시 대상, 즉시 실행, 정리 함수, 오래된 요청 문제를 어떻게 다루면 좋은지 설명합니다."
image:
  path: "/assets/img/og/vue-series-cover.svg"
  alt: "Vue 입문 시리즈 대표 이미지"
tags: [Vue, Composable, watch, 비동기]
---

# Vue 입문 64: Composable 안에서 watch를 사용할 때 주의할 점

> Composable 내부 watch가 편리한 만큼 감시 대상과 정리 타이밍을 명확히 해야 한다는 점을 배웁니다.
>
> 이전 글: [Vue 입문 63: useFetch를 만들며 로딩, 성공, 실패 상태 다루기](/vue/intro-63-use-fetch-loading-error)
> 다음 글: [Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제](/vue/intro-65-composable-lifecycle-dependency)
> 함께 보면 좋은 글:
> - [Vue 입문 63: useFetch를 만들며 로딩, 성공, 실패 상태 다루기](/vue/intro-63-use-fetch-loading-error)
> - [Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제](/vue/intro-65-composable-lifecycle-dependency)
> - [Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?](/vue/intro-61-what-is-composable)

Composable을 만들다 보면 어떤 값이 바뀔 때 자동으로 다시 실행되는 로직이 필요해집니다. 검색어가 바뀌면 목록을 다시 조회하거나, route param이 바뀌면 상세 데이터를 다시 가져오는 상황이 대표적입니다. 이때 `watch`를 Composable 안에 넣으면 컴포넌트마다 같은 감시 코드를 반복하지 않아도 됩니다.

하지만 `watch`는 편리한 만큼 흐름이 숨어버리기 쉽습니다. 컴포넌트에서는 `useSomething()` 한 줄만 보이는데 내부에서 API 요청이 자동으로 일어나면, 나중에 "왜 요청이 두 번 가지?" 같은 문제를 만났을 때 추적이 어려울 수 있습니다. 그래서 Composable 내부의 watch는 감시 대상과 실행 조건을 분명히 해야 합니다.

특히 비동기 요청과 watch가 만나면 오래된 요청 문제가 생길 수 있습니다. 사용자가 검색어를 빠르게 바꿨는데 첫 번째 요청이 늦게 끝나서 최신 결과를 덮어쓰는 상황입니다. 이런 경우에는 정리 함수나 요청 식별자를 사용해 오래된 결과를 무시해야 합니다.

## 이번 글에서 먼저 잡을 관점

이번 Composable 구간에서는 "코드를 함수로 빼는 것"보다 "상태와 동작의 책임을 어디까지 묶을 것인가"를 더 중요하게 보겠습니다. Composable은 반복 코드를 줄이는 도구이기도 하지만, 화면 컴포넌트가 너무 많은 일을 떠안지 않게 도와주는 설계 단위이기도 합니다.

읽으실 때는 예제의 파일 이름보다 데이터 흐름을 먼저 따라가 보시면 좋습니다. 어떤 값이 `ref`로 만들어지는지, 어떤 함수가 상태를 바꾸는지, 컴포넌트는 무엇을 몰라도 되는지 확인해보면 Composable의 장점과 위험이 훨씬 선명하게 보입니다.

지금 단계에서는 완벽한 구조를 외우려고 하기보다, 왜 이 코드가 컴포넌트 안에 있으면 불편해지는지, 분리하면 어떤 장점이 생기는지, 반대로 너무 분리하면 어떤 비용이 생기는지를 함께 생각해보시면 좋습니다. 기술을 배우는 속도보다 판단 기준을 만드는 속도가 더 중요할 때가 많습니다.

## 작은 실습으로 확인해보기

아래 예제는 검색어 ref를 받아 자동으로 상품 목록을 다시 조회하는 Composable입니다. watch의 세 번째 인자인 cleanup 등록 함수를 사용해 이전 요청을 취소합니다.

{% raw %}
~~~vue
// composables/useProductSearch.js
import { ref, watch } from 'vue'

export function useProductSearch(keyword) {
  const items = ref([])
  const loading = ref(false)
  const error = ref(null)

  watch(
    keyword,
    async (nextKeyword, _oldKeyword, onCleanup) => {
      if (!nextKeyword.trim()) {
        items.value = []
        return
      }

      const controller = new AbortController()

      onCleanup(() => {
        controller.abort()
      })

      loading.value = true
      error.value = null

      try {
        const response = await fetch(`/api/products?keyword=${encodeURIComponent(nextKeyword)}`, {
          signal: controller.signal
        })

        if (!response.ok) {
          throw new Error('상품 검색 요청에 실패했습니다.')
        }

        items.value = await response.json()
      } catch (err) {
        if (err.name !== 'AbortError') {
          error.value = err
        }
      } finally {
        loading.value = false
      }
    },
    { immediate: true }
  )

  return {
    items,
    loading,
    error
  }
}
~~~
{% endraw %}

## 코드가 움직이는 순서

`watch`는 `keyword`가 바뀔 때마다 실행됩니다. `immediate: true`를 넣었기 때문에 처음 Composable이 실행될 때도 한 번 동작합니다. 중요한 부분은 `AbortController`와 `onCleanup`입니다. 검색어가 바뀌어 watch 콜백이 다시 실행되기 직전에 이전 요청을 취소해서 오래된 응답이 최신 결과를 덮어쓰지 않도록 막습니다.

예제를 따라 작성할 때는 먼저 상태가 만들어지는 위치를 찾고, 그다음 그 상태를 바꾸는 함수를 찾고, 마지막으로 템플릿에서 어떤 조건으로 화면이 바뀌는지 확인해보세요. Vue 코드는 겉으로는 HTML과 JavaScript가 섞여 보이지만, 실제로는 "상태가 바뀌고, 계산값이 갱신되고, 템플릿이 다시 그려지는 흐름"을 반복합니다. 이 순서를 말로 설명할 수 있으면 문법을 조금 잊어도 다시 찾아가며 구현할 수 있습니다.

## 자주 만나는 에러와 유의사항

- **확인 포인트**: watch 대상이 ref인지 getter인지 분명히 해야 합니다. reactive 객체의 속성을 바로 넘기면 값이 끊겨 감시가 동작하지 않는 경우가 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: `immediate: true`를 넣으면 초기 실행이 생깁니다. 컴포넌트의 onMounted에서도 같은 함수를 호출하고 있다면 요청이 두 번 갈 수 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: 정리 함수를 넣지 않은 비동기 watch는 빠른 입력, 빠른 페이지 이동, 느린 네트워크에서 오래된 결과를 보여줄 수 있습니다. 예제 단계에서 잘 보이지 않는 실무형 버그입니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.

문제가 생겼을 때는 한 번에 모든 코드를 의심하지 말고, 입력값, 상태 변경 함수, 반환값 또는 라우트 이동 결과를 차례로 확인해보세요. 특히 Composable과 Router는 코드가 여러 파일에 나뉘기 쉬워서 "어디서 바뀌었는지"를 놓치기 쉽습니다. 브라우저 콘솔, Vue Devtools, Network 탭을 함께 보면 눈으로 보이지 않는 상태 변화와 요청 흐름을 훨씬 빠르게 확인할 수 있습니다.

## 실무에서 생각할 점

watch를 Composable 내부에 넣으면 호출하는 쪽은 편해지지만, 자동 실행이라는 성격 때문에 흐름이 덜 눈에 띌 수 있습니다. 사용자가 명시적으로 버튼을 눌러 요청해야 하는 화면이라면 자동 watch보다 `execute` 함수를 반환하는 구조가 더 읽기 쉬울 수 있습니다.

실무에서는 정답 패턴을 한 번에 고르는 것보다 현재 팀과 프로젝트의 복잡도에 맞는 선택을 하는 것이 중요합니다. 작은 화면에서는 단순한 코드가 가장 좋은 코드일 수 있고, 여러 화면에서 같은 문제가 반복되기 시작하면 그때 분리하는 편이 더 안전할 수 있습니다. 반대로 인증, 결제, 관리자 권한처럼 실수의 영향이 큰 영역은 초반부터 책임 경계를 조금 더 엄격하게 잡는 것이 좋습니다.

## 디버깅할 때 확인할 순서

1. 현재 값이 어디에서 만들어지는지 확인합니다.
2. 그 값을 바꾸는 함수가 어느 파일에 있는지 확인합니다.
3. 값이 바뀐 뒤 화면이 어떤 조건으로 다시 그려지는지 확인합니다.
4. 비동기 요청이 있다면 loading, success, error 상태가 모두 처리되는지 확인합니다.
5. 라우터 이동이 있다면 URL, route params, query, guard 결과를 차례로 확인합니다.

이 순서를 습관처럼 적용하면 문제를 만났을 때 훨씬 덜 흔들립니다. 특히 입문 단계에서는 코드를 많이 아는 것보다, 문제가 생겼을 때 확인할 순서를 알고 있는 것이 더 큰 힘이 됩니다.

## 오늘 내용 정리

이번 글에서는 'Vue 입문 64: Composable 안에서 watch를 사용할 때 주의할 점' 주제를 중심으로 Vue 프로젝트에서 자주 만나는 구조 문제를 살펴봤습니다. 핵심은 코드를 예쁘게 나누는 것이 아니라, 상태와 책임이 어디에 있는지 설명할 수 있게 만드는 것입니다. 예제 코드를 그대로 따라 작성한 뒤에는 이름을 바꿔보거나, 실패 상황을 일부러 만들어보거나, 조건을 하나 추가해보면서 흐름을 직접 확인해보시면 좋습니다.

다음 글로 넘어가기 전에 오늘 예제에서 "이 코드는 컴포넌트가 알아야 할 일인가, 아니면 별도 함수나 라우터 설정이 맡아야 할 일인가"를 한 번 말로 정리해보세요. 이 질문에 답하는 연습이 쌓이면 Vue 프로젝트가 커졌을 때도 무작정 파일을 늘리거나 한 파일에 모두 몰아넣는 실수를 줄일 수 있습니다.

## 공식 문서와 같이 확인하면 좋은 부분

- [Vue 공식 문서 - Watchers](https://vuejs.org/guide/essentials/watchers.html)
- [Vue 공식 문서 - Composables](https://vuejs.org/guide/reusability/composables.html)

공식 문서는 문법의 기준점입니다. 블로그 글로 흐름을 잡고, 공식 문서로 정확한 옵션과 예외를 확인하면 오래된 예제나 다른 스타일의 코드를 만났을 때도 덜 흔들립니다. 특히 Vue Router와 Composable 관련 API는 작성 방식이 다양해 보일 수 있으므로, 현재 공식 문서를 기준으로 다시 확인하는 습관을 추천드립니다.

## 이어서 읽어보시면 좋습니다

### [Vue 입문 63: useFetch를 만들며 로딩, 성공, 실패 상태 다루기](/vue/intro-63-use-fetch-loading-error)

이전 글에서 다룬 흐름을 다시 확인하는 용도로 좋습니다. 지금 글은 앞 글에서 만든 개념을 한 단계 확장하기 때문에, 예제의 전제가 갑자기 빠르게 느껴진다면 바로 앞 글로 돌아가 데이터가 어디서 시작되고 어떤 함수가 상태를 바꾸는지 다시 짚어보시면 좋습니다. 시리즈 글은 작은 기준이 계속 누적되는 구조라서, 이전 글을 다시 읽는 것이 오히려 가장 빠른 복습이 될 때가 많습니다.

### [Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제](/vue/intro-65-composable-lifecycle-dependency)

다음 글은 지금 배운 기준을 바로 이어서 적용하는 내용입니다. 현재 글에서 개념의 필요성을 이해했다면 다음 글에서는 그 개념이 다른 상황에서 어떻게 변형되는지 볼 수 있습니다. 이렇게 바로 다음 예제로 넘어가면 문법을 외우는 느낌보다 실제 프로젝트에서 판단 기준을 넓혀가는 느낌으로 학습할 수 있습니다.

### [Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?](/vue/intro-61-what-is-composable)

Composable 구간의 출발점으로 돌아가고 싶을 때 좋은 글입니다. 지금 글에서 다루는 예제가 조금 복잡하게 느껴진다면, 먼저 Composable이 일반 유틸 함수와 어떻게 다른지 다시 정리해보는 편이 좋습니다. 반응형 상태, 생명주기, 반환값 설계를 구분하면 뒤의 API 호출과 폼 검증 예제도 훨씬 덜 부담스럽게 읽힙니다.

Vue 입문 시리즈는 한 편씩 따로 읽어도 되지만, 가능하면 순서대로 따라가시는 것을 추천드립니다. 앞에서 만든 작은 감각이 뒤의 문법을 이해하는 발판이 되기 때문입니다. 지금 단계에서 중요한 것은 빠르게 많이 아는 것보다, 상태 흐름과 화면 이동 기준을 천천히 몸에 익히는 것입니다.