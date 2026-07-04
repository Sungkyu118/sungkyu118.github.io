---
layout: post
title: "Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?"
date: 2026-07-05 07:01:00 +0900
category: Vue
permalink: /vue/intro-61-what-is-composable
description: "Vue Composable이 일반 유틸 함수와 어떻게 다르고, 반응형 상태와 생명주기까지 함께 다룰 때 어떤 장점과 주의점이 있는지 설명합니다."
image:
  path: "/assets/img/og/vue-series-cover.svg"
  alt: "Vue 입문 시리즈 대표 이미지"
tags: [Vue, Composable, Composition API, 재사용]
---

# Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?

> Composable을 단순 함수 분리가 아니라 Vue 반응성과 생명주기를 재사용하는 방식으로 이해합니다.
>
> 이전 글: [Vue 입문 60: 재사용 컴포넌트에서 접근성과 사용성을 함께 고려하기](/vue/intro-60-reusable-component-accessibility)
> 다음 글: [Vue 입문 62: useCounter로 가장 작은 Composable 만들어보기](/vue/intro-62-use-counter-composable)
> 함께 보면 좋은 글:
> - [Vue 입문 60: 재사용 컴포넌트에서 접근성과 사용성을 함께 고려하기](/vue/intro-60-reusable-component-accessibility)
> - [Vue 입문 62: useCounter로 가장 작은 Composable 만들어보기](/vue/intro-62-use-counter-composable)
> - [Vue 입문 41: Options API와 Composition API를 비교해서 이해하기](/vue/intro-41-options-vs-composition-api)

Vue를 어느 정도 사용하다 보면 여러 컴포넌트에서 비슷한 코드가 반복되기 시작합니다. 예를 들어 창 크기를 감지하는 코드, API를 호출하면서 loading과 error를 관리하는 코드, 폼 입력값을 검증하는 코드가 여러 화면에 흩어질 수 있습니다. 처음에는 복사해서 붙여넣는 편이 빠르게 느껴지지만, 시간이 지나면 같은 버그를 여러 파일에서 고쳐야 하고, 어떤 화면은 최신 방식으로 고쳤는데 다른 화면은 예전 방식으로 남는 일이 생깁니다.

이때 등장하는 개념이 Composable입니다. Composable은 Vue Composition API에서 상태와 로직을 재사용하기 위해 만드는 함수입니다. 이름만 보면 일반 JavaScript 유틸 함수와 비슷해 보이지만, 핵심 차이는 `ref`, `computed`, `watch`, `onMounted`, `onUnmounted` 같은 Vue의 반응형 기능과 생명주기 기능을 함께 품을 수 있다는 점입니다.

그래서 Composable을 배울 때는 "함수로 빼면 되는 것"이라고만 이해하면 조금 위험합니다. 단순 계산은 유틸 함수로 충분하고, 화면 상태와 연결되는 로직은 Composable이 더 어울립니다. 이 구분을 초반에 잡아두면 이후 `useFetch`, `useForm`, `useAuth` 같은 이름을 만났을 때 훨씬 덜 헷갈립니다.

## 이번 글에서 먼저 잡을 관점

이번 Composable 구간에서는 "코드를 함수로 빼는 것"보다 "상태와 동작의 책임을 어디까지 묶을 것인가"를 더 중요하게 보겠습니다. Composable은 반복 코드를 줄이는 도구이기도 하지만, 화면 컴포넌트가 너무 많은 일을 떠안지 않게 도와주는 설계 단위이기도 합니다.

읽으실 때는 예제의 파일 이름보다 데이터 흐름을 먼저 따라가 보시면 좋습니다. 어떤 값이 `ref`로 만들어지는지, 어떤 함수가 상태를 바꾸는지, 컴포넌트는 무엇을 몰라도 되는지 확인해보면 Composable의 장점과 위험이 훨씬 선명하게 보입니다.

지금 단계에서는 완벽한 구조를 외우려고 하기보다, 왜 이 코드가 컴포넌트 안에 있으면 불편해지는지, 분리하면 어떤 장점이 생기는지, 반대로 너무 분리하면 어떤 비용이 생기는지를 함께 생각해보시면 좋습니다. 기술을 배우는 속도보다 판단 기준을 만드는 속도가 더 중요할 때가 많습니다.

## 작은 실습으로 확인해보기

아래 예제는 일반 유틸 함수와 Composable을 나란히 비교합니다. 가격 포맷처럼 Vue 상태와 상관없는 일은 유틸 함수로 두고, 창 너비처럼 화면 생명주기와 반응형 상태가 필요한 일은 Composable로 분리합니다.

{% raw %}
~~~vue
// utils/formatPrice.js
export function formatPrice(value) {
  return new Intl.NumberFormat('ko-KR').format(value) + '원'
}

