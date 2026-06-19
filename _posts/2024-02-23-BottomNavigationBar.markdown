---
layout: post
title: "BottomNavigationBar: 하단 탭 화면 전환을 안정적으로 구성하기"
date: 2024-02-23 22:37:00 +0900
category: Flutter
permalink: /flutter/navigator
description: "BottomNavigationBar로 하단 탭 전환을 구현할 때 상태 유지와 화면 구조를 안정적으로 가져가는 방법을 설명합니다."
image:
  path: "/assets/img/og/flutter-series-cover.svg"
  alt: "Flutter 시리즈 공통 대표 이미지"
tags: [BottomNavigationBar, Navigator]
---

# BottomNavigationBar: 하단 탭 화면 전환을 안정적으로 구성하기

> BottomNavigationBar로 하단 탭 전환을 구현할 때 상태 유지와 화면 구조를 안정적으로 가져가는 방법을 설명합니다.
>
> 이전 글: [Flutter Image 등록 및 사용: assets, pubspec.yaml, 에러 해결](/flutter/images)
> 다음 글: [VoidCallback과 Function: Flutter 위젯 사이에 이벤트 전달하기](/flutter/callback)
> 함께 보면 좋은 글:
> - [MaterialApp 이해하기: Flutter 앱의 가장 바깥 구조](/flutter/materialapp)
> - [go_router로 Flutter 라우팅 구성하기: 기본 이동, 파라미터, redirect](/flutter/go-router)

모바일 앱에서 하단 탭은 매우 익숙한 UI입니다. 홈, 검색, 알림, 마이페이지처럼 주요 영역을 빠르게 오갈 수 있게 해줍니다. Flutter에서는 `Scaffold`의 `bottomNavigationBar`와 `BottomNavigationBar`를 조합해 기본적인 하단 탭 구조를 만들 수 있습니다.

중요한 점은 탭을 누를 때마다 새 페이지를 `push`하는 것이 아니라, 현재 선택된 index에 따라 `body`를 바꾸는 방식이 일반적이라는 것입니다. 이 차이를 이해하지 못하면 뒤로가기 스택이 이상해지거나, 탭을 누를 때마다 같은 화면이 계속 쌓이는 문제가 생길 수 있습니다.

## 가장 기본적인 하단 탭

먼저 `StatefulWidget`으로 선택된 탭 index를 관리합니다.

```dart
class MainTabPage extends StatefulWidget {
  const MainTabPage({super.key});

  @override
  State<MainTabPage> createState() => _MainTabPageState();
}

class _MainTabPageState extends State<MainTabPage> {
  int currentIndex = 0;

  final pages = const [
    HomeTab(),
    SearchTab(),
    MyTab(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: pages[currentIndex],
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: currentIndex,
        onTap: (index) {
          setState(() {
            currentIndex = index;
          });
        },
        items: const [
          BottomNavigationBarItem(
            icon: Icon(Icons.home),
            label: '홈',
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.search),
            label: '검색',
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.person),
            label: '내정보',
          ),
        ],
      ),
    );
  }
}
```

`currentIndex`는 현재 선택된 탭 번호입니다. `onTap`에서 index를 받아 상태를 바꾸면 `body`가 해당 페이지로 변경됩니다.

## 탭 페이지 예시

각 탭은 일반 위젯으로 만들면 됩니다.

```dart
class HomeTab extends StatelessWidget {
  const HomeTab({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(child: Text('홈 화면'));
  }
}

class SearchTab extends StatelessWidget {
  const SearchTab({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(child: Text('검색 화면'));
  }
}

class MyTab extends StatelessWidget {
  const MyTab({super.key});

  @override
  Widget build(BuildContext context) {
    return const Center(child: Text('내정보 화면'));
  }
}
```

처음에는 이렇게 단순하게 시작하고, 각 탭이 커지면 별도 파일로 분리하면 됩니다.

## 탭 화면 상태 유지하기

위 기본 예제는 탭을 바꿀 때마다 `body`에 보이는 위젯이 교체됩니다. 단순 화면은 괜찮지만, 스크롤 위치나 입력값을 유지하고 싶다면 `IndexedStack`을 사용할 수 있습니다.

