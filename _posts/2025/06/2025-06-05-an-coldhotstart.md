---
title: Android 앱 시작 정리 - Cold Start & Warm Start & Hot Start ?
tags: [ Android ]
style: fill
color: dark
description: Android 앱의 시작 유형인 Cold, Warm, Hot Start의 차이점을 명확히 설명하고, 각 방식의 특징과 최적화 방법을 소개합니다
---

## ✨ 개요

- 사용자가 앱을 실행할 때마다 Android는 앱의 상태에 따라 서로 다른 방식으로 앱을 시작합니다. 
- 이 시작 방식은 앱의 체감 속도, UX 품질, 리소스 최적화에 직접적인 영향을 미칩니다.
- 대표적인 시작 유형 Cold Start, Warm Start, Hot Start 3가지가 있습니다.

---

## 1. ✅ Cold Start (콜드 스타트)

- 앱 프로세스가 메모리에 존재하지 않을 때 발생
- 완전히 새로운 인스턴스가 생성되며, Application과 Activity가 모두 초기화됨
- 가장 많은 시간과 리소스를 소비


- 예시 상황: 
  + 기기 부팅 후 첫 실행
  + 앱이 시스템에 의해 강제 종료된 후 재실행


- 수행 과정:
  + 앱 프로세스 생성
  + Application 클래스 초기화
  + 첫 번째 Activity의 onCreate(), onStart(), onResume() 호출


- 최적화 팁:
  + SplashScreen API 사용
  + setContentView() 전에 무거운 로직 지양
  + View 초기화는 비동기로 처리 (예: Coroutine, post {} 등)

---

## 2. ✅ Hot Start (핫 스타트)

- 앱 프로세스와 Activity 인스턴스가 모두 메모리에 존재할 때 발생
- UI만 다시 보여주면 되는 수준으로 매우 빠름


- 예시 상황:
  + 최근 앱 목록에서 바로 복귀
  + 홈 화면 → 앱 복귀 시 (특정 시간 내)


- 수행 과정:
  + onRestart(), onStart(), onResume() 호출
  + Application 및 Activity 인스턴스 그대로 유지


- 특징:
  + 앱 실행 속도 최상
  + 사용자 입장에서는 거의 딜레이를 못 느낌

---

## 3. ✅ Warm Start (웜 스타트)

- 앱 프로세스는 살아있지만 Activity는 새로 생성되어야 할 때 발생
- Activity의 onCreate()가 재호출됨


- 예시 상황:
  + 사용자가 앱을 백그라운드로 보내고 일정 시간이 지난 뒤 복귀
  + 메모리 압박으로 Activity가 제거되었지만 프로세스는 유지된 경우


- 특징:
  + Application은 재실행되지 않음
  + Cold Start보다는 빠르지만 Hot Start보다는 느림

---

## 4.🧠 **결론**

- Cold / Warm / Hot Start의 차이를 이해하면 앱 시작 속도에 대한 디버깅과 UX 최적화에 큰 도움이 됩니다.
- 특히 Cold Start를 줄이는 것은 사용자 첫 경험에 있어 매우 중요합니다.
- Android 12부터 도입된 SplashScreen API 활용으로 Cold Start UX를 개선할 수 있습니다.