// composables/useWindowWidth.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowWidth() {
  const width = ref(window.innerWidth)

  function updateWidth() {
    width.value = window.innerWidth
  }

  onMounted(() => {
    window.addEventListener('resize', updateWidth)
  })

  onUnmounted(() => {
    window.removeEventListener('resize', updateWidth)
  })

  return {
    width
  }
}

// ProductSummary.vue
<script setup>
import { computed } from 'vue'
import { formatPrice } from '@/utils/formatPrice'
import { useWindowWidth } from '@/composables/useWindowWidth'

const { width } = useWindowWidth()

const layout = computed(() => {
  return width.value < 768 ? 'mobile' : 'desktop'
})

const priceText = formatPrice(12900)
</script>

<template>
  <section>
    <p>현재 레이아웃: {{ layout }}</p>
    <p>상품 가격: {{ priceText }}</p>
  </section>
</template>
~~~
{% endraw %}

## 코드가 움직이는 순서

여기서 `formatPrice`는 Vue와 아무 관련이 없습니다. 숫자를 받아 문자열을 돌려주면 끝이므로 테스트하기 쉽고, React나 Node.js 코드에서도 그대로 쓸 수 있습니다. 반면 `useWindowWidth`는 `ref`로 상태를 만들고, 컴포넌트가 화면에 붙었을 때 이벤트를 등록하며, 사라질 때 이벤트를 정리합니다. 이런 코드는 Vue 컴포넌트의 생명주기와 연결되므로 Composable로 분리하는 편이 자연스럽습니다.

예제를 따라 작성할 때는 먼저 상태가 만들어지는 위치를 찾고, 그다음 그 상태를 바꾸는 함수를 찾고, 마지막으로 템플릿에서 어떤 조건으로 화면이 바뀌는지 확인해보세요. Vue 코드는 겉으로는 HTML과 JavaScript가 섞여 보이지만, 실제로는 "상태가 바뀌고, 계산값이 갱신되고, 템플릿이 다시 그려지는 흐름"을 반복합니다. 이 순서를 말로 설명할 수 있으면 문법을 조금 잊어도 다시 찾아가며 구현할 수 있습니다.

## 자주 만나는 에러와 유의사항

- **확인 포인트**: 모든 함수를 `useSomething`으로 만들 필요는 없습니다. Vue 반응형 상태나 생명주기를 쓰지 않는 순수 계산 함수까지 Composable로 만들면 오히려 의도가 흐려집니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: Composable 안에서 이벤트 리스너나 타이머를 만들었다면 정리 코드까지 같이 넣어야 합니다. 등록만 하고 정리하지 않으면 페이지를 이동해도 이벤트가 남아 예상하지 못한 동작이 생길 수 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: Composable은 재사용을 쉽게 해주지만, 너무 많은 책임을 한 함수에 몰아넣으면 이름은 `useSomething`인데 내부는 작은 서비스 객체처럼 복잡해집니다. 처음에는 좁은 책임으로 시작하는 것이 좋습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.

문제가 생겼을 때는 한 번에 모든 코드를 의심하지 말고, 입력값, 상태 변경 함수, 반환값 또는 라우트 이동 결과를 차례로 확인해보세요. 특히 Composable과 Router는 코드가 여러 파일에 나뉘기 쉬워서 "어디서 바뀌었는지"를 놓치기 쉽습니다. 브라우저 콘솔, Vue Devtools, Network 탭을 함께 보면 눈으로 보이지 않는 상태 변화와 요청 흐름을 훨씬 빠르게 확인할 수 있습니다.

## 실무에서 생각할 점

Composable은 중복을 줄이고 테스트 가능한 단위를 만들기 좋지만, 프로젝트 초반부터 모든 코드를 Composable로 빼려고 하면 파일 이동만 많아질 수 있습니다. 같은 로직이 두세 번 반복되고, 그 로직이 Vue 상태나 생명주기와 연결되어 있다는 신호가 보일 때 분리하면 더 안전합니다.

실무에서는 정답 패턴을 한 번에 고르는 것보다 현재 팀과 프로젝트의 복잡도에 맞는 선택을 하는 것이 중요합니다. 작은 화면에서는 단순한 코드가 가장 좋은 코드일 수 있고, 여러 화면에서 같은 문제가 반복되기 시작하면 그때 분리하는 편이 더 안전할 수 있습니다. 반대로 인증, 결제, 관리자 권한처럼 실수의 영향이 큰 영역은 초반부터 책임 경계를 조금 더 엄격하게 잡는 것이 좋습니다.

## 디버깅할 때 확인할 순서

