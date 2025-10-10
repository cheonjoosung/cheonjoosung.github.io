---
title: Android targetSdkVersion vs compileSdkVersion 차이 완벽 정리 (+minSdk, 설정 팁)
tags: [Android]
style: fill
color: dark
description: Android targetSdkVersion vs compileSdkVersion 차이 완벽 정리 (+minSdk, 설정 팁)
---

## ✨ 개요

`minSdk`, `targetSdk`, `compileSdk`는 빌드/호환성에 핵심인 설정이지만 헷갈리기 쉽습니다.  
이 글에서는 targetSdk와 compileSdk의 역할/영향/베스트 프랙티스를 중심으로 한 번에 정리합니다.

---

## 요약

- **compileSdkVersion**: **앱을 “무엇으로 컴파일”할지**.  
  최신 API를 **코드에서 사용**할 수 있게 해 주는 **컴파일 타임 상한선**.
- **targetSdkVersion**: **앱이 “어떤 OS 버전을 목표로” 설계되었는지**.  
  그 버전부터 **새 동작(Behavior Change)** 적용이 **의도적으로 켜진다**.

> 둘은 같게 맞추는 것이 일반적이지만 **의미가 다릅니다**.  
> `compileSdk`는 “SDK 헤더”, `targetSdk`는 “런타임 정책/동작”에 영향.

---

## 1. 세 가지 설정의 역할

### ✅ minSdkVersion
- **최소 실행 가능 버전**(최저 호환 API 레벨)
- 이보다 낮은 기기에서는 설치/실행 불가

### ✅ compileSdkVersion
- **앱을 컴파일할 때 사용하는 Android SDK 버전**
- 이 버전의 **API, 상수, annotation, deprecation 정보**를 컴파일러가 인식
- **런타임 동작에는 직접 영향 없음**
- 낮으면 최신 API 심볼을 **import조차 못 함** (컴파일 에러)

### ✅ targetSdkVersion
- **앱이 검증/최적화된 대상 OS 버전**
- 이 값 이상에서 도입된 **행동 변경(behavior changes)** 이 **활성화**됨  
  (예: 권한, 백그라운드 제한, 스토리지 정책 등)
- Play 정책은 **주기적으로 targetSdk 상향을 요구**함

---

## 2. compileSdk가 낮으면 생기는 문제

- 최신 AndroidX/Jetpack 라이브러리가 **최신 compileSdk**를 요구하는 경우가 많음
- 새로운 `Manifest` 속성, 퍼미션 상수, API 심볼을 **인식하지 못해 컴파일 실패**
- 해결: **`compileSdk`는 항상 최신 안정 버전**로 유지하는 것이 안전

> 컴파일은 최신으로, 런타임 호환은 `minSdk`/조건부 코드로 해결하는 게 기본 전략.

---

## 3. targetSdk가 낮으면 생기는 문제

- 최신 OS에서 **레거시 호환 모드**로 동작 → 일부 신규 제한이 면제되기도 하지만
- **Play 스토어 게시/업데이트 제한**에 걸릴 수 있음 (정책 상향 요구)
- 시스템이 기대하는 최신 보안/프라이버시 동작을 따르지 않아 **예상치 못한 동작 차이** 발생

> **출시/유지보수 중인 앱은 targetSdk 상향이 필수 과제**입니다.

---

## 4. 런타임 동작 예시 (왜 targetSdk가 중요?)

| 항목 | 낮은 targetSdk | 높은 targetSdk(해당 버전 이상) |
|---|---|---|
| 포그라운드 서비스, 백그라운드 제한 | 느슨 | **더 엄격한 제한 적용** |
| 권한 모델 변경(예: 알림 권한) | 기존 동작 | **새 권한 요구** |
| 저장소 정책(Scoped Storage) | 레거시 경로 허용 | **스코프드 스토리지 강제** |
| Job/Alarm, 배터리 최적화 | 기존 정책 | **새 정책/제약 적용** |

> 즉, **동일 APK라도 targetSdk에 따라 런타임 정책이 달라질 수 있음**.

---

## 5. 마이그레이션 체크리스트 (targetSdk 상향 시)

- 릴리즈 노트/Behavior Changes 읽기 (해당 OS 버전)
- 권한 변경: 알림/미디어 접근/백그라운드 위치 등 플로우 갱신
- 백그라운드 제한: 서비스, 브로드캐스트, 작업 스케줄 영향 점검
- 스토리지: Scoped Storage, SAF/MediaStore 경로 재검토
- 암호화/네트워크: TLS/암호 스위트, Cleartext 정책 확인
- 테스트 매트릭: 실제 기기 + 에뮬레이터(해당 OS) 회귀 테스트
- 서드파티 SDK 업데이트: 최신 버전 적용 (종종 target 요구 있음)

---

## 6. compileSdk = “어떤 SDK로 컴파일할지” (코드 가능 범위)

- targetSdk = “어떤 OS 버전을 목표로 동작 검증했는지” (런타임 동작/정책)
- 실무 권장: compileSdk 최신, targetSdk 정책 요구치 충족, minSdk는 비즈니스 타깃에 맞추기.
- 상향 시에는 행동 변경 목록을 기준으로 기능 별 회귀 테스트를 반드시 수행하세요.