---
layout: post
title: "flutter_secure_storage로 토큰 안전하게 저장하기"
date: 2026-05-15 00:40:00 +0900
category: Flutter
permalink: /flutter/secure-storage
description: "flutter_secure_storage로 토큰과 민감 정보를 저장할 때 보안상 주의할 점과 사용법을 정리합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
---

# flutter_secure_storage로 토큰 안전하게 저장하기

> flutter_secure_storage로 토큰과 민감 정보를 저장할 때 보안상 주의할 점과 사용법을 정리합니다.
>
> 이전 글: [Flutter isolate와 compute: 무거운 작업으로 UI가 멈출 때](/flutter/isolate-compute)
> 다음 글: [freezed와 json_serializable로 안전한 모델 만들기](/flutter/freezed-json-basics)
> 함께 보면 좋은 글:
> - [Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기](/flutter/dio-interceptor)
> - [freezed와 json_serializable로 안전한 모델 만들기](/flutter/freezed-json-basics)

로그인 기능이 있는 Flutter 앱에서는 access token, refresh token 같은 민감한 값을 저장해야 하는 경우가 많습니다. 이 값을 단순히 `SharedPreferences`에 저장하면 구현은 쉽지만 보안상 적절하지 않을 수 있습니다. 토큰은 사용자의 인증 권한과 직접 연결되므로 가능한 한 플랫폼에서 제공하는 안전한 저장소를 사용해야 합니다.

`flutter_secure_storage`는 Android의 Keystore, iOS의 Keychain 같은 보안 저장소를 활용해 값을 저장할 수 있게 도와주는 패키지입니다. 이번 글에서는 기본 사용법, 토큰 저장 구조, 로그아웃 처리, 자주 발생하는 에러와 주의사항까지 실제 앱 기준으로 정리해보겠습니다.

## 설치

`pubspec.yaml`에 패키지를 추가합니다.

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

패키지 버전은 프로젝트 SDK에 맞춰 조정해야 합니다. 여러 명이 함께 개발한다면 [FVM 글](/flutter/fvm)처럼 Flutter 버전을 고정해두면 빌드 환경 차이를 줄일 수 있습니다.

## 기본 읽기와 쓰기

가장 단순한 사용법은 `write`, `read`, `delete`입니다.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

const secureStorage = FlutterSecureStorage();

Future<void> saveToken(String token) async {
  await secureStorage.write(key: 'accessToken', value: token);
}

Future<String?> readToken() async {
  return secureStorage.read(key: 'accessToken');
}

Future<void> deleteToken() async {
  await secureStorage.delete(key: 'accessToken');
}
```

`read`는 값이 없으면 `null`을 반환합니다. 그래서 로그인 여부를 판단할 때는 null 처리를 반드시 해야 합니다.

```dart
final token = await readToken();

if (token == null || token.isEmpty) {
  // 로그인 화면으로 이동하거나 비로그인 상태로 처리합니다.
}
```

## 토큰 저장소 클래스로 감싸기

앱 곳곳에서 `FlutterSecureStorage`를 직접 호출하면 key 이름이 흩어지고 테스트도 어려워집니다. 보통은 저장소 클래스로 감쌉니다.

```dart
class TokenStorage {
  TokenStorage(this.storage);

  final FlutterSecureStorage storage;

  static const _accessTokenKey = 'auth.accessToken';
  static const _refreshTokenKey = 'auth.refreshToken';

  Future<void> saveTokens({
    required String accessToken,
    required String refreshToken,
  }) async {
    await storage.write(key: _accessTokenKey, value: accessToken);
    await storage.write(key: _refreshTokenKey, value: refreshToken);
  }

  Future<String?> readAccessToken() {
    return storage.read(key: _accessTokenKey);
  }

  Future<String?> readRefreshToken() {
    return storage.read(key: _refreshTokenKey);
  }

  Future<void> clear() async {
    await storage.delete(key: _accessTokenKey);
    await storage.delete(key: _refreshTokenKey);
  }
}
```

key 이름에 `auth.` 같은 prefix를 붙이면 다른 저장 값과 충돌할 가능성을 줄일 수 있습니다.

## Dio Interceptor와 연결하기

토큰을 저장했다면 API 요청마다 `Authorization` 헤더에 붙여야 합니다. 이 작업은 [Dio Interceptor 글](/flutter/dio-interceptor)에서 다룬 것처럼 인터셉터에 맡기면 좋습니다.

```dart
class SecureTokenStore {
  SecureTokenStore(this.tokenStorage);

  final TokenStorage tokenStorage;

  Future<String?> readAccessToken() {
    return tokenStorage.readAccessToken();
  }
}

class AuthInterceptor extends Interceptor {
  AuthInterceptor(this.tokenStore);

