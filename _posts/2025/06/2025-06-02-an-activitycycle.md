---
title: Android Activity 생명주기 완벽 정리 - onCreate부터 onDestroy까지
tags: [ Android ]
style: fill
color: dark
description: Android 앱 개발에서 가장 기본이 되는 Activity Lifecycle(생명주기)을 이해하고 각 단계에서 해야 할 작업과 주의사항을 정리합니다.
---

## ✨ 개요

Android 앱 개발에서 **Activity 생명주기(Lifecycle)** 는 앱의 동작 흐름을 이해하고 안정적인 앱을 만들기 위한 핵심 개념입니다.

앱이 실행되고, 화면이 바뀌고, 백그라운드로 전환되며, 종료되기까지의 일련의 흐름은 **Activity의 생명주기 메서드**를 통해 관리됩니다.

---

## 1. ✅ 전체 생명주기 흐름

+  앱 시작 시 `onCreate()` → `onStart()` → `onResume()`
+  다른 앱 열어서 가릴 때 `onPause()` → `onStop()`
+  다시 돌아왔을 때 `onRestart()` → `onStart()` → `onResume()`
+  앱 완전 종료 또는 시스템에 의한 제거 `onPause()` → `onStop()` → `onDestroy()`
+ 화면 회전 시
    * `onPause()` → `onStop()` → `onSaveInstanceState` → `onDestroy()`
    * →`onCreate()` → `onStart()` → `onRestoreInstanceState()` → `onResume()`
    * 왜 액티비티는 재생성 할까?
        1. 레이아웃을 다시 계산해야하는데 기존을 파괴하고 새로 만드는 것이 깔끔한 방법
        2. 이전 상태 유지시 레이아웃도 꼬이고 메모리 누수 위험이 커짐

---

## 2. ✅ 각 생명주기 함수 설명

### 2.1 onCreate()
- 액티비티가 처음 생성될 때 호출됨
- UI 초기화, ViewBinding, ViewModel 생성, 초기 데이터 로드 등 설정
- setContentView() 필수 호출

### 2.2 onStart()
- 화면에 보여지기 직전 호출
- 아직 포커스를 받지 않은 상태

### 2.3 onResume()
- 사용자가 실제로 상호작용할 수 있는 시점
- 포커스를 받은 상태

### 2.4 onPause()
- 다른 액티비티가 전면에 뜨기 직전 호출
- 짧은 시간 동안 처리해야 할 작업 수행

### 2.5 onStop()
- UI가 화면에서 완전히 사라진 상태
- 백그라운드로 전환됨

### 2.6 onRestart()
- 액티비티가 정지된 뒤 다시 시작될 때 호출됨

### 2.7 onDestroy()
- 액티비티가 종료될 때 호출
- finish() 또는 시스템이 메모리 회수할 때 발생

---

## 3. 🔍  생명주기 요약표

| 메서드       | 호출 시점               | 주 작업 예시                  |
| --------- | ------------------- | ------------------------ |
| onCreate  | 최초 실행 시 1회만         | 뷰 바인딩, 초기화, 뷰 모델 생성      |
| onStart   | 화면 표시 직전            | UI 업데이트, 센서 연결 등         |
| onResume  | 사용자와의 상호작용 시작       | 애니메이션 재개, 타이머 시작         |
| onPause   | 다른 액티비티 뜨기 직전       | 애니메이션 정지, 리소스 해제         |
| onStop    | 화면 완전히 가려짐          | 네트워크 중단, DB 저장 등         |
| onRestart | onStop → 다시 시작되는 경우 | 필요시 상태 복원                |
| onDestroy | 액티비티 완전 종료 시        | 모든 자원 해제, memory leak 방지 |

---

## 4.⚠️ **주의사항**

- onCreate()에 너무 많은 작업을 넣지 말기 (ANR 위험)
- onResume() → 포커스 받는 시점이므로 네트워크 호출이나 애니메이션 재개에 적합
- onStop() → 메모리 누수를 방지하기 위한 정리 작업 수행 필요
- ViewModel, LiveData, LifecycleOwner와 연계 시 더 유연한 생명주기 관리 가능

---

## 5.🧠 **결론**

- Activity 생명주기는 Android 앱 개발의 기초이자 핵심
- 각 메서드가 언제 호출되고, 어떤 작업을 하면 되는지 명확히 이해하면 안정적인 앱 개발이 가능
- LifecycleObserver, ViewModel, Jetpack 컴포넌트와 함께 사용하면 더욱 효과적입니다.