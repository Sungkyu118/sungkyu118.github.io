---
layout: post
title: "Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제"
date: 2026-07-05 07:05:00 +0900
category: Vue
permalink: /vue/intro-65-composable-lifecycle-dependency
description: "Vue Composable에서 onMounted, onUnmounted 같은 생명주기 훅을 사용할 때 호출 위치와 정리 책임을 어떻게 생각해야 하는지 설명합니다."
image:
  path: "/assets/img/og/vue-series-cover.svg"
  alt: "Vue 입문 시리즈 대표 이미지"
tags: [Vue, Composable, 생명주기, onUnmounted]
---

# Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제

> Composable이 생명주기 훅을 사용할 때 반드시 setup 흐름 안에서 호출되어야 한다는 점과 정리 책임을 배웁니다.
>
> 이전 글: [Vue 입문 64: Composable 안에서 watch를 사용할 때 주의할 점](/vue/intro-64-watch-inside-composable-cautions)
> 다음 글: [Vue 입문 66: API 호출 로직을 Composable로 분리하는 기준](/vue/intro-66-api-logic-composable-boundary)
> 함께 보면 좋은 글:
> - [Vue 입문 64: Composable 안에서 watch를 사용할 때 주의할 점](/vue/intro-64-watch-inside-composable-cautions)
> - [Vue 입문 66: API 호출 로직을 Composable로 분리하는 기준](/vue/intro-66-api-logic-composable-boundary)
> - [Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?](/vue/intro-61-what-is-composable)

Composable은 함수이지만, 모든 함수처럼 아무 곳에서나 호출해도 되는 것은 아닙니다. 내부에서 `onMounted`, `onUnmounted`, `provide`, `inject` 같은 Vue 컨텍스트에 의존하는 API를 사용한다면 반드시 컴포넌트의 setup 흐름 안에서 호출되어야 합니다. 이 차이를 모르고 일반 함수처럼 다루면 경고가 나거나 정리 코드가 실행되지 않을 수 있습니다.

예를 들어 타이머를 관리하는 `useInterval`을 만든다고 해보겠습니다. 컴포넌트가 화면에 있을 때만 타이머가 돌고, 컴포넌트가 사라지면 타이머를 멈춰야 합니다. 이때 `onUnmounted`를 Composable 안에서 사용하면 정리 책임을 Composable이 함께 가져갈 수 있어서 편리합니다.

다만 이 편리함은 호출 위치에 대한 규칙을 동반합니다. setup 밖, 일반 이벤트 핸들러 안, store 파일의 최상단 같은 곳에서 생명주기 의존 Composable을 호출하면 현재 컴포넌트 인스턴스가 없어서 의도대로 동작하지 않을 수 있습니다. 그래서 Composable이 생명주기에 의존하는지 문서화하거나 이름과 사용 예시를 분명히 해두는 것이 좋습니다.

## 이번 글에서 먼저 잡을 관점

이번 Composable 구간에서는 "코드를 함수로 빼는 것"보다 "상태와 동작의 책임을 어디까지 묶을 것인가"를 더 중요하게 보겠습니다. Composable은 반복 코드를 줄이는 도구이기도 하지만, 화면 컴포넌트가 너무 많은 일을 떠안지 않게 도와주는 설계 단위이기도 합니다.

읽으실 때는 예제의 파일 이름보다 데이터 흐름을 먼저 따라가 보시면 좋습니다. 어떤 값이 `ref`로 만들어지는지, 어떤 함수가 상태를 바꾸는지, 컴포넌트는 무엇을 몰라도 되는지 확인해보면 Composable의 장점과 위험이 훨씬 선명하게 보입니다.

지금 단계에서는 완벽한 구조를 외우려고 하기보다, 왜 이 코드가 컴포넌트 안에 있으면 불편해지는지, 분리하면 어떤 장점이 생기는지, 반대로 너무 분리하면 어떤 비용이 생기는지를 함께 생각해보시면 좋습니다. 기술을 배우는 속도보다 판단 기준을 만드는 속도가 더 중요할 때가 많습니다.

## 작은 실습으로 확인해보기

아래 예제는 일정 시간마다 숫자를 증가시키는 `useIntervalCounter`입니다. 타이머 생성과 정리를 Composable 안에 함께 넣어 컴포넌트가 정리 코드를 잊지 않도록 만듭니다.

{% raw %}
~~~vue
// composables/useIntervalCounter.js
import { ref, onMounted, onUnmounted } from 'vue'

export function useIntervalCounter(delay = 1000) {
  const count = ref(0)
  let timerId = null

  function start() {
    if (timerId !== null) {
      return
    }

    timerId = window.setInterval(() => {
      count.value += 1
    }, delay)
  }

  function stop() {
    if (timerId === null) {
      return
    }

    window.clearInterval(timerId)
    timerId = null
  }

  onMounted(start)
  onUnmounted(stop)

  return {
    count,
    start,
    stop
  }
}

