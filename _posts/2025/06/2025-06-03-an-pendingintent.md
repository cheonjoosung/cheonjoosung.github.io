---
title: Android PendingIntent 가이드 - 개념부터 실전 사용까지
tags: [ Android ]
style: fill
color: dark
description: Android의 컴포넌트 간 안전한 작업 위임을 위한 PendingIntent의 개념과 생성 방법, 사용 예시, 보안 이슈까지 한 번에 정리합니다
---

## ✨ 개요

Android에서 PendingIntent는 다른 앱이나 시스템이 앱의 권한으로 특정 작업을 수행할 수 있도록 허용하는 메커니즘입니다. 

예를 들어, 알림 클릭 시 액티비티 실행, 알람 매니저를 통해 예약된 작업 실행 등 다양한 곳에서 사용됩니다.

> “Intent를 캡슐화하여 나중에 다른 컴포넌트(시스템 또는 외부 앱)가 실행할 수 있도록 위임하는 객체”

---

## 1. ✅ PendingIntent가 필요한 이유

- 알림(Notification), AlarmManager, AppWidget, Broadcast 등에서 앱 컴포넌트 호출 시 필요
- 시스템이나 다른 앱이 나 대신 인텐트를 실행해야 할 때 사용
- 보안을 위해 명시적으로 권한을 위임

---

## 2. ✅ PendingIntent의 기본 구성 요소

- Context: 생성 시 현재 앱의 컨텍스트
- Intent: 실행할 대상 컴포넌트를 지정하는 명시적 또는 암시적 인텐트
- requestCode: 여러 PendingIntent 구분을 위한 식별자
- flags: 기존 PendingIntent 재사용 여부, 보안 설정 등 지정

---

## 3. 🔍 PendingIntent 생성 메서드 & Flag 설명

- 주요 메소드
  + getActivity()
  + getService()
  + getBroadcast()

- 주요 Flags
  + FLAG_UPDATE_CURRENT: 기존 인텐트를 업데이트
  + FLAG_CANCEL_CURRENT: 기존 PendingIntent 취소 후 새로 생성
  + FLAG_IMMUTABLE: Android 12 이상 필수, 인텐트 수정 불가
  + FLAG_ONE_SHOT: 한 번만 사용 가능한 PendingIntent

---

## 4.🧩 **기본 사용법**

### 4.1 AlarmManager

```kotlin
// 권한필요 <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
val alarmManager = getSystemService(ALARM_SERVICE) as AlarmManager
val intent = Intent(context, AlarmReceiver::class.java)
val pendingIntent = PendingIntent.getBroadcast(context, 0, intent, FLAG_IMMUTABLE)
alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerTime, pendingIntent)
```

### 4.2 Notification

```kotlin
val intent = Intent(this, TargetActivity::class.java)
val pendingIntent = PendingIntent.getActivity(this, 0, intent, FLAG_IMMUTABLE)

val notification = NotificationCompat.Builder(this, "CHANNEL_ID")
    .setContentTitle("알림 제목")
    .setContentIntent(pendingIntent)
    .setAutoCancel(true)
    .build()
```

## 5.⚠️ **주의사항**

- requestCode는 항상 고유하게 설정
- 동일한 인텐트라도 flags와 requestCode가 다르면 별도 취급
- 오래된 PendingIntent 무효화 필요시 cancel() 호출
- Android 12(S)부터 FLAG_IMMUTABLE 또는 FLAG_MUTABLE 중 하나 반드시 명시
- Android 13(T): ForegroundService에 대한 제한 강화됨

---

## 6.🧠 **결론**

- PendingIntent는 Android 컴포넌트 간 작업을 안전하게 위임하는 강력한 도구
- 사용 목적에 따라 적절한 생성 메서드와 플래그 조합을 선택하는 것이 중요
- 보안 관점에서 FLAG_IMMUTABLE 지정은 필수이며, 최신 버전 정책을 반영해야 함