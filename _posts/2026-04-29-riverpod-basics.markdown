---
layout: post
title: "[Flutter] Riverpod 기본 사용법 (Provider, Consumer, 상태 갱신)"
date: 2026-04-29 00:00:00 +0900
category: Flutter
permalink: /flutter/riverpod-basics
---

# [Flutter] Riverpod 기본 사용법 (Provider, Consumer, 상태 갱신)

Flutter에서 상태를 관리할 때 Riverpod은 "의존성 주입 + 상태관리"를 깔끔하게 묶어줍니다. 이 글에서는 가장 자주 쓰는 3가지(Provider, StateProvider, StateNotifierProvider)만 빠르게 감 잡는 걸 목표로 정리합니다.

## 1) 설치

`pubspec.yaml`에 추가:

```yaml
dependencies:
  flutter_riverpod: ^2.0.0
```

## 2) ProviderScope로 시작하기

앱 루트에 `ProviderScope`를 감싸야 합니다.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() {
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: CounterPage());
  }
}
```

## 3) 읽기 전용: Provider

환경설정 값이나 계산된 값처럼 "변하지 않는 의존성"에 잘 맞습니다.

```dart
final apiBaseUrlProvider = Provider<String>((ref) {
  return "https://api.example.com";
});
```

사용:

```dart
class DebugBaseUrl extends ConsumerWidget {
  const DebugBaseUrl({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final baseUrl = ref.watch(apiBaseUrlProvider);
    return Text(baseUrl);
  }
}
```

## 4) 간단 상태: StateProvider

정말 단순한 상태(토글, 선택값, 숫자 카운터 등)라면 `StateProvider`가 제일 빠릅니다.

```dart
final counterProvider = StateProvider<int>((ref) => 0);
```

`watch`로 읽고, `.notifier`로 수정:

```dart
class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);

    return Scaffold(
      appBar: AppBar(title: const Text("Riverpod Counter")),
      body: Center(child: Text("$count")),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          ref.read(counterProvider.notifier).state++;
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

## 5) 로직 분리: StateNotifierProvider

상태 갱신 규칙이 늘어나거나 비동기 처리, 검증 로직이 들어가면 `StateNotifier`로 옮기는 편이 유지보수에 유리합니다.

예: 로딩/에러/데이터를 함께 다루는 간단한 상태 모델

```dart
class ProfileState {
  final bool isLoading;
  final String? error;
  final String? nickname;

  const ProfileState({
    this.isLoading = false,
    this.error,
    this.nickname,
  });

  ProfileState copyWith({bool? isLoading, String? error, String? nickname}) {
    return ProfileState(
      isLoading: isLoading ?? this.isLoading,
      error: error,
      nickname: nickname ?? this.nickname,
    );
  }
}

class ProfileNotifier extends StateNotifier<ProfileState> {
  ProfileNotifier() : super(const ProfileState());

  Future<void> load() async {
    state = state.copyWith(isLoading: true, error: null);
    try {
      await Future<void>.delayed(const Duration(milliseconds: 400));
      state = state.copyWith(isLoading: false, nickname: "Sungkyu");
    } catch (e) {
      state = state.copyWith(isLoading: false, error: e.toString());
    }
  }
}

final profileProvider =
    StateNotifierProvider<ProfileNotifier, ProfileState>((ref) {
  return ProfileNotifier();
});
```

UI에서 사용:

```dart
class ProfilePage extends ConsumerWidget {
  const ProfilePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(profileProvider);

    return Scaffold(
      appBar: AppBar(title: const Text("Profile")),
      body: Center(
        child: state.isLoading
            ? const CircularProgressIndicator()
            : Text(state.error ?? (state.nickname ?? "-")),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => ref.read(profileProvider.notifier).load(),
        child: const Icon(Icons.refresh),
      ),
    );
  }
}
```

## 정리

- `Provider`: 읽기 전용 의존성/계산값
- `StateProvider`: 아주 단순한 상태
- `StateNotifierProvider`: 상태 변경 규칙이 있는 로직

다음 글에서는 "네트워크 요청 + 캐시 + 에러 처리" 같이 실전 패턴을 Riverpod으로 어떻게 구조화하는지(예: `AsyncValue`)로 이어가면 좋아요.
