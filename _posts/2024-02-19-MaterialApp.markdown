---
layout: post
title: "MaterialApp 이해하기: Flutter 앱의 가장 바깥 구조"
date: 2024-02-19 21:53:00 +0900
category: Flutter
permalink: /flutter/materialapp
description: "MaterialApp이 앱 전체 구조에서 맡는 역할과 테마, 라우팅, 초기 화면 설정의 기본을 정리합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
tags: [MaterialApp]
---

# MaterialApp 이해하기: Flutter 앱의 가장 바깥 구조

> MaterialApp이 앱 전체 구조에서 맡는 역할과 테마, 라우팅, 초기 화면 설정의 기본을 정리합니다.
>
> 이전 글: [FVM으로 Flutter 버전 관리하기: 프로젝트마다 SDK 고정하기](/flutter/fvm)
> 다음 글: [Container와 BoxDecoration: 색상 충돌 에러부터 실전 스타일링까지](/flutter/containercolor)
> 함께 보면 좋은 글:
> - [go_router로 Flutter 라우팅 구성하기: 기본 이동, 파라미터, redirect](/flutter/go-router)
> - [BottomNavigationBar: 하단 탭 화면 전환을 안정적으로 구성하기](/flutter/navigator)

Flutter 프로젝트를 만들면 `main.dart`에서 거의 항상 `MaterialApp`을 만나게 됩니다. 처음에는 예제 코드에 있으니까 그냥 복사해서 쓰지만, 앱이 커질수록 `MaterialApp`이 어떤 역할을 하는지 이해하는 것이 중요해집니다. 라우팅, 테마, 언어 설정, 디버그 배너, Navigator 환경이 이 위젯을 중심으로 구성되기 때문입니다.

`MaterialApp`은 화면 하나를 예쁘게 보여주는 위젯이라기보다, Material Design 기반 앱이 동작하기 위한 공통 환경을 제공하는 위젯입니다. 쉽게 말하면 Flutter 앱의 "바깥 껍데기이자 설정 중심지"라고 볼 수 있습니다.

## 가장 기본적인 구조

Flutter 앱의 시작점은 `main()`입니다.

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      home: HomePage(),
    );
  }
}

class HomePage extends StatelessWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Home')),
      body: const Center(child: Text('Hello Flutter')),
    );
  }
}
```

`runApp`은 Flutter 위젯 트리의 시작점을 등록합니다. 그 안에 `MaterialApp`이 있고, `home`에는 첫 화면이 들어갑니다. 실제 화면 구조는 보통 `Scaffold`가 담당합니다.

## MaterialApp과 Scaffold의 차이

초보자가 가장 많이 헷갈리는 부분이 `MaterialApp`과 `Scaffold`의 역할입니다. `MaterialApp`은 앱 전체 설정을 담당하고, `Scaffold`는 한 화면의 기본 구조를 담당합니다.

```dart
MaterialApp(
  home: Scaffold(
    appBar: AppBar(title: const Text('제목')),
    body: const Center(child: Text('본문')),
  ),
)
```

`Scaffold`는 appBar, body, floatingActionButton, drawer, bottomNavigationBar 같은 화면 구성 요소를 제공합니다. 반면 `MaterialApp`은 테마, 라우팅, locale, navigator 같은 앱 전체 환경을 제공합니다.

## title과 debugShowCheckedModeBanner

처음 앱을 실행하면 오른쪽 위에 DEBUG 배너가 보입니다. 개발 중이라는 표시입니다. 스크린샷이나 데모 화면에서 거슬린다면 끌 수 있습니다.

```dart
MaterialApp(
  title: 'Flutter Practice',
  debugShowCheckedModeBanner: false,
  home: const HomePage(),
)
```

`title`은 Android task switcher나 웹 페이지 제목 등에서 사용될 수 있습니다. 앱 이름을 의미 있게 넣어두는 편이 좋습니다.

## ThemeData로 공통 스타일 정하기

앱 전체 색상과 스타일은 `theme`에서 관리합니다.

```dart
final appTheme = ThemeData(
  colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
  useMaterial3: true,
  appBarTheme: const AppBarTheme(
    centerTitle: true,
  ),
);

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: appTheme,
      home: const HomePage(),
    );
  }
}
```

화면마다 직접 색상과 글꼴을 지정하면 나중에 디자인을 바꾸기 어렵습니다. 공통 스타일은 가능하면 `ThemeData`에 올리고, 각 화면에서는 `Theme.of(context)`를 통해 가져다 쓰는 방식이 유지보수에 좋습니다.

```dart
Text(
  '중요한 문구',
  style: Theme.of(context).textTheme.titleLarge,
)
```

## 라우팅 설정

작은 앱에서는 `home` 하나로 시작해도 됩니다. 하지만 화면이 여러 개가 되면 `routes`를 사용할 수 있습니다.

```dart
MaterialApp(
  initialRoute: '/',
  routes: {
    '/': (context) => const HomePage(),
    '/settings': (context) => const SettingsPage(),
  },
)
```

이동할 때는 다음처럼 호출합니다.

```dart
Navigator.of(context).pushNamed('/settings');
```

앱 규모가 더 커지고 로그인 가드, 딥링크, URL 파라미터가 필요해지면 [go_router 글](/flutter/go-router)처럼 `MaterialApp.router`를 사용하는 방식으로 확장할 수 있습니다.

## MaterialApp을 화면마다 넣으면 안 되는 이유

가끔 페이지 위젯 안에 다시 `MaterialApp`을 넣는 코드가 있습니다.

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        body: Center(child: Text('Login')),
      ),
    );
  }
}
```

이 코드는 화면이 보일 수는 있지만 좋은 구조가 아닙니다. `MaterialApp`이 여러 개 생기면 Navigator, Theme, MediaQuery 흐름이 의도와 다르게 분리될 수 있습니다. 일반 페이지는 `Scaffold`부터 작성하고, `MaterialApp`은 앱 최상단에 하나만 두는 것을 기본으로 생각하면 됩니다.

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: const Center(child: Text('Login')),
    );
  }
}
```

## context 위치 주의

`Theme.of(context)`나 `Navigator.of(context)`는 context가 위젯 트리에서 어디에 있는지에 따라 결과가 달라집니다. `MaterialApp` 아래에 있는 context에서 호출해야 Material 관련 설정을 찾을 수 있습니다. 앱을 만들다 보면 context 관련 에러가 나오는데, 이때는 "이 context가 MaterialApp 아래에 있는가"를 먼저 생각해보면 좋습니다.

## 정리

`MaterialApp`은 Flutter Material 앱의 시작점입니다. 화면 하나의 레이아웃보다는 앱 전체의 테마, 라우팅, Navigator 환경을 제공하는 역할에 가깝습니다. 처음에는 `MaterialApp -> Scaffold -> body` 구조만 정확히 이해해도 많은 예제를 읽기 쉬워집니다. 이후 테마, 라우팅, 상태 관리로 확장하면 Flutter 앱 구조가 훨씬 자연스럽게 보입니다.
