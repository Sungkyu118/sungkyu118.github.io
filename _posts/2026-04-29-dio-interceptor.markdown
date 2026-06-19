---
layout: post
title: "Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기"
date: 2026-04-29 00:10:00 +0900
category: Flutter
permalink: /flutter/dio-interceptor
description: "Dio Interceptor로 토큰 주입, 공통 로깅, 에러 변환을 한 곳에서 처리하는 구조를 설명합니다."
image:
  path: "/assets/img/og/flutter-dio-cover.svg"
  alt: "Flutter Dio Interceptor 포스트 대표 이미지"
---

# Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기

> Dio Interceptor로 토큰 주입, 공통 로깅, 에러 변환을 한 곳에서 처리하는 구조를 설명합니다.
>
> 이전 글: [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)
> 다음 글: [LayoutBuilder로 Flutter 반응형 레이아웃 만들기](/flutter/responsive-layoutbuilder)
> 함께 보면 좋은 글:
> - [flutter_secure_storage로 토큰 안전하게 저장하기](/flutter/secure-storage)
> - [Riverpod 기본 사용법: Provider, StateProvider, AsyncValue까지](/flutter/riverpod-basics)

Flutter 앱에서 서버 API를 호출하다 보면 거의 모든 요청에 반복되는 코드가 생깁니다. `Authorization` 헤더를 붙이고, 요청과 응답을 로그로 확인하고, 401 에러가 오면 로그인 화면으로 보내고, 네트워크가 끊기면 사용자에게 안내해야 합니다. 이 코드를 API 함수마다 직접 쓰면 처음에는 괜찮아 보여도, API가 20개만 넘어가도 유지보수가 힘들어집니다.

`Dio`의 `Interceptor`는 이런 공통 처리를 한곳에 모으기 위한 기능입니다. 요청이 서버로 나가기 전, 응답이 앱으로 돌아온 직후, 에러가 발생한 순간에 끼어들 수 있습니다. 이 글에서는 토큰 주입, 로그 출력, 공통 에러 처리, 401 처리 시 주의사항까지 실제 앱 구조에 가깝게 정리해보겠습니다.

## 설치와 기본 Dio 생성

먼저 `pubspec.yaml`에 Dio를 추가합니다.

```yaml
dependencies:
  dio: ^5.0.0
```

API 클라이언트는 앱 전체에서 재사용되므로 생성 함수를 따로 두는 편이 좋습니다.

```dart
import 'package:dio/dio.dart';

Dio createDio() {
  return Dio(
    BaseOptions(
      baseUrl: 'https://api.example.com',
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 10),
      headers: {
        'Accept': 'application/json',
      },
    ),
  );
}
```

`baseUrl`을 지정하면 API 호출 코드에서는 `/users/me`처럼 상대 경로만 사용할 수 있습니다. timeout은 반드시 넣는 것을 권장합니다. 네트워크가 불안정할 때 timeout이 없으면 사용자는 앱이 멈춘 것처럼 느낄 수 있습니다.

## 요청 전에 토큰 붙이기

로그인 후 받은 access token을 모든 인증 API에 붙여야 한다고 해봅시다. 가장 단순한 방식은 API 호출마다 다음 코드를 반복하는 것입니다.

```dart
await dio.get(
  '/users/me',
  options: Options(headers: {'Authorization': 'Bearer $token'}),
);
```

하지만 이 방식은 빠뜨리기 쉽고, 토큰 저장소가 바뀌면 모든 호출부를 고쳐야 합니다. 대신 토큰을 읽는 인터페이스를 만들고 인터셉터에서 처리합니다.

```dart
abstract class TokenStore {
  Future<String?> readAccessToken();
}

class AuthInterceptor extends Interceptor {
  AuthInterceptor(this.tokenStore);

  final TokenStore tokenStore;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await tokenStore.readAccessToken();

    if (token != null && token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }

    handler.next(options);
  }
}
```

인터셉터에서는 마지막에 반드시 `handler.next(options)`를 호출해야 요청이 다음 단계로 넘어갑니다. 이 호출을 빠뜨리면 API 요청이 끝나지 않고 대기 상태로 남을 수 있습니다.

```dart
Dio createAuthedDio(TokenStore tokenStore) {
  final dio = createDio();
  dio.interceptors.add(AuthInterceptor(tokenStore));
  return dio;
}
```

## 요청과 응답 로그 남기기

개발 중에는 어떤 URL로 요청이 나갔고, 어떤 상태 코드가 돌아왔는지 확인하는 것만으로도 디버깅 시간이 크게 줄어듭니다.

