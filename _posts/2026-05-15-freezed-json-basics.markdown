---
layout: post
title: "freezed와 json_serializable로 안전한 모델 만들기"
date: 2026-05-15 01:00:00 +0900
category: Flutter
permalink: /flutter/freezed-json-basics
---

# freezed와 json_serializable로 안전한 모델 만들기

Flutter 앱에서 서버 API를 사용하면 JSON 데이터를 Dart 객체로 바꾸는 작업을 계속 하게 됩니다. 처음에는 `Map<String, dynamic>`에서 값을 꺼내 쓰는 방식으로도 충분해 보입니다. 하지만 필드가 많아지고 nullable 값이 섞이고, 응답 구조가 바뀌기 시작하면 런타임 에러가 늘어납니다. 문자열 key 오타 하나 때문에 앱이 터지는 일도 흔합니다.

`freezed`와 `json_serializable`은 모델 클래스를 더 안전하고 편하게 관리하기 위한 조합입니다. `freezed`는 불변 객체, `copyWith`, 값 비교를 도와주고, `json_serializable`은 JSON 변환 코드를 생성해줍니다. 이번 글에서는 설치부터 모델 작성, 코드 생성, 자주 발생하는 에러까지 차근차근 정리하겠습니다.

## 패키지 설치

`pubspec.yaml`에 런타임 의존성과 개발 의존성을 나눠서 추가합니다.

```yaml
dependencies:
  freezed_annotation: ^2.4.0
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.0
  freezed: ^2.5.0
  json_serializable: ^6.8.0
```

`freezed_annotation`, `json_annotation`은 앱 실행 코드에서 참조되는 annotation 패키지입니다. 반면 `build_runner`, `freezed`, `json_serializable`은 코드를 생성할 때만 필요하므로 `dev_dependencies`에 둡니다.

## 기본 모델 작성

사용자 모델을 만들어보겠습니다.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user.freezed.dart';
part 'user.g.dart';

@freezed
class User with _$User {
  const factory User({
    required int id,
    required String name,
    required String email,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
```

여기서 `part` 파일 이름은 현재 파일 이름과 맞아야 합니다. 파일이 `user.dart`라면 `user.freezed.dart`, `user.g.dart`가 됩니다. 파일 이름을 바꾸고 part 이름을 그대로 두면 생성 과정에서 에러가 납니다.

## 코드 생성

터미널에서 다음 명령을 실행합니다.

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

최근 프로젝트에서는 다음 명령을 사용해도 됩니다.

```bash
dart run build_runner build --delete-conflicting-outputs
```

성공하면 `user.freezed.dart`, `user.g.dart` 파일이 생성됩니다. 이 파일들은 직접 수정하지 않습니다. 모델 정의를 바꾸고 다시 생성해야 합니다.

## JSON 변환 사용하기

서버 응답을 모델로 바꾸는 코드는 단순해집니다.

```dart
final json = {
  'id': 1,
  'name': 'Sungkyu',
  'email': 'sungkyu@example.com',
};

final user = User.fromJson(json);
print(user.name);
```

반대로 모델을 JSON으로 바꿀 수도 있습니다.

```dart
final body = user.toJson();
```

API 요청/응답을 [Dio Interceptor 글](/flutter/dio-interceptor)의 네트워크 계층과 함께 구성하면, repository에서는 `User.fromJson(response.data!)`처럼 명확한 변환 흐름을 만들 수 있습니다.

## copyWith 사용하기

freezed의 큰 장점 중 하나는 `copyWith`입니다. 불변 객체는 필드를 직접 바꾸지 않고 새 객체를 만들어야 합니다.

```dart
final user = User(
  id: 1,
  name: 'Sungkyu',
  email: 'old@example.com',
);

final updated = user.copyWith(email: 'new@example.com');
```

상태 관리에서 특히 유용합니다.

```dart
@freezed
class ProfileState with _$ProfileState {
  const factory ProfileState({
    @Default(false) bool isLoading,
    String? errorMessage,
    User? user,
  }) = _ProfileState;
}
```

```dart
state = state.copyWith(isLoading: true, errorMessage: null);
```

이런 방식은 [Riverpod 기본 글](/flutter/riverpod-basics)의 상태 관리 구조와 잘 맞습니다.

## 기본값과 nullable

서버 값이 없을 수 있다면 nullable로 선언합니다.

```dart
@freezed
class Article with _$Article {
  const factory Article({
    required int id,
    required String title,
    String? thumbnailUrl,
    @Default(0) int likeCount,
  }) = _Article;

  factory Article.fromJson(Map<String, dynamic> json) =>
      _$ArticleFromJson(json);
}
```

`@Default(0)`을 사용하면 JSON에 `likeCount`가 없을 때 기본값 0을 사용할 수 있습니다. 다만 서버에서 `null`을 명시적으로 보내는 경우와 필드가 아예 없는 경우는 다르게 처리될 수 있으므로 API 응답 규칙을 확인해야 합니다.

## JSON key 이름이 다를 때

서버는 snake_case를 쓰고 Dart는 camelCase를 쓰는 경우가 많습니다.

```dart
@freezed
class Product with _$Product {
  const factory Product({
    required int id,
    @JsonKey(name: 'product_name') required String productName,
    @JsonKey(name: 'created_at') required DateTime createdAt,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) =>
      _$ProductFromJson(json);
}
```

`DateTime`은 서버 문자열 형식이 ISO 8601이면 자동 변환이 잘 되는 편입니다. 하지만 서버가 특이한 날짜 형식을 보내면 custom converter가 필요할 수 있습니다.

## 자주 만나는 에러

`Target of URI hasn't been generated` 에러는 생성 파일이 아직 없을 때 자주 나옵니다. `build_runner`를 실행했는지 확인해야 합니다.

`The name '_$UserFromJson' isn't defined` 에러는 `part 'user.g.dart';`가 없거나, `factory User.fromJson` 선언이 빠졌거나, 코드 생성이 실패했을 때 발생합니다.

`Conflicting outputs were detected` 에러가 나오면 생성된 파일과 현재 생성 결과가 충돌한다는 뜻입니다. 일반적으로 `--delete-conflicting-outputs` 옵션을 붙여 해결합니다.

```bash
dart run build_runner build --delete-conflicting-outputs
```

또 하나의 실수는 생성 파일을 직접 수정하는 것입니다. 생성 파일은 언제든 다시 덮어써질 수 있으므로 수정하면 안 됩니다. 모델 원본 파일을 고치고 다시 생성해야 합니다.

## 정리

`freezed`와 `json_serializable`을 사용하면 JSON 모델을 더 안전하게 다룰 수 있습니다. 문자열 key를 직접 꺼내는 코드를 줄이고, 불변 객체와 `copyWith`, 값 비교, JSON 변환을 자동 생성할 수 있습니다. 처음 설정은 조금 번거롭지만, API 모델이 늘어날수록 안정성과 생산성 차이가 커집니다. 특히 Riverpod 같은 상태 관리와 함께 사용하면 상태 변경 흐름이 훨씬 명확해집니다.
