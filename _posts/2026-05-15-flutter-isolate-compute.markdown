---
layout: post
title: "[Flutter] compute/Isolate로 메인 스레드 프리징 막기 (무거운 작업 분리)"
date: 2026-05-15 00:20:00 +0900
category: Flutter
permalink: /flutter/flutter-isolate-compute
---

# [Flutter] compute/Isolate로 메인 스레드 프리징 막기 (무거운 작업 분리)

Flutter 앱이 "버튼 누르면 잠깐 멈춤", "스크롤 중 끊김" 같은 느낌이 들 때, 원인 중 하나는 **메인 Isolate(UI 스레드)** 에서 무거운 연산을 돌리는 겁니다.

이 글은 복잡한 구조 없이도 바로 적용 가능한 `compute`와 Isolate 기본 사용법을 정리합니다.

## 1) 언제 Isolate가 필요한가

다음 같은 작업은 메인 isolate를 잠그기 쉽습니다.

- 큰 JSON 파싱/가공
- 이미지 리사이즈/압축
- 대량 데이터 정렬/필터링
- 암호화/해시 등 CPU 연산

반대로 네트워크 요청은 "대기"가 대부분이라 isolate 분리만으로 체감이 크게 개선되지 않는 경우가 많습니다(물론 응답 처리에서 큰 파싱이 있다면 효과 있음).

## 2) 가장 쉬운 방법: compute

`compute`는 "함수 + 입력 1개"를 다른 isolate에서 실행하고 결과를 돌려줍니다.

주의: `compute`에 넘기는 함수는 **top-level 함수**(또는 static)여야 합니다.

```dart
import 'dart:convert';
import 'package:flutter/foundation.dart';

List<int> parseBigJson(String rawJson) {
  final decoded = jsonDecode(rawJson) as List<dynamic>;
  return decoded.cast<int>();
}

Future<List<int>> parseInBackground(String rawJson) {
  return compute(parseBigJson, rawJson);
}
```

UI에서 사용:

```dart
Future<void> onLoad() async {
  setState(() => loading = true);
  final list = await parseInBackground(bigJsonString);
  setState(() {
    loading = false;
    items = list;
  });
}
```

## 3) Isolate 직접 쓰기 (개념만)

`compute`로 안 되는 "메시지 주고받기", "긴 작업 스트리밍" 같은 케이스는 Isolate를 직접 다루기도 합니다.

핵심 구성요소는 이 3개입니다.

- `Isolate.spawn`로 새 isolate 시작
- `SendPort/ReceivePort`로 메시지 통신
- 메시지로 결과/진행률 전달

처음에는 `compute`로 시작하고, 정말 필요할 때만 Isolate로 내려가는 걸 추천합니다.

## 4) 실전 팁

- "파싱/가공"은 isolate, "상태 반영"은 UI에서
- isolate로는 가능한 한 단순한 데이터만 전달(직렬화 가능한 형태)
- 너무 작은 작업을 isolate로 보내면 오히려 오버헤드가 더 큼

## 정리

- UI가 프리징되면 "메인 isolate에서 CPU 작업을 하는지" 의심
- 간단한 분리는 `compute`가 가장 쉽고 안전
- 복잡해지면 Isolate 직접 제어를 고려

무거운 JSON 처리 때문에 화면이 끊기는 앱이라면, `compute` 하나만으로도 체감이 확 달라지는 경우가 많아요.

