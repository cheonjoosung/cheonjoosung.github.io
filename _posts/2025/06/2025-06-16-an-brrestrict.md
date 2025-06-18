---
title: Android 8.0(Oreo)에서의 Broadcast 제한 사항 정리
tags: [ Android ]
style: fill
color: dark
description: Android 8.0(Oreo)부터 변경된 브로드캐스트 등록 정책과 그로 인해 발생하는 제약사항 및 대안 방안을 간단하고 명확하게 설명합니다
---

## ✨ 개요

Android 8.0 (API 26)부터 시스템 성능과 배터리 수명을 개선하기 위해 암시적 브로드캐스트(implicit broadcast) 에 대한 등록 제한이 도입되었습니다.
이는 앱이 백그라운드일 때 브로드캐스트 수신이 차단될 수 있음을 의미합니다.

---

## 1. 🚫 어떤 브로드캐스트가 제한되는가?

- AndroidManifest.xml에 등록한 암시적 브로드캐스트
- 예외 없이 제한되는 대표 인텐트 예:
  + CONNECTIVITY_CHANGE
  + ACTION_NEW_PICTURE
  + ACTION_PACKAGE_ADDED
- 허용예외
  + 명시적 브로드캐스트 (setComponent() 또는 setPackage() 사용)
  + 시스템에서 허용한 인텐트 (예: BOOT_COMPLETED, SMS_RECEIVED 등 일부)
  + 동적으로 등록한 Context.registerReceiver() 는 여전히 동작 가능 (단, 앱이 포그라운드에 있을 때만)

---

## 2. ✅ 대체 방법

- JobScheduler 또는 WorkManager 를 사용하여 백그라운드 작업 예약
- 명시적 인텐트 사용
- 앱 실행 중 동적 등록 방식으로 수신

```kotlin
val intentFilter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
registerReceiver(myReceiver, intentFilter)
```

---

## 3. ⚠️ 개발 시 주의사항

- Android 8.0 이상을 타겟팅하면 제한 자동 적용됨
- 백그라운드에서의 무분별한 브로드캐스트 사용은 앱의 정상 동작을 막을 수 있음
- 테스트 기기의 OS 버전에 따라 동작이 달라질 수 있음 → minSdkVersion과 targetSdkVersion 확인 필요

---

## 4. 🧾 결론

Android 8.0부터 도입된 브로드캐스트 제한은 배터리 효율과 시스템 자원 최적화를 위한 조치입니다. 
이에 따라 기존의 BroadcastReceiver 사용 방식에서 WorkManager, 명시적 인텐트 등으로의 전환이 필수적입니다. 
정확한 OS 정책 이해와 적절한 대안을 마련하여 호환성과 안정성을 확보해야 합니다.