  final SecureTokenStore tokenStore;

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

저장소와 네트워크 계층을 분리하면 로그인, 로그아웃, 토큰 갱신 흐름을 관리하기 쉬워집니다.

## 자동 로그인 흐름

앱을 시작할 때 저장된 토큰이 있는지 확인해 자동 로그인처럼 처리할 수 있습니다.

```dart
enum AuthStatus {
  checking,
  authenticated,
  unauthenticated,
}

class AuthController extends ChangeNotifier {
  AuthController(this.tokenStorage);

  final TokenStorage tokenStorage;
  AuthStatus status = AuthStatus.checking;

  Future<void> load() async {
    final token = await tokenStorage.readAccessToken();
    status = token == null || token.isEmpty
        ? AuthStatus.unauthenticated
        : AuthStatus.authenticated;
    notifyListeners();
  }

  Future<void> logout() async {
    await tokenStorage.clear();
    status = AuthStatus.unauthenticated;
    notifyListeners();
  }
}
```

이 상태는 [go_router 글](/flutter/go-router)의 redirect와 연결할 수 있습니다. 앱 시작 직후에는 `checking` 상태를 두고 스플래시나 로딩 화면을 보여주면, 토큰을 읽기 전에 로그인 화면이 잠깐 보였다가 홈으로 이동하는 깜빡임을 줄일 수 있습니다.

## SharedPreferences와의 차이

`SharedPreferences`는 설정값, 온보딩 확인 여부, 테마 선택처럼 민감하지 않은 작은 값을 저장하는 데 적합합니다. 반면 토큰, 세션 키, 개인 식별 정보처럼 유출되면 문제가 되는 값은 secure storage를 사용하는 것이 좋습니다.

다만 secure storage가 모든 보안 문제를 해결해주는 것은 아닙니다. 루팅된 기기, 탈옥된 기기, 악성 앱, 디버그 로그 유출 같은 위험은 별도로 고려해야 합니다. 토큰 만료 시간을 짧게 가져가고, refresh token 회전, 서버 측 폐기 처리도 함께 설계해야 합니다.

## 자주 만나는 문제

Android에서 백업/복원 과정 때문에 암호화 키가 맞지 않아 읽기 실패가 발생하는 사례가 있습니다. 앱을 재설치하거나 기기를 바꾼 뒤 기존 암호화 데이터가 복원되었지만 Keystore 키는 복원되지 않는 상황입니다. 이런 경우 패키지 문서에서 안내하는 Android backup 설정을 확인해야 합니다.

또 하나는 로그에 토큰을 찍는 실수입니다. 디버깅 중 `print(token)`을 남겨두면 로그 수집 도구나 화면 공유 과정에서 토큰이 노출될 수 있습니다. 네트워크 로그에서도 `Authorization` 헤더는 마스킹하는 것이 안전합니다.

토큰을 너무 오래 저장하는 것도 위험합니다. "자동 로그인이 편하니까 refresh token을 영구 저장하자"는 접근은 보안 사고 시 피해를 키울 수 있습니다. 서비스 성격에 따라 만료 정책과 재로그인 정책을 정해야 합니다.

<!-- codex-category-inline-links:start -->

지금 읽고 계신 주제가 아직 조금 추상적으로 느껴지신다면 [Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기](/flutter/dio-interceptor), [go_router로 Flutter 라우팅 구성하기: 기본 이동, 파라미터, redirect](/flutter/go-router), [Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지](/flutter/widget-test-basics) 글도 함께 읽어보시면 좋겠습니다. 같은 Flutter 흐름 안에서 앞단의 배경과 다음 단계의 확장 포인트를 같이 보실 수 있어서, 지금 배우는 내용이 실제 프로젝트에서 어디에 연결되는지 훨씬 더 선명하게 이해하실 수 있습니다.

<!-- codex-category-inline-links:end -->
## 정리

`flutter_secure_storage`는 토큰처럼 민감한 값을 플랫폼 보안 저장소에 저장하기 위한 도구입니다. 직접 호출을 앱 전체에 흩뿌리기보다 `TokenStorage` 같은 클래스로 감싸고, 네트워크 요청에는 Dio 인터셉터를 통해 토큰을 붙이는 구조가 유지보수에 좋습니다. secure storage를 쓰더라도 로그 노출, 백업/복원, 토큰 만료 정책까지 함께 고려해야 실제로 안전한 인증 흐름을 만들 수 있습니다.

<!-- codex-category-links:start -->

## 이어서 읽어보시면 좋습니다

- [Dio Interceptor로 토큰, 로그, 에러 처리를 공통화하기](/flutter/dio-interceptor)
- [go_router로 Flutter 라우팅 구성하기: 기본 이동, 파라미터, redirect](/flutter/go-router)
- [Flutter Widget Test 입문: 버튼 클릭부터 비동기 화면까지](/flutter/widget-test-basics)

지금 글과 바로 이어서 읽기 좋은 흐름으로 묶어두었으니, 개념을 비교해보시거나 다음 실습으로 넘어가실 때 차근차근 따라가보시면 좋겠습니다.

<!-- codex-category-links:end -->
