---
layout: post
title: "Vue 입문 74: 동적 라우트 파라미터로 상세 페이지 만들기"
date: 2026-07-05 07:14:00 +0900
category: Vue
permalink: /vue/intro-74-dynamic-route-params-detail-page
description: "Vue Router의 동적 라우트 파라미터를 사용해 목록에서 상세 페이지로 이동하고 useRoute로 id를 읽는 방법을 설명합니다."
image:
  path: "/assets/img/og/vue-series-cover.svg"
  alt: "Vue 입문 시리즈 대표 이미지"
tags: [Vue, Vue Router, 동적 라우트, params]
---

# Vue 입문 74: 동적 라우트 파라미터로 상세 페이지 만들기

> 여러 상세 페이지를 하나의 컴포넌트로 처리하기 위해 path에 `:id` 같은 동적 세그먼트를 사용하는 법을 배웁니다.
>
> 이전 글: [Vue 입문 73: router-view와 router-link가 하는 일](/vue/intro-73-router-view-router-link)
> 다음 글: [Vue 입문 75: query string으로 검색 조건 유지하기](/vue/intro-75-query-string-search-state)
> 함께 보면 좋은 글:
> - [Vue 입문 73: router-view와 router-link가 하는 일](/vue/intro-73-router-view-router-link)
> - [Vue 입문 75: query string으로 검색 조건 유지하기](/vue/intro-75-query-string-search-state)
> - [Vue 입문 71: SPA에서 라우터가 필요한 이유](/vue/intro-71-why-spa-needs-router)

상품 상세 페이지를 생각해보겠습니다. 상품이 100개 있다고 해서 상세 컴포넌트를 100개 만들지는 않습니다. `/products/1`, `/products/2`, `/products/3`처럼 주소의 일부만 달라지고, 같은 상세 컴포넌트가 해당 id를 기준으로 데이터를 불러오는 구조가 자연스럽습니다.

Vue Router에서는 이런 주소 일부를 동적 라우트 파라미터라고 부릅니다. 라우트 path에 `:id`처럼 콜론으로 시작하는 이름을 넣으면, 실제 URL의 해당 위치 값이 `route.params.id`로 들어옵니다.

주의할 점은 같은 컴포넌트가 재사용될 수 있다는 것입니다. `/products/1`에서 `/products/2`로 이동할 때 컴포넌트가 완전히 새로 만들어지지 않고 같은 인스턴스가 재사용될 수 있습니다. 그래서 param 변화에 따라 데이터를 다시 불러오는 처리가 필요할 수 있습니다.

## 이번 글에서 먼저 잡을 관점

이번 Router 구간에서는 "주소가 바뀌면 컴포넌트가 바뀐다"라는 단순한 설명을 넘어, URL이 사용자 경험과 앱 구조를 어떻게 지탱하는지 살펴보겠습니다. 라우터는 페이지 이동 도구이기도 하지만, 사용자가 현재 어디에 있는지 설명하는 약속이기도 합니다.

예제를 읽을 때는 라우트 설정, 화면 컴포넌트, 브라우저 히스토리, 서버 배포 설정이 서로 떨어진 주제가 아니라는 점을 기억해보시면 좋습니다. 입문 단계에서 이 연결을 잡아두면 나중에 인증, 권한, API 연동을 붙일 때 훨씬 덜 흔들립니다.

지금 단계에서는 완벽한 구조를 외우려고 하기보다, 왜 이 코드가 컴포넌트 안에 있으면 불편해지는지, 분리하면 어떤 장점이 생기는지, 반대로 너무 분리하면 어떤 비용이 생기는지를 함께 생각해보시면 좋습니다. 기술을 배우는 속도보다 판단 기준을 만드는 속도가 더 중요할 때가 많습니다.

## 작은 실습으로 확인해보기

아래 예제는 상품 목록에서 상세 페이지로 이동하고, 상세 페이지에서 route param을 읽어 데이터를 조회하는 흐름입니다.

{% raw %}
~~~vue
// router/index.js
const routes = [
  {
    path: '/products',
    name: 'products',
    component: () => import('@/views/ProductListView.vue')
  },
  {
    path: '/products/:id',
    name: 'product-detail',
    component: () => import('@/views/ProductDetailView.vue')
  }
]