```dart
Scaffold(
  body: IndexedStack(
    index: currentIndex,
    children: pages,
  ),
  bottomNavigationBar: BottomNavigationBar(
    currentIndex: currentIndex,
    onTap: (index) => setState(() => currentIndex = index),
    items: const [
      BottomNavigationBarItem(icon: Icon(Icons.home), label: '홈'),
      BottomNavigationBarItem(icon: Icon(Icons.search), label: '검색'),
      BottomNavigationBarItem(icon: Icon(Icons.person), label: '내정보'),
    ],
  ),
)
```

`IndexedStack`은 모든 child를 유지하고, 현재 index에 해당하는 child만 보여줍니다. 그래서 탭을 이동해도 각 탭의 상태가 유지됩니다. 대신 탭이 많고 각 탭이 무거우면 메모리 사용량이 늘 수 있습니다. 상태 유지가 필요한 화면인지 먼저 판단하는 것이 좋습니다.

## 탭 클릭과 Navigator.push를 구분하기

하단 탭을 누를 때마다 다음처럼 `Navigator.push`를 호출하면 문제가 생길 수 있습니다.

```dart
onTap: (index) {
  Navigator.of(context).push(
    MaterialPageRoute(builder: (_) => const SearchTab()),
  );
}
```

이 방식은 탭 전환이 아니라 새 화면을 스택에 쌓는 동작입니다. 사용자가 탭을 여러 번 누르면 같은 화면이 계속 쌓이고, 뒤로가기를 눌렀을 때 이전 탭 상태로 돌아가는 이상한 흐름이 생길 수 있습니다. 하단 탭은 일반적으로 index 상태를 바꿔 body를 교체하는 방식으로 처리합니다.

상세 화면처럼 현재 탭 안에서 더 깊이 들어가는 경우에는 `Navigator.push`를 사용해도 됩니다.

```dart
ListTile(
  title: const Text('상품 상세 보기'),
  onTap: () {
    Navigator.of(context).push(
      MaterialPageRoute(
        builder: (_) => const ProductDetailPage(),
      ),
    );
  },
)
```

즉, 탭 전환은 index 변경, 상세 진입은 push라고 구분하면 됩니다.

## 아이템이 4개 이상일 때

`BottomNavigationBar`는 item이 4개 이상이면 기본 type이 shifting으로 동작할 수 있습니다. 색상이나 라벨 동작이 예상과 다르면 `type`을 명시합니다.

```dart
BottomNavigationBar(
  type: BottomNavigationBarType.fixed,
  currentIndex: currentIndex,
  onTap: (index) => setState(() => currentIndex = index),
  items: const [
    BottomNavigationBarItem(icon: Icon(Icons.home), label: '홈'),
    BottomNavigationBarItem(icon: Icon(Icons.search), label: '검색'),
    BottomNavigationBarItem(icon: Icon(Icons.notifications), label: '알림'),
    BottomNavigationBarItem(icon: Icon(Icons.person), label: '내정보'),
  ],
)
```

탭이 너무 많아지면 하단 바가 복잡해집니다. 주요 메뉴 3~5개 정도로 유지하고, 나머지는 마이페이지나 더보기 화면으로 보내는 것이 일반적입니다.

## go_router와 함께 확장하기

간단한 앱은 `BottomNavigationBar`와 index 상태만으로 충분합니다. 하지만 딥링크, 웹 URL, 탭별 Navigator, 로그인 redirect가 필요하면 라우터 기반 구조가 더 적합할 수 있습니다. 이때는 [go_router 글](/flutter/go-router)의 `ShellRoute` 같은 구조로 확장할 수 있습니다.

처음부터 복잡하게 시작할 필요는 없습니다. 먼저 index 기반 하단 탭을 이해하고, 요구사항이 생겼을 때 라우팅 구조로 옮겨도 충분합니다.

## 정리

`BottomNavigationBar`는 `currentIndex`와 `onTap`을 중심으로 동작합니다. 탭을 누를 때 새 화면을 push하는 것이 아니라 선택된 index를 바꾸고 `body`를 교체하는 것이 기본입니다. 탭별 상태를 유지해야 한다면 `IndexedStack`을 사용하고, 상세 화면으로 들어갈 때는 `Navigator.push`를 사용합니다. 이 구분만 명확히 해도 하단 탭 구조에서 생기는 많은 혼란을 줄일 수 있습니다.
