---
layout: post
title: "freezed + json_serializable로 모델/JSON 파싱 생산성 올리기"
date: 2026-05-15 00:30:00 +0900
category: Flutter
permalink: /flutter/freezed-json-basics
---

# freezed + json_serializable로 모델/JSON 파싱 생산성 올리기

API를 붙이다 보면 시간이 가장 많이 새는 곳이 "모델 클래스 만들기 + copyWith + equality + JSON 파싱"입니다. `freezed` + `json_serializable` 조합을 쓰면 이 반복 작업을 자동화해서 생산성이 확 올라가요.

이 글은 아래 4가지를 모두 다룹니다.

1. 언제/왜 쓰는지(트레이드오프)
2. 실전 예제 1개를 끝까지(네트워크 응답 파싱까지)
3. 흔한 실수/디버깅 포인트
4. 대안 비교(수동 모델, json_serializable 단독 등)

## 1) 언제/왜 쓰나 (트레이드오프)

### 쓰면 좋은 경우

- API 응답 모델이 많다(모델 10개만 넘어도 보일러플레이트가 급증)
- 상태관리에서 `copyWith`로 부분 업데이트를 자주 한다
- equality(==)가 필요하다(예: 상태 변경 감지, 리스트 diff 등)
- 팀에서 모델 작성 규칙을 통일하고 싶다

### 단점/주의점

- 코드 생성 단계가 생긴다(`build_runner`)
- `part`/파일명 규칙이 깨지면 생성이 안 된다
- JSON 구조가 자주 바뀌면 DTO/Domain 분리 전략이 필요할 수 있다

결론적으로 "앱이 조금이라도 커질 예정"이면 초반에 깔아두는 편이 보통 이득입니다.

## 2) 설치

```yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.4.0
  json_serializable: ^6.8.0
```

## 3) 모델 만들기

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

여기서 자동으로 생기는 것들:

- `copyWith`
- `==`, `hashCode`
- `toJson`, `fromJson`

## 4) 코드 생성

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

개발 중에는 watch가 편합니다.

```bash
flutter pub run build_runner watch --delete-conflicting-outputs
```

## 5) 실전 예제: Dio 응답 파싱까지 한 번에

서버가 이런 응답을 준다고 가정해볼게요.

```json
{
  "id": "1",
  "name": "Sungkyu",
  "age": 20
}
```

### (1) API 클라이언트

```dart
import 'package:dio/dio.dart';

class UserApi {
  final Dio dio;
  UserApi(this.dio);

  Future<User> fetchMe() async {
    final res = await dio.get("/me");
    final data = res.data as Map<String, dynamic>;
    return User.fromJson(data);
  }
}
```

### (2) UI/상태에서 copyWith로 가공

```dart
final user = await api.fetchMe();
final normalized = user.copyWith(name: user.name.trim());
```

`freezed`는 equality도 자동으로 만들어주기 때문에, 상태관리에서 불필요한 재빌드/업데이트를 줄이는 데에도 도움이 되는 경우가 많아요.

## 6) 흔한 실수/디버깅 포인트

### (1) part/파일명 불일치

`part 'user.freezed.dart';`, `part 'user.g.dart';`는 현재 파일명과 정확히 맞아야 합니다. 이게 어긋나면 생성이 안 됩니다.

### (2) 생성 충돌/꼬임

이미 생성된 파일이 충돌하면 아래 옵션이 거의 필수입니다.

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

### (3) 응답 타입 캐스팅 오류

`res.data`가 `List`인데 `Map`으로 캐스팅하면 런타임 에러가 납니다.

- 단건: `Map<String, dynamic>`
- 목록: `List<dynamic>`로 받고 각 요소를 map으로 변환

### (4) null/기본값 처리

서버가 필드를 생략할 수 있으면 `@Default(...)` 또는 nullable(`int?`)로 설계해야 합니다. 서버 스펙이 애매한 구간을 초반에 정리해두면 디버깅 시간이 크게 줄어요.

## 7) 대안 비교

### 수동 모델(plain class)

모델이 1~2개인 작은 앱이라면 수동이 더 빠를 수 있어요. 하지만 모델이 늘어날수록 `copyWith/equality/toJson/fromJson` 반복이 커집니다.

### json_serializable 단독 사용

`freezed`까지는 과하다고 느끼면 `json_serializable`만 사용해서 JSON 파싱만 자동화할 수도 있습니다. 대신 `copyWith/equality`는 직접 구현해야 합니다.

### built_value 등

대안은 있지만, Flutter 생태계에서 `freezed`는 레퍼런스가 많고 팀원 온보딩이 쉬운 편입니다.

## 정리

- 모델이 늘어날수록 `freezed`의 이득이 커진다
- 실전에서는 "파일명/part/타입 캐스팅"에서 가장 자주 삽질한다
- 작은 앱이면 수동도 가능하지만, 장기적으로는 자동화가 시간을 아껴준다
