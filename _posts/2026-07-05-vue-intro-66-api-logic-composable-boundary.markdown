---
layout: post
title: "Vue 입문 66: API 호출 로직을 Composable로 분리하는 기준"
date: 2026-07-05 07:06:00 +0900
category: Vue
permalink: /vue/intro-66-api-logic-composable-boundary
description: "Vue 컴포넌트 안의 API 호출 코드를 언제 Composable로 분리하면 좋은지, 화면 책임과 데이터 요청 책임을 나누는 기준을 설명합니다."
image:
  path: "/assets/img/og/vue-series-cover.svg"
  alt: "Vue 입문 시리즈 대표 이미지"
tags: [Vue, Composable, API, 설계]
---

# Vue 입문 66: API 호출 로직을 Composable로 분리하는 기준

> API 호출을 무조건 빼는 것이 아니라 반복, 상태 전환, 화면 책임을 기준으로 분리하는 법을 배웁니다.
>
> 이전 글: [Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제](/vue/intro-65-composable-lifecycle-dependency)
> 다음 글: [Vue 입문 67: 폼 검증 로직을 Composable로 빼는 실습](/vue/intro-67-form-validation-composable)
> 함께 보면 좋은 글:
> - [Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제](/vue/intro-65-composable-lifecycle-dependency)
> - [Vue 입문 67: 폼 검증 로직을 Composable로 빼는 실습](/vue/intro-67-form-validation-composable)
> - [Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?](/vue/intro-61-what-is-composable)

API 호출 코드를 보자마자 무조건 Composable로 분리해야 하는 것은 아닙니다. 한 화면에서만 쓰이고, 코드가 아주 짧고, 별도의 재사용 가능성이 없다면 컴포넌트 안에 있어도 괜찮습니다. 오히려 너무 빨리 분리하면 파일을 오가며 읽어야 해서 전체 흐름이 더 어려워질 수 있습니다.

하지만 API 호출 주변에 loading, error, retry, pagination, 검색 조건, 인증 헤더 같은 상태가 붙기 시작하면 이야기가 달라집니다. 화면 컴포넌트가 UI 배치와 사용자 이벤트에 집중하지 못하고 서버 통신의 세부사항까지 모두 떠안게 됩니다. 이때 Composable은 책임을 나누는 좋은 경계가 됩니다.

분리 기준은 "재사용되는가" 하나만이 아닙니다. 재사용되지 않아도 복잡한 화면에서 데이터 요청 책임을 따로 설명할 수 있다면 Composable로 뺄 수 있습니다. 반대로 여러 곳에서 쓰이더라도 각 화면의 요구사항이 너무 다르면 억지로 하나의 Composable에 모두 넣지 않는 편이 좋습니다.

## 이번 글에서 먼저 잡을 관점

이번 Composable 구간에서는 "코드를 함수로 빼는 것"보다 "상태와 동작의 책임을 어디까지 묶을 것인가"를 더 중요하게 보겠습니다. Composable은 반복 코드를 줄이는 도구이기도 하지만, 화면 컴포넌트가 너무 많은 일을 떠안지 않게 도와주는 설계 단위이기도 합니다.

읽으실 때는 예제의 파일 이름보다 데이터 흐름을 먼저 따라가 보시면 좋습니다. 어떤 값이 `ref`로 만들어지는지, 어떤 함수가 상태를 바꾸는지, 컴포넌트는 무엇을 몰라도 되는지 확인해보면 Composable의 장점과 위험이 훨씬 선명하게 보입니다.

지금 단계에서는 완벽한 구조를 외우려고 하기보다, 왜 이 코드가 컴포넌트 안에 있으면 불편해지는지, 분리하면 어떤 장점이 생기는지, 반대로 너무 분리하면 어떤 비용이 생기는지를 함께 생각해보시면 좋습니다. 기술을 배우는 속도보다 판단 기준을 만드는 속도가 더 중요할 때가 많습니다.

## 작은 실습으로 확인해보기

아래 예제는 상품 목록 화면의 API 요청을 `useProducts`로 분리한 모습입니다. 컴포넌트는 검색어와 버튼, 목록 렌더링에 집중하고, 요청 상태 관리는 Composable이 담당합니다.

{% raw %}
~~~vue
// composables/useProducts.js
import { ref } from 'vue'

