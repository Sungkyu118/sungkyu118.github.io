---
layout: post
title: "[Flutter] freezed + json_serializable로 모델/JSON 파싱 생산성 올리기"
date: 2026-05-15 00:30:00 +0900
category: Flutter
permalink: /flutter/freezed-json-basics
---

# [Flutter] freezed + json_serializable로 모델/JSON 파싱 생산성 올리기

API 붙이다 보면 가장 시간이 많이 새는 곳이 "모델 클래스 만들기 + copyWith + equality + JSON 파싱"입니다. `freezed` + `json_serializable` 조합을 쓰면 이 반복 작업을 자동화해서 생산성이 확 올라가요.

## 1) 설치

```yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.4.0
  json_serializable: ^6.8.0
```

## 2) 모델 만들기

예: `user.dart`

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    @Default(0) int age,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

포인트:
- `@freezed`가 `copyWith`, `==`, `hashCode` 등을 생성
- `fromJson/toJson`은 `json_serializable`이 생성
- `@Default`로 기본값도 간단히 처리

## 3) 코드 생성

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

개발 중에는 watch가 편합니다.

```bash
flutter pub run build_runner watch --delete-conflicting-outputs
```

## 4) 사용 예시

```dart
final user = User.fromJson({"id": "1", "name": "Sungkyu"});
final updated = user.copyWith(age: 20);
print(updated.toJson());
```

## 5) 실전 팁

- 파일명/part 이름만 정확히 맞추면 스트레스가 확 줄어듭니다.
- DTO(네트워크용)와 Domain(앱 내부 모델)을 분리하는 팀도 많아요.
- JSON 키가 다르면 `@JsonKey(name: "user_id")`로 매핑합니다.

## 정리

`freezed`는 "모델 보일러플레이트"를 거의 없애주는 도구라, 앱 규모가 커질수록 체감이 큽니다. API 많이 붙일 예정이면 초반에 세팅해두는 걸 추천해요.