```dart
class SimpleLogInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // 실제 서비스에서는 민감 정보가 찍히지 않도록 주의합니다.
    print('[DIO] -> ${options.method} ${options.uri}');
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    print('[DIO] <- ${response.statusCode} ${response.requestOptions.uri}');
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    print('[DIO] !! ${err.type} ${err.requestOptions.uri}');
    print('[DIO] message: ${err.message}');
    handler.next(err);
  }
}
```

로그 인터셉터는 편리하지만 운영 환경에서 token, password, 주민번호, 결제 정보 같은 값이 출력되면 심각한 보안 문제가 됩니다. 그래서 보통 debug 빌드에서만 등록합니다.

```dart
import 'package:flutter/foundation.dart';

final dio = createAuthedDio(tokenStore);

if (kDebugMode) {
  dio.interceptors.add(SimpleLogInterceptor());
}
```

## API 에러를 앱 에러로 변환하기

Dio의 에러를 화면에서 그대로 다루면 UI 코드가 복잡해집니다. 서버 에러, timeout, 네트워크 끊김을 앱에서 쓰기 좋은 예외 타입으로 바꿔두면 화면에서는 메시지만 보여주면 됩니다.

```dart
class ApiException implements Exception {
  const ApiException(this.message);

  final String message;

  @override
  String toString() => message;
}

class ErrorMappingInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final statusCode = err.response?.statusCode;

    if (err.type == DioExceptionType.connectionTimeout ||
        err.type == DioExceptionType.receiveTimeout) {
      return handler.reject(
        DioException(
          requestOptions: err.requestOptions,
          error: const ApiException('서버 응답이 지연되고 있습니다. 잠시 후 다시 시도해주세요.'),
        ),
      );
    }

    if (statusCode == 500) {
      return handler.reject(
        DioException(
          requestOptions: err.requestOptions,
          response: err.response,
          error: const ApiException('서버에서 오류가 발생했습니다.'),
        ),
      );
    }

    handler.next(err);
  }
}
```

이렇게 하면 repository나 화면 코드에서 `err.error is ApiException` 형태로 사용자 메시지를 분리할 수 있습니다.

## 401 처리와 무한 루프 주의

401은 보통 access token 만료를 의미합니다. refresh token으로 access token을 재발급받고 원래 요청을 다시 보내는 구조를 만들 수 있지만, 여기서 가장 조심해야 할 문제는 무한 루프입니다. refresh 요청 자체가 401을 받았는데 다시 refresh를 시도하면 앱이 계속 같은 요청을 반복할 수 있습니다.

```dart
class UnauthorizedInterceptor extends Interceptor {
  UnauthorizedInterceptor({
    required this.onUnauthorized,
  });

  final VoidCallback onUnauthorized;

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final statusCode = err.response?.statusCode;
    final path = err.requestOptions.path;

    if (statusCode == 401 && !path.contains('/auth/refresh')) {
      onUnauthorized();
    }

    handler.next(err);
  }
}
```

실무에서는 여기서 바로 화면 이동을 하기보다 인증 상태를 관리하는 객체에 "로그아웃 필요" 이벤트를 전달하는 편이 안전합니다. 인터셉터는 네트워크 계층이고, 화면 이동은 UI 계층이기 때문입니다. 계층을 너무 섞으면 테스트와 유지보수가 어려워집니다.

## API 호출 코드 예시

인터셉터를 잘 구성하면 실제 API 코드는 훨씬 단순해집니다.

```dart
class UserApi {
  UserApi(this.dio);

  final Dio dio;

  Future<Map<String, dynamic>> fetchMe() async {
    final response = await dio.get<Map<String, dynamic>>('/users/me');
    return response.data ?? <String, dynamic>{};
  }
}
```

화면에서는 네트워크 세부 구현보다 성공, 로딩, 실패 상태에 집중할 수 있습니다. 이 흐름은 [Riverpod 기본 글](/flutter/riverpod-basics)의 `AsyncValue`나 notifier 구조와 함께 쓰면 더 깔끔해집니다.

## 정리

`onRequest`는 요청이 나가기 전 공통 헤더나 query parameter를 붙일 때 사용합니다. `onResponse`는 응답 로그나 공통 변환이 필요할 때 사용합니다. `onError`는 timeout, 401, 500 같은 실패를 한곳에서 처리할 때 사용합니다. 다만 인터셉터는 강력한 만큼 책임을 너무 많이 넣으면 거대한 블랙박스가 됩니다. 토큰 주입, 로그, 에러 매핑처럼 명확한 책임 단위로 나누고, 민감 정보 로그와 401 무한 루프를 조심하는 것이 핵심입니다.