export function useProducts() {
  const products = ref([])
  const loading = ref(false)
  const errorMessage = ref('')

  async function loadProducts(params = {}) {
    loading.value = true
    errorMessage.value = ''

    const query = new URLSearchParams(params).toString()

    try {
      const response = await fetch(`/api/products?${query}`)

      if (!response.ok) {
        throw new Error('상품 목록을 불러오지 못했습니다.')
      }

      products.value = await response.json()
    } catch (error) {
      products.value = []
      errorMessage.value = '상품 목록을 불러오지 못했습니다. 잠시 후 다시 시도해주세요.'
    } finally {
      loading.value = false
    }
  }

  return {
    products,
    loading,
    errorMessage,
    loadProducts
  }
}

// ProductPage.vue
<script setup>
import { ref, onMounted } from 'vue'
import { useProducts } from '@/composables/useProducts'

const keyword = ref('')
const { products, loading, errorMessage, loadProducts } = useProducts()

onMounted(() => {
  loadProducts()
})

function search() {
  loadProducts({ keyword: keyword.value })
}
</script>
~~~
{% endraw %}

## 코드가 움직이는 순서

`ProductPage.vue`는 사용자가 검색어를 입력하고 검색 버튼을 누르는 흐름만 알고 있으면 됩니다. 실제 URL을 만드는 일, 응답 실패 처리, 목록 초기화, loading 전환은 `useProducts`가 담당합니다. 이렇게 나누면 나중에 상품 목록 API 응답 구조가 바뀌어도 화면 템플릿보다 Composable을 먼저 보면 됩니다.

예제를 따라 작성할 때는 먼저 상태가 만들어지는 위치를 찾고, 그다음 그 상태를 바꾸는 함수를 찾고, 마지막으로 템플릿에서 어떤 조건으로 화면이 바뀌는지 확인해보세요. Vue 코드는 겉으로는 HTML과 JavaScript가 섞여 보이지만, 실제로는 "상태가 바뀌고, 계산값이 갱신되고, 템플릿이 다시 그려지는 흐름"을 반복합니다. 이 순서를 말로 설명할 수 있으면 문법을 조금 잊어도 다시 찾아가며 구현할 수 있습니다.

## 자주 만나는 에러와 유의사항

- **확인 포인트**: Composable 이름이 너무 추상적이면 책임이 흐려집니다. `useApi`보다 `useProducts`, `useProductDetail`처럼 도메인이 보이는 이름이 읽기 쉬울 때가 많습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: 모든 에러를 같은 문구로 처리하면 디버깅 정보가 부족해질 수 있습니다. 사용자 메시지와 개발자 로그를 분리하는 습관이 필요합니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: 페이지네이션이나 정렬이 들어오면 params 구조가 금방 커질 수 있습니다. 문자열을 직접 이어붙이기보다 `URLSearchParams`처럼 안전한 도구를 사용하면 실수를 줄일 수 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.

문제가 생겼을 때는 한 번에 모든 코드를 의심하지 말고, 입력값, 상태 변경 함수, 반환값 또는 라우트 이동 결과를 차례로 확인해보세요. 특히 Composable과 Router는 코드가 여러 파일에 나뉘기 쉬워서 "어디서 바뀌었는지"를 놓치기 쉽습니다. 브라우저 콘솔, Vue Devtools, Network 탭을 함께 보면 눈으로 보이지 않는 상태 변화와 요청 흐름을 훨씬 빠르게 확인할 수 있습니다.

## 실무에서 생각할 점

API 로직을 Composable로 분리하면 화면은 깔끔해지지만, 데이터 계층이 너무 화면 중심으로 굳어질 수 있습니다. 여러 도메인에서 공유되는 통신 규칙은 별도 API 클라이언트로, 화면 상태와 연결된 loading/error는 Composable로 나누면 균형이 좋습니다.

실무에서는 정답 패턴을 한 번에 고르는 것보다 현재 팀과 프로젝트의 복잡도에 맞는 선택을 하는 것이 중요합니다. 작은 화면에서는 단순한 코드가 가장 좋은 코드일 수 있고, 여러 화면에서 같은 문제가 반복되기 시작하면 그때 분리하는 편이 더 안전할 수 있습니다. 반대로 인증, 결제, 관리자 권한처럼 실수의 영향이 큰 영역은 초반부터 책임 경계를 조금 더 엄격하게 잡는 것이 좋습니다.

## 디버깅할 때 확인할 순서

1. 현재 값이 어디에서 만들어지는지 확인합니다.
2. 그 값을 바꾸는 함수가 어느 파일에 있는지 확인합니다.
3. 값이 바뀐 뒤 화면이 어떤 조건으로 다시 그려지는지 확인합니다.
4. 비동기 요청이 있다면 loading, success, error 상태가 모두 처리되는지 확인합니다.
5. 라우터 이동이 있다면 URL, route params, query, guard 결과를 차례로 확인합니다.

