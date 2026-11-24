---
title: (Android/System) Service vs IntentService vs JobIntentService 비교
tags: [ Android, Service ]
style: fill
color: dark
description: Service, IntentService, JobIntentService의 차이와 백그라운드 실행 제한 이후 지원 중단 배경, 현재 권장되는 WorkManager와 Foreground Service 선택 기준을 정리합니다.
---

---

## 1. Service란?

UI 없이 **백그라운드에서 작업을 수행**하는 컴포넌트입니다. 단, "별도 스레드"가 아니라 **메인 스레드에서 실행**되므로 오래 걸리는 작업은 직접 스레드/코루틴으로 넘겨야 합니다.

```kotlin
class MusicService : Service() {

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // 메인 스레드! 무거운 작업은 별도 스레드/코루틴에서
        scope.launch { doWork() }
        return START_STICKY            // 시스템이 종료해도 재시작 요청
    }
}
```

| 반환값 | 의미 |
|--------|------|
| `START_STICKY` | 종료 후 재시작 (intent는 null) |
| `START_NOT_STICKY` | 종료되어도 재시작하지 않음 |
| `START_REDELIVER_INTENT` | 재시작하며 마지막 intent 다시 전달 |

---

## 2. IntentService (Deprecated)

`Service` + **워커 스레드 + 작업 큐**가 내장된 클래스였습니다. `onHandleIntent`가 별도 스레드에서 순차 실행되고, 큐가 비면 자동 종료됩니다.

```kotlin
class UploadService : IntentService("UploadService") {
    override fun onHandleIntent(intent: Intent?) {
        // 별도 스레드에서 순차 처리, 끝나면 서비스 자동 종료
    }
}
```

**Android 11(API 30)에서 Deprecated.** Android 8.0 이후의 백그라운드 제한과 맞지 않기 때문입니다.

---

## 3. JobIntentService (Deprecated)

Android 8.0+의 백그라운드 서비스 제한을 우회하기 위한 **IntentService 대체재**였습니다.
- Android 8.0 이상: 내부적으로 `JobScheduler` 사용
- 그 미만: 일반 Service 사용

이 역시 **WorkManager 도입으로 Deprecated** 되었습니다.

---

## 4. 왜 사라졌나 — 백그라운드 실행 제한

```
Android 8.0 (API 26): 앱이 백그라운드일 때 startService() 호출 시 IllegalStateException
Android 12 (API 31): 백그라운드에서 Foreground Service 시작도 제한
```

배터리 소모를 줄이기 위해 시스템이 **백그라운드에서 마음대로 실행되는 서비스**를 막았습니다. 그래서 "앱이 꺼져 있어도 알아서 도는 서비스"라는 개념이 더 이상 통하지 않습니다.

---

## 5. 비교 표

| | Service | IntentService | JobIntentService |
|---|---------|---------------|------------------|
| 스레드 | 메인 | 워커(내장) | 워커 |
| 작업 큐 | 직접 구현 | 내장(순차) | 내장 |
| 종료 | 직접 `stopSelf()` | 자동 | 자동 |
| 백그라운드 제한 대응 | X | X | O (Job 기반) |
| 현재 상태 | 유효 (용도 한정) | **Deprecated** | **Deprecated** |

---

## 6. 현재 권장 — 작업 성격별 선택

```
앱이 꺼져도 보장되어야 하는 비동기 작업 (업로드, 동기화, 정리)
        → WorkManager

사용자가 인지하는 장시간 작업 (음악 재생, 내비게이션, 운동 기록, 다운로드)
        → Foreground Service (알림 필수)

앱이 실행 중일 때의 비동기 작업
        → 코루틴 (viewModelScope, lifecycleScope)

정확한 시각에 실행 (알람, 캘린더)
        → AlarmManager (+ 정확한 알람 권한)

다른 컴포넌트에 기능 제공 (바운드 서비스)
        → Bound Service
```

```kotlin
// IntentService 대신 WorkManager
class UploadWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        upload()
        return Result.success()
    }
}

WorkManager.getInstance(context).enqueue(OneTimeWorkRequestBuilder<UploadWorker>().build())
```

---

## 7. Service가 여전히 필요한 경우

- **Foreground Service**: 음악 플레이어, 위치 추적, 통화 등 지속 작업 (알림 필수)
- **Bound Service**: 다른 컴포넌트가 연결해 메서드를 호출 (`bindService`)
- **MessagingService 등 시스템 연동**: `FirebaseMessagingService`, `AccessibilityService`, `NotificationListenerService`

---

## 8. 정리

- `Service`는 메인 스레드에서 실행되며 직접 스레드 처리가 필요
- `IntentService`와 `JobIntentService`는 **Deprecated** — 새 코드에서 사용하지 않음
- 백그라운드 제한(Android 8+) 때문에 "알아서 도는 서비스"는 불가능
- 보장되는 비동기 작업은 **WorkManager**, 사용자가 인지하는 장시간 작업은 **Foreground Service**
- 앱이 실행 중일 때의 작업은 코루틴으로 충분
