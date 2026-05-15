---
layout: post
title: "[Flutter] 위젯 테스트 기본: pumpWidget, find, tap, expect"
date: 2026-05-15 00:10:00 +0900
category: Flutter
permalink: /flutter/flutter-widget-test-basics
---

# [Flutter] 위젯 테스트 기본: pumpWidget, find, tap, expect

Flutter는 UI가 복잡해질수록 "수동 테스트" 비용이 급격히 올라갑니다. 위젯 테스트는 화면의 핵심 동작을 코드로 고정해두는 가장 가성비 좋은 방법 중 하나예요.

이 글은 위젯 테스트에서 가장 많이 쓰는 흐름을 "한 번에" 잡는 걸 목표로 합니다.

## 1) 테스트 파일 만들기

보통 `test/` 아래에 만듭니다.

예: `test/counter_test.dart`

## 2) 가장 기본 형태

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets("counter increments", (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        home: CounterPage(),
      ),
    );

    expect(find.text("0"), findsOneWidget);

    await tester.tap(find.byIcon(Icons.add));
    await tester.pump();

    expect(find.text("1"), findsOneWidget);
  });
}

class CounterPage extends StatefulWidget {
  const CounterPage({super.key});
  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int count = 0;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      floatingActionButton: FloatingActionButton(
        onPressed: () => setState(() => count++),
        child: const Icon(Icons.add),
      ),
      body: Center(child: Text("$count")),
    );
  }
}
```

핵심은 이 4가지 조합입니다.

- `pumpWidget`: 화면 띄우기
- `find`: 위젯 찾기
- `tap/enterText/drag`: 사용자 행동 시뮬레이션
- `expect`: 결과 검증

## 3) find 치트시트

```dart
find.text("로그인");
find.byType(TextField);
find.byIcon(Icons.settings);
find.byKey(const Key("login_button"));
```

실전에서는 `Key`를 박아두는 게 가장 안정적입니다.

## 4) 비동기/애니메이션이 있으면 pumpAndSettle

버튼 탭 후 화면 전환이나 애니메이션이 돌면, `pump()` 한 번으로는 상태가 덜 반영될 수 있어요.

```dart
await tester.tap(find.text("다음"));
await tester.pumpAndSettle();
```

## 5) 네트워크/의존성은 주입해서 테스트하기

위젯 테스트에서 실제 HTTP를 때리면 느리고 불안정합니다. 보통은 다음 중 하나로 분리합니다.

- API/Repo를 위젯 밖에서 주입
- fake/mock 구현체로 대체
- Riverpod/Provider라면 오버라이드로 대체

## 정리

- 위젯 테스트는 UI의 핵심 동작을 자동으로 보장해준다
- `pumpWidget → action(tap/enterText) → pump/pumpAndSettle → expect` 흐름을 익히면 된다
- `Key` 기반 찾기가 안정적이다

다음 글로는 "로그인 폼 테스트(텍스트 입력 + 버튼 활성화 + 에러 메시지)" 같은 패턴을 한 번 더 잡아두면 실전에서 바로 써먹기 좋아요.