// TimerCard.vue
<script setup>
import { useIntervalCounter } from '@/composables/useIntervalCounter'

const { count, stop, start } = useIntervalCounter(1000)
</script>

<template>
  <article>
    <p>화면에 머문 시간: {{ count }}초</p>
    <button type="button" @click="stop">멈추기</button>
    <button type="button" @click="start">다시 시작</button>
  </article>
</template>
~~~
{% endraw %}

## 코드가 움직이는 순서

`timerId`는 반응형으로 화면에 보여줄 값이 아니므로 일반 변수로 두었습니다. `count`만 화면에 표시되어야 하므로 `ref`입니다. `onMounted`에서 시작하고 `onUnmounted`에서 멈추기 때문에 컴포넌트가 사라진 뒤에도 타이머가 계속 도는 문제를 줄일 수 있습니다. `start` 안에서 이미 타이머가 있으면 다시 만들지 않도록 막은 것도 중요합니다.

예제를 따라 작성할 때는 먼저 상태가 만들어지는 위치를 찾고, 그다음 그 상태를 바꾸는 함수를 찾고, 마지막으로 템플릿에서 어떤 조건으로 화면이 바뀌는지 확인해보세요. Vue 코드는 겉으로는 HTML과 JavaScript가 섞여 보이지만, 실제로는 "상태가 바뀌고, 계산값이 갱신되고, 템플릿이 다시 그려지는 흐름"을 반복합니다. 이 순서를 말로 설명할 수 있으면 문법을 조금 잊어도 다시 찾아가며 구현할 수 있습니다.

## 자주 만나는 에러와 유의사항

- **확인 포인트**: 생명주기 훅을 사용하는 Composable은 setup 흐름 안에서 호출해야 합니다. 조건문 안이나 나중에 실행되는 콜백 안에서 호출하면 현재 인스턴스가 없거나 호출 순서가 꼬일 수 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: 타이머, 이벤트 리스너, 구독, 소켓 연결은 만들었으면 반드시 정리해야 합니다. 화면에서는 보이지 않지만 메모리와 네트워크를 계속 사용할 수 있습니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.
- **확인 포인트**: Composable 내부의 일반 변수는 컴포넌트마다 새로 만들어지는지, 모듈 전체에서 공유되는지 확인해야 합니다. 함수 안에 있으면 호출마다 새 값이고, 함수 밖에 있으면 공유될 가능성이 큽니다. 입문 예제에서는 작게 보일 수 있지만, 실제 프로젝트에서는 원인을 찾기 어려운 버그나 사용자 경험 문제로 이어질 수 있습니다. 코드를 작성한 뒤에는 이 문제가 어떤 상황에서 나타날 수 있는지 직접 한 번 재현해보는 습관을 들이면 좋습니다.

문제가 생겼을 때는 한 번에 모든 코드를 의심하지 말고, 입력값, 상태 변경 함수, 반환값 또는 라우트 이동 결과를 차례로 확인해보세요. 특히 Composable과 Router는 코드가 여러 파일에 나뉘기 쉬워서 "어디서 바뀌었는지"를 놓치기 쉽습니다. 브라우저 콘솔, Vue Devtools, Network 탭을 함께 보면 눈으로 보이지 않는 상태 변화와 요청 흐름을 훨씬 빠르게 확인할 수 있습니다.

## 실무에서 생각할 점

생명주기까지 포함한 Composable은 호출하는 쪽을 단순하게 만들어줍니다. 대신 Composable이 Vue 컴포넌트 환경에 강하게 묶이므로 일반 JavaScript 테스트나 store 레벨 재사용은 어려워질 수 있습니다. 순수 로직과 생명주기 연결 코드를 분리해두면 테스트와 재사용성이 더 좋아집니다.

실무에서는 정답 패턴을 한 번에 고르는 것보다 현재 팀과 프로젝트의 복잡도에 맞는 선택을 하는 것이 중요합니다. 작은 화면에서는 단순한 코드가 가장 좋은 코드일 수 있고, 여러 화면에서 같은 문제가 반복되기 시작하면 그때 분리하는 편이 더 안전할 수 있습니다. 반대로 인증, 결제, 관리자 권한처럼 실수의 영향이 큰 영역은 초반부터 책임 경계를 조금 더 엄격하게 잡는 것이 좋습니다.

## 디버깅할 때 확인할 순서