이 순서를 습관처럼 적용하면 문제를 만났을 때 훨씬 덜 흔들립니다. 특히 입문 단계에서는 코드를 많이 아는 것보다, 문제가 생겼을 때 확인할 순서를 알고 있는 것이 더 큰 힘이 됩니다.

## 오늘 내용 정리

이번 글에서는 'Vue 입문 66: API 호출 로직을 Composable로 분리하는 기준' 주제를 중심으로 Vue 프로젝트에서 자주 만나는 구조 문제를 살펴봤습니다. 핵심은 코드를 예쁘게 나누는 것이 아니라, 상태와 책임이 어디에 있는지 설명할 수 있게 만드는 것입니다. 예제 코드를 그대로 따라 작성한 뒤에는 이름을 바꿔보거나, 실패 상황을 일부러 만들어보거나, 조건을 하나 추가해보면서 흐름을 직접 확인해보시면 좋습니다.

다음 글로 넘어가기 전에 오늘 예제에서 "이 코드는 컴포넌트가 알아야 할 일인가, 아니면 별도 함수나 라우터 설정이 맡아야 할 일인가"를 한 번 말로 정리해보세요. 이 질문에 답하는 연습이 쌓이면 Vue 프로젝트가 커졌을 때도 무작정 파일을 늘리거나 한 파일에 모두 몰아넣는 실수를 줄일 수 있습니다.

## 공식 문서와 같이 확인하면 좋은 부분

- [Vue 공식 문서 - Composables](https://vuejs.org/guide/reusability/composables.html)
- [Vue Router 공식 문서 - Data Fetching](https://router.vuejs.org/guide/advanced/data-fetching.html)

공식 문서는 문법의 기준점입니다. 블로그 글로 흐름을 잡고, 공식 문서로 정확한 옵션과 예외를 확인하면 오래된 예제나 다른 스타일의 코드를 만났을 때도 덜 흔들립니다. 특히 Vue Router와 Composable 관련 API는 작성 방식이 다양해 보일 수 있으므로, 현재 공식 문서를 기준으로 다시 확인하는 습관을 추천드립니다.

## 이어서 읽어보시면 좋습니다

### [Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제](/vue/intro-65-composable-lifecycle-dependency)

이전 글에서 다룬 흐름을 다시 확인하는 용도로 좋습니다. 지금 글은 앞 글에서 만든 개념을 한 단계 확장하기 때문에, 예제의 전제가 갑자기 빠르게 느껴진다면 바로 앞 글로 돌아가 데이터가 어디서 시작되고 어떤 함수가 상태를 바꾸는지 다시 짚어보시면 좋습니다. 시리즈 글은 작은 기준이 계속 누적되는 구조라서, 이전 글을 다시 읽는 것이 오히려 가장 빠른 복습이 될 때가 많습니다.

### [Vue 입문 67: 폼 검증 로직을 Composable로 빼는 실습](/vue/intro-67-form-validation-composable)

다음 글은 지금 배운 기준을 바로 이어서 적용하는 내용입니다. 현재 글에서 개념의 필요성을 이해했다면 다음 글에서는 그 개념이 다른 상황에서 어떻게 변형되는지 볼 수 있습니다. 이렇게 바로 다음 예제로 넘어가면 문법을 외우는 느낌보다 실제 프로젝트에서 판단 기준을 넓혀가는 느낌으로 학습할 수 있습니다.

### [Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?](/vue/intro-61-what-is-composable)

Composable 구간의 출발점으로 돌아가고 싶을 때 좋은 글입니다. 지금 글에서 다루는 예제가 조금 복잡하게 느껴진다면, 먼저 Composable이 일반 유틸 함수와 어떻게 다른지 다시 정리해보는 편이 좋습니다. 반응형 상태, 생명주기, 반환값 설계를 구분하면 뒤의 API 호출과 폼 검증 예제도 훨씬 덜 부담스럽게 읽힙니다.

Vue 입문 시리즈는 한 편씩 따로 읽어도 되지만, 가능하면 순서대로 따라가시는 것을 추천드립니다. 앞에서 만든 작은 감각이 뒤의 문법을 이해하는 발판이 되기 때문입니다. 지금 단계에서 중요한 것은 빠르게 많이 아는 것보다, 상태 흐름과 화면 이동 기준을 천천히 몸에 익히는 것입니다.