1. 현재 값이 어디에서 만들어지는지 확인합니다.
2. 그 값을 바꾸는 함수가 어느 파일에 있는지 확인합니다.
3. 값이 바뀐 뒤 화면이 어떤 조건으로 다시 그려지는지 확인합니다.
4. 비동기 요청이 있다면 loading, success, error 상태가 모두 처리되는지 확인합니다.
5. 라우터 이동이 있다면 URL, route params, query, guard 결과를 차례로 확인합니다.

이 순서를 습관처럼 적용하면 문제를 만났을 때 훨씬 덜 흔들립니다. 특히 입문 단계에서는 코드를 많이 아는 것보다, 문제가 생겼을 때 확인할 순서를 알고 있는 것이 더 큰 힘이 됩니다.

## 오늘 내용 정리

이번 글에서는 'Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?' 주제를 중심으로 Vue 프로젝트에서 자주 만나는 구조 문제를 살펴봤습니다. 핵심은 코드를 예쁘게 나누는 것이 아니라, 상태와 책임이 어디에 있는지 설명할 수 있게 만드는 것입니다. 예제 코드를 그대로 따라 작성한 뒤에는 이름을 바꿔보거나, 실패 상황을 일부러 만들어보거나, 조건을 하나 추가해보면서 흐름을 직접 확인해보시면 좋습니다.

다음 글로 넘어가기 전에 오늘 예제에서 "이 코드는 컴포넌트가 알아야 할 일인가, 아니면 별도 함수나 라우터 설정이 맡아야 할 일인가"를 한 번 말로 정리해보세요. 이 질문에 답하는 연습이 쌓이면 Vue 프로젝트가 커졌을 때도 무작정 파일을 늘리거나 한 파일에 모두 몰아넣는 실수를 줄일 수 있습니다.

## 공식 문서와 같이 확인하면 좋은 부분

- [Vue 공식 문서 - Composables](https://vuejs.org/guide/reusability/composables.html)
- [Vue 공식 문서 - Lifecycle Hooks](https://vuejs.org/guide/essentials/lifecycle.html)

공식 문서는 문법의 기준점입니다. 블로그 글로 흐름을 잡고, 공식 문서로 정확한 옵션과 예외를 확인하면 오래된 예제나 다른 스타일의 코드를 만났을 때도 덜 흔들립니다. 특히 Vue Router와 Composable 관련 API는 작성 방식이 다양해 보일 수 있으므로, 현재 공식 문서를 기준으로 다시 확인하는 습관을 추천드립니다.

## 이어서 읽어보시면 좋습니다

### [Vue 입문 60: 재사용 컴포넌트에서 접근성과 사용성을 함께 고려하기](/vue/intro-60-reusable-component-accessibility)

이전 글에서 다룬 흐름을 다시 확인하는 용도로 좋습니다. 지금 글은 앞 글에서 만든 개념을 한 단계 확장하기 때문에, 예제의 전제가 갑자기 빠르게 느껴진다면 바로 앞 글로 돌아가 데이터가 어디서 시작되고 어떤 함수가 상태를 바꾸는지 다시 짚어보시면 좋습니다. 시리즈 글은 작은 기준이 계속 누적되는 구조라서, 이전 글을 다시 읽는 것이 오히려 가장 빠른 복습이 될 때가 많습니다.

### [Vue 입문 62: useCounter로 가장 작은 Composable 만들어보기](/vue/intro-62-use-counter-composable)

다음 글은 지금 배운 기준을 바로 이어서 적용하는 내용입니다. 현재 글에서 개념의 필요성을 이해했다면 다음 글에서는 그 개념이 다른 상황에서 어떻게 변형되는지 볼 수 있습니다. 이렇게 바로 다음 예제로 넘어가면 문법을 외우는 느낌보다 실제 프로젝트에서 판단 기준을 넓혀가는 느낌으로 학습할 수 있습니다.

### [Vue 입문 41: Options API와 Composition API를 비교해서 이해하기](/vue/intro-41-options-vs-composition-api)

Composition API의 기본 관점이 아직 살짝 흔들린다면 이 글을 함께 읽어보시면 좋습니다. Composable은 결국 Composition API 위에서 만들어지는 패턴이기 때문에, setup 안에서 상태와 함수를 어떻게 배치하는지 이해하고 있으면 지금 글의 예제가 훨씬 자연스럽게 보입니다. 특히 Options API와 비교해보면 왜 로직을 함수 단위로 묶기 쉬운지 감이 더 잘 잡힙니다.

Vue 입문 시리즈는 한 편씩 따로 읽어도 되지만, 가능하면 순서대로 따라가시는 것을 추천드립니다. 앞에서 만든 작은 감각이 뒤의 문법을 이해하는 발판이 되기 때문입니다. 지금 단계에서 중요한 것은 빠르게 많이 아는 것보다, 상태 흐름과 화면 이동 기준을 천천히 몸에 익히는 것입니다.