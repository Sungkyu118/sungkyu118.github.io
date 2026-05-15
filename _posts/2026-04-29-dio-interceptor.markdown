---
layout: post
title: "Dio Interceptor로 토큰 붙이기, 로그 남기기"
date: 2026-04-29 00:10:00 +0900
category: Flutter
permalink: /flutter/dio-interceptor
---

# Dio Interceptor로 토큰 붙이기, 로그 남기기

Dio를 쓰다 보면 "모든 요청에 Authorization 헤더를 붙이기" 같은 공통 처리가 필요합니다. 이럴 때 `Interceptor`를 붙여두면 각 API 호출 코드가 깔끔해집니다.

## 1) 설치

```yaml
dependencies:
  dio: ^5.0.0
```

## 2) 기본 Dio 생성

```dart
import 'package:dio/dio.dart';

Dio createDio() {
  final dio = Dio(
    BaseOptions(
      baseUrl: "https://api.example.com",
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 10),
    ),
  );

  return dio;
}
```

## 3) Authorization 토큰 자동 첨부

토큰 저장소를 인터셉터에 주입해두면, 요청마다 토큰을 꺼내서 헤더에 붙일 수 있습니다.

```dart
abstract class TokenStore {
  Future<String?> readAccessToken();
}

class AuthInterceptor extends Interceptor {
  final TokenStore tokenStore;
  AuthInterceptor(this.tokenStore);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await tokenStore.readAccessToken();
    if (token != null && token.isNotEmpty) {
      options.headers["Authorization"] = "Bearer $token";
    }
    handler.next(options);
  }
}
```

적용:

```dart
Dio createAuthedDio(TokenStore tokenStore) {
  final dio = createDio();
  dio.interceptors.add(AuthInterceptor(tokenStore));
  return dio;
}
```

## 4) 요청/응답 로깅 (개발용)

개발 중에는 요청 URL, status code, 응답 일부를 찍어두면 디버깅이 쉬워집니다.

```dart
class SimpleLogInterceptor extends Interceptor {
  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // ignore: avoid_print
    print("[DIO] -> ${options.method} ${options.uri}");
    handler.next(options);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    // ignore: avoid_print
    print("[DIO] <- ${response.statusCode} ${response.requestOptions.uri}");
    handler.next(response);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    // ignore: avoid_print
    print("[DIO] !! ${err.type} ${err.requestOptions.uri} ${err.message}");
    handler.next(err);
  }
}
```

적용:

```dart
dio.interceptors.add(SimpleLogInterceptor());
```

## 5) 실패 공통 처리 (예: 401)

401이 오면 "토큰 만료"일 가능성이 큽니다. 이때는 재로그인 유도, 리프레시 토큰 플로우 등으로 이어질 수 있어요.

```dart
class UnauthorizedInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (err.response?.statusCode == 401) {
      // TODO: logout, refresh flow, navigate to login, ...
    }
    handler.next(err);
  }
}
```

## 정리

- `onRequest`: 공통 헤더/쿼리 파라미터 주입
- `onResponse`: 로깅/응답 가공
- `onError`: 공통 에러 처리

다음 글로는 "리프레시 토큰으로 자동 재시도" 패턴을 한 번 제대로 정리해두면, 앱 전체 네트워크 레이어가 훨씬 안정적으로 굴러가요.
