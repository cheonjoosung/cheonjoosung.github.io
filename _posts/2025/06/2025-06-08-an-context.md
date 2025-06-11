---
title: Android Context란 무엇인가? 종류와 올바른 사용법
tags: [ Android ]
style: fill
color: dark
description: Android 개발에서 자주 접하는 Context의 개념과 종류, 그리고 언제 어떤 Context를 사용해야 하는지를 정리합니다.
---

## ✨ 개요

- Context는 Android에서 앱의 환경, 상태, 리소스에 접근하기 위한 핵심 객체입니다. 
- 컴포넌트(Activity, Service 등)들은 이 Context를 통해 시스템 서비스에 접근하고, 리소스를 로드하며, 새로운 컴포넌트를 실행합니다.
- 간단히 말해, Context는 앱 안에서 모든 것에 접근할 수 있는 게이트웨이입니다.

---

## 1. ✅ Context의 주요 역할

- 시스템 서비스 접근 (getSystemService())
- 리소스 및 파일 접근 (getResources(), openFileInput() 등)
- Activity, Service, Broadcast 시작
- 앱 패키지 정보 조회 (getPackageName(), getApplicationInfo() 등)

---

## 2. ✅ Context의 주요 종류

### 2.1 Application Context

- 앱 전체에 걸쳐 존재하는 Context (싱글 인스턴스)
- 메모리 누수 없이 장기 작업에 적합
- 예: 데이터베이스, SharedPreferences 등 초기화

### 2.2 Activity Context

- 특정 Activity의 생명주기와 연결됨
- UI 요소에 접근 가능
- Dialog, Toast, View 등의 생성에 필요

### 2.3 Service Context

- Service 내에서 사용되는 Context
- Application Context와 유사하나, Service의 생명주기를 따름 

---

## 3. ✅ 언제 어떤 Context를 써야 하나?

| 사용 상황               | 추천 Context        | 이유                              |
|------------------------|---------------------|-----------------------------------|
| View 생성 또는 UI 조작 | Activity Context    | UI 생명주기와 밀접한 연관          |
| DB, 로그 초기화 등     | Application Context | Activity 종료와 무관하게 유지 필요 |
| 앱 전체 설정 접근      | Application Context | 리소스 누수 방지 및 효율성         |

---

## 4.🧠 결론

- Context는 Android 앱 구성요소 간의 연결을 담당하는 중요한 매개체입니다.
- 용도에 맞는 Context를 정확히 사용하는 것이 메모리 누수 방지와 앱 안정성의 핵심입니다.
- Application Context와 Activity Context를 혼동하지 않고 구분하는 습관을 들이면 버그를 줄일 수 있습니다.