// ProductListView.vue
<template>
  <ul>
    <li v-for="product in products" :key="product.id">
      <RouterLink :to="{ name: 'product-detail', params: { id: product.id } }">
        {{ product.name }}
      </RouterLink>
    </li>
  </ul>
</template>

// ProductDetailView.vue
<script setup>
import { ref, watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()
const product = ref(null)

async function loadProduct(id) {
  const response = await fetch(`/api/products/${id}`)
  product.value = await response.json()
}

watch(
  () => route.params.id,
  (id) => {
    loadProduct(id)
  },
  { immediate: true }
)
</script>
~~~
{% endraw %}

## 코드가 움직이는 순서

`/products/:id`에서 `:id`가 동적 부분입니다. 실제 URL이 `/products/10`이면 `route.params.id`는 `"10"`이 됩니다. 문자열로 들어오는 경우가 많으므로 숫자가 필요하다면 변환해야 합니다. 상세 컴포넌트에서는 `watch`로 param 변화를 감시해 같은 컴포넌트 안에서 다른 id로 이동해도 데이터를 다시 불러오게 했습니다.

예제를 따라 작성할 때는 먼저 상태가 만들어지는 위치를 찾고, 그다음 그 상태를 바꾸는 함수를 찾고, 마지막으로 템플릿에서 어떤 조건으로 화면이 바뀌는지 확인해보세요. Vue 코드는 겉으로는 HTML과 JavaScript가 섞여 보이지만, 실제로는 "상태가 바뀌고, 계산값이 갱신되고, 템플릿이 다시 그려지는 흐름"을 반복합니다. 이 순서를 말로 설명할 수 있으면 문법을 조금 잊어도 다시 찾아가며 구현할 수 있습니다.

## 자주 만나는 에러와 유의사항

- **확인 포인트**: route param은 보통 문자열입니다. API가 숫자를 기대한다면 `Number(route.params.id)`처럼 변환하고 NaN 처리까지 고려해야 합니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: param만 바뀌는 이동에서는 컴포넌트가 재사용될 수 있습니다. `onMounted`만 사용하면 처음 한 번만 로드되고 이후 id 변경을 놓칠 수 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: 사용자가 존재하지 않는 id로 직접 접근할 수 있습니다. 프론트에서 404 UI를 보여주고, 서버에서도 데이터 존재 여부를 정확히 응답해야 합니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.

문제가 생겼을 때는 한 번에 모든 코드를 의심하지 말고, 입력값, 상태 변경 함수, 반환값 또는 라우트 이동 결과를 차례로 확인해보세요. 특히 Composable과 Router는 코드가 여러 파일에 나뉘기 쉬워서 "어디서 바뀌었는지"를 놓치기 쉽습니다. 브라우저 콘솔, Vue Devtools, Network 탭을 함께 보면 눈으로 보이지 않는 상태 변화와 요청 흐름을 훨씬 빠르게 확인할 수 있습니다.

## 실무에서 생각할 점

동적 라우트는 상세 페이지를 만들 때 매우 자연스럽습니다. 다만 param이 많아지고 중첩이 깊어지면 URL만 보고 의미를 알기 어려워질 수 있으므로, path 구조는 사용자와 개발자가 모두 이해하기 쉬운 형태로 유지하는 것이 좋습니다.

실무에서는 정답 패턴을 한 번에 고르는 것보다 현재 팀과 프로젝트의 복잡도에 맞는 선택을 하는 것이 중요합니다. 작은 화면에서는 단순한 코드가 가장 좋은 코드일 수 있고, 여러 화면에서 같은 문제가 반복되기 시작하면 그때 분리하는 편이 더 안전할 수 있습니다. 반대로 인증, 결제, 관리자 권한처럼 실수의 영향이 큰 영역은 초반부터 책임 경계를 조금 더 엄격하게 잡는 것이 좋습니다.

## 디버깅할 때 확인할 순서

1. 현재 값이 어디에서 만들어지는지 확인합니다.
2. 그 값을 바꾸는 함수가 어느 파일에 있는지 확인합니다.
3. 값이 바뀐 뒤 화면이 어떤 조건으로 다시 그려지는지 확인합니다.
4. 비동기 요청이 있다면 loading, success, error 상태가 모두 처리되는지 확인합니다.
5. 라우터 이동이 있다면 URL, route params, query, guard 결과를 차례로 확인합니다.

이 순서를 습관처럼 적용하면 문제를 만났을 때 훨씬 덜 흔들립니다. 특히 입문 단계에서는 코드를 많이 아는 것보다, 문제가 생겼을 때 확인할 순서를 알고 있는 것이 더 큰 힘이 됩니다.

## 오늘 내용 정리

이번 글에서는 'Vue 입문 74: 동적 라우트 파라미터로 상세 페이지 만들기' 주제를 중심으로 Vue 프로젝트에서 자주 만나는 구조 문제를 살펴봤습니다. 핵심은 코드를 예쁘게 나누는 것이 아니라, 상태와 책임이 어디에 있는지 설명할 수 있게 만드는 것입니다. 예제 코드를 그대로 따라 작성한 뒤에는 이름을 바꿔보거나, 실패 상황을 일부러 만들어보거나, 조건을 하나 추가해보면서 흐름을 직접 확인해보시면 좋습니다.

다음 글로 넘어가기 전에 오늘 예제에서 "이 코드는 컴포넌트가 알아야 할 일인가, 아니면 별도 함수나 라우터 설정이 맡아야 할 일인가"를 한 번 말로 정리해보세요. 이 질문에 답하는 연습이 쌓이면 Vue 프로젝트가 커졌을 때도 무작정 파일을 늘리거나 한 파일에 모두 몰아넣는 실수를 줄일 수 있습니다.

## 공식 문서와 같이 확인하면 좋은 부분

- [Vue Router 공식 문서 - Dynamic Route Matching](https://router.vuejs.org/guide/essentials/dynamic-matching.html)
- [Vue Router 공식 문서 - Composition API](https://router.vuejs.org/guide/advanced/composition-api.html)

공식 문서는 문법의 기준점입니다. 블로그 글로 흐름을 잡고, 공식 문서로 정확한 옵션과 예외를 확인하면 오래된 예제나 다른 스타일의 코드를 만났을 때도 덜 흔들립니다. 특히 Vue Router와 Composable 관련 API는 작성 방식이 다양해 보일 수 있으므로, 현재 공식 문서를 기준으로 다시 확인하는 습관을 추천드립니다.

## 이어서 읽어보시면 좋습니다

### [Vue 입문 73: router-view와 router-link가 하는 일](/vue/intro-73-router-view-router-link)

이전 글에서 다룬 흐름을 다시 확인하는 용도로 좋습니다. 지금 글은 앞 글에서 만든 개념을 한 단계 확장하기 때문에, 예제의 전제가 갑자기 빠르게 느껴진다면 바로 앞 글로 돌아가 데이터가 어디서 시작되고 어떤 함수가 상태를 바꾸는지 다시 짚어보시면 좋습니다. 시리즈 글은 작은 기준이 계속 누적되는 구조라서, 이전 글을 다시 읽는 것이 오히려 가장 빠른 복습이 될 때가 많습니다.

### [Vue 입문 75: query string으로 검색 조건 유지하기](/vue/intro-75-query-string-search-state)

다음 글은 지금 배운 기준을 바로 이어서 적용하는 내용입니다. 현재 글에서 개념의 필요성을 이해했다면 다음 글에서는 그 개념이 다른 상황에서 어떻게 변형되는지 볼 수 있습니다. 이렇게 바로 다음 예제로 넘어가면 문법을 외우는 느낌보다 실제 프로젝트에서 판단 기준을 넓혀가는 느낌으로 학습할 수 있습니다.

### [Vue 입문 71: SPA에서 라우터가 필요한 이유](/vue/intro-71-why-spa-needs-router)

Router 구간의 큰 그림을 다시 잡고 싶을 때 읽기 좋습니다. 세부 기능을 배우다 보면 params, query, guard 같은 단어가 따로 노는 것처럼 보일 수 있는데, 71번 글은 라우터가 왜 필요한지와 URL이 화면 상태를 어떻게 설명하는지부터 다시 정리해줍니다. 현재 글의 세부 기능이 어디에 쓰이는지 감을 되찾는 데 도움이 됩니다.

Vue 입문 시리즈는 한 편씩 따로 읽어도 되지만, 가능하면 순서대로 따라가시는 것을 추천드립니다. 앞에서 만든 작은 감각이 뒤의 문법을 이해하는 발판이 되기 때문입니다. 지금 단계에서 중요한 것은 빠르게 많이 아는 것보다, 상태 흐름과 화면 이동 기준을 천천히 몸에 익히는 것입니다.