1. 현재 값이 어디에서 만들어지는지 확인합니다.
2. 그 값을 바꾸는 함수가 어느 파일에 있는지 확인합니다.
3. 값이 바뀐 뒤 화면이 어떤 조건으로 다시 그려지는지 확인합니다.
4. 비동기 요청이 있다면 loading, success, error 상태가 모두 처리되는지 확인합니다.
5. 라우터 이동이 있다면 URL, route params, query, guard 결과를 차례로 확인합니다.

이 순서를 습관처럼 적용하면 문제를 만났을 때 훨씬 덜 흔들립니다. 특히 입문 단계에서는 코드를 많이 아는 것보다, 문제가 생겼을 때 확인할 순서를 알고 있는 것이 더 큰 힘이 됩니다.

## 오늘 내용 정리

이번 글에서는 'Vue 입문 65: Composable이 컴포넌트 생명주기에 의존할 때 생기는 문제' 주제를 중심으로 Vue 프로젝트에서 자주 만나는 구조 문제를 살펴봤습니다. 핵심은 코드를 예쁘게 나누는 것이 아니라, 상태와 책임이 어디에 있는지 설명할 수 있게 만드는 것입니다. 예제 코드를 그대로 따라 작성한 뒤에는 이름을 바꿔보거나, 실패 상황을 일부러 만들어보거나, 조건을 하나 추가해보면서 흐름을 직접 확인해보시면 좋습니다.

다음 글로 넘어가기 전에 오늘 예제에서 "이 코드는 컴포넌트가 알아야 할 일인가, 아니면 별도 함수나 라우터 설정이 맡아야 할 일인가"를 한 번 말로 정리해보세요. 이 질문에 답하는 연습이 쌓이면 Vue 프로젝트가 커졌을 때도 무작정 파일을 늘리거나 한 파일에 모두 몰아넣는 실수를 줄일 수 있습니다.

## 공식 문서와 같이 확인하면 좋은 부분

- [Vue 공식 문서 - Lifecycle Hooks](https://vuejs.org/guide/essentials/lifecycle.html)
- [Vue 공식 문서 - Composables](https://vuejs.org/guide/reusability/composables.html)

공식 문서는 문법의 기준점입니다. 블로그 글로 흐름을 잡고, 공식 문서로 정확한 옵션과 예외를 확인하면 오래된 예제나 다른 스타일의 코드를 만났을 때도 덜 흔들립니다. 특히 Vue Router와 Composable 관련 API는 작성 방식이 다양해 보일 수 있으므로, 현재 공식 문서를 기준으로 다시 확인하는 습관을 추천드립니다.

## 이어서 읽어보시면 좋습니다

### [Vue 입문 64: Composable 안에서 watch를 사용할 때 주의할 점](/vue/intro-64-watch-inside-composable-cautions)

이전 글에서 다룬 흐름을 다시 확인하는 용도로 좋습니다. 지금 글은 앞 글에서 만든 개념을 한 단계 확장하기 때문에, 예제의 전제가 갑자기 빠르게 느껴진다면 바로 앞 글로 돌아가 데이터가 어디서 시작되고 어떤 함수가 상태를 바꾸는지 다시 짚어보시면 좋습니다. 시리즈 글은 작은 기준이 계속 누적되는 구조라서, 이전 글을 다시 읽는 것이 오히려 가장 빠른 복습이 될 때가 많습니다.

### [Vue 입문 66: API 호출 로직을 Composable로 분리하는 기준](/vue/intro-66-api-logic-composable-boundary)

다음 글은 지금 배운 기준을 바로 이어서 적용하는 내용입니다. 현재 글에서 개념의 필요성을 이해했다면 다음 글에서는 그 개념이 다른 상황에서 어떻게 변형되는지 볼 수 있습니다. 이렇게 바로 다음 예제로 넘어가면 문법을 외우는 느낌보다 실제 프로젝트에서 판단 기준을 넓혀가는 느낌으로 학습할 수 있습니다.

### [Vue 입문 61: Composable은 단순 유틸 함수와 무엇이 다를까요?](/vue/intro-61-what-is-composable)

Composable 구간의 출발점으로 돌아가고 싶을 때 좋은 글입니다. 지금 글에서 다루는 예제가 조금 복잡하게 느껴진다면, 먼저 Composable이 일반 유틸 함수와 어떻게 다른지 다시 정리해보는 편이 좋습니다. 반응형 상태, 생명주기, 반환값 설계를 구분하면 뒤의 API 호출과 폼 검증 예제도 훨씬 덜 부담스럽게 읽힙니다.

Vue 입문 시리즈는 한 편씩 따로 읽어도 되지만, 가능하면 순서대로 따라가시는 것을 추천드립니다. 앞에서 만든 작은 감각이 뒤의 문법을 이해하는 발판이 되기 때문입니다. 지금 단계에서 중요한 것은 빠르게 많이 아는 것보다, 상태 흐름과 화면 이동 기준을 천천히 몸에 익히는 것입니다.
