---
title: (Android/System) AlarmManager vs WorkManager 선택 기준
tags: [ Android, WorkManager ]
style: fill
color: dark
description: AlarmManager와 WorkManager의 역할 차이와 정확한 알람 권한, Doze 모드 영향, 제약 조건과 재시도 정책을 비교해 상황별 선택 기준을 정리합니다.
---

---

## 1. 핵심 차이

| | AlarmManager | WorkManager |
|---|-------------|-------------|
| 목적 | **정확한 시각**에 실행 | **언젠가 반드시** 실행 (보장) |
| 실행 시점 | 지정한 시각 (정확/비정확) | 시스템이 조건 맞을 때 최적 시점 |
| 앱 종료/재부팅 | 재부팅 시 사라짐 (재등록 필요) | **재부팅 후에도 유지** |
| 제약 조건 | 없음 | 네트워크, 충전, 저전력 등 |
| 재시도/체이닝 | 없음 | 백오프 재시도, 작업 연결 |
| 대표 용도 | 알람 시계, 일정 알림 | 업로드, 동기화, 로그 전송 |

> **질문 하나로 결정**: "정확히 그 시각에 사용자에게 보여야 하는가?" → AlarmManager, "나중에라도 꼭 실행되면 되는가?" → WorkManager.

---

## 2. AlarmManager

```kotlin
val alarmManager = context.getSystemService(AlarmManager::class.java)
val pendingIntent = PendingIntent.getBroadcast(
    context, REQUEST_CODE,
    Intent(context, AlarmReceiver::class.java),
    PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
)

// 정확한 알람 (Doze 중에도 실행)
alarmManager.setExactAndAllowWhileIdle(
    AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent
)

// 대략적인 알람 (배터리 절약, 시스템이 묶어서 실행)
alarmManager.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAtMillis, pendingIntent)

// 취소 (같은 PendingIntent 필요)
alarmManager.cancel(pendingIntent)
```

| 종류 | 설명 |
|------|------|
| `RTC` / `RTC_WAKEUP` | 벽시계 시간 기준 (WAKEUP은 기기를 깨움) |
| `ELAPSED_REALTIME(_WAKEUP)` | 부팅 후 경과 시간 기준 (시간대/시계 변경 영향 없음) |

---

## 3. 정확한 알람 권한 (Android 12+)

```xml
<!-- 알람/캘린더 같은 핵심 기능일 때 (Play 정책상 제한적 허용) -->
<uses-permission android:name="android.permission.USE_EXACT_ALARM" />

<!-- 사용자가 설정에서 허용해야 하는 방식 -->
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
```

```kotlin
if (Build.VERSION.SDK_INT >= 31 && !alarmManager.canScheduleExactAlarms()) {
    // 설정 화면으로 안내
    startActivity(Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM))
} else {
    scheduleExact()
}
```

`USE_EXACT_ALARM`은 알람 시계/캘린더 앱처럼 **정확한 시각이 핵심 기능**인 경우에만 Play에서 허용됩니다. 그 외 앱은 `SCHEDULE_EXACT_ALARM`(사용자 허용) 또는 비정확 알람을 사용하세요.

---

## 4. 재부팅 후 재등록

AlarmManager의 알람은 **기기를 재부팅하면 사라집니다.**

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

<receiver android:name=".BootReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED) {
            rescheduleAlarms(context)      // 저장해 둔 알람을 다시 등록
        }
    }
}
```

---

## 5. WorkManager

```kotlin
class SyncWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result =
        try {
            repository.sync()
            Result.success()
        } catch (e: IOException) {
            Result.retry()                 // 백오프 정책에 따라 재시도
        }
}

val request = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)    // Wi-Fi일 때만
            .setRequiresCharging(true)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
    .build()

WorkManager.getInstance(context).enqueueUniqueWork("sync", ExistingWorkPolicy.KEEP, request)
```

### 주기 작업

```kotlin
val periodic = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS).build()
WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork("hourly-sync", ExistingPeriodicWorkPolicy.KEEP, periodic)
```

> 주기 작업의 **최소 간격은 15분**이며, 정확한 시각에 실행되지 않고 시스템이 유연하게 배치합니다. "매일 오전 9시 정각에 알림"은 WorkManager로 보장할 수 없습니다.

---

## 6. Doze 모드의 영향

기기가 오래 정지해 있으면 시스템이 **Doze 모드**로 진입해 작업을 유예합니다.

| 방식 | Doze 중 동작 |
|------|-------------|
| `setExact()` | 지연될 수 있음 |
| `setExactAndAllowWhileIdle()` | 실행됨 (제한적 빈도) |
| WorkManager | 유지보수 창(maintenance window)에 묶어 실행 |

배터리와 직결되므로 **정말 정확해야 하는 경우에만** 정확한 알람을 쓰세요.

---

## 7. 상황별 선택

| 요구사항 | 선택 |
|---------|------|
| 알람 시계, 약 복용 알림 | AlarmManager (정확한 알람) |
| 캘린더 일정 10분 전 알림 | AlarmManager |
| 로그/분석 데이터 업로드 | WorkManager |
| 백그라운드 동기화 (1시간 주기) | WorkManager 주기 작업 |
| Wi-Fi 연결 시 대용량 업로드 | WorkManager + 제약 조건 |
| "5분 뒤 이 작업" (앱 실행 중) | 코루틴 `delay` |
| 정각에 사용자에게 알림이 꼭 보여야 함 | AlarmManager + 알림 |

---

## 8. 함께 쓰는 패턴

```
AlarmManager (정확한 시각에 깨움)
     → BroadcastReceiver 짧게 실행
     → WorkManager에 실제 무거운 작업 위임
```

알람 수신자는 약 10초 안에 끝나야 하므로, 시각은 AlarmManager가, **작업 실행은 WorkManager**가 맡는 조합이 안정적입니다.

---

## 9. 정리

- AlarmManager는 **정확한 시각**, WorkManager는 **실행 보장**
- 정확한 알람은 Android 12+에서 **권한이 필요**하고 재부팅 시 **재등록**해야 함
- WorkManager는 제약 조건, 백오프 재시도, 재부팅 후 유지를 지원하지만 시각은 보장하지 않음
- 주기 작업 최소 간격은 15분, Doze에서는 유예됨
- 알람이 깨우고 WorkManager가 처리하는 **조합**도 자주 사용
