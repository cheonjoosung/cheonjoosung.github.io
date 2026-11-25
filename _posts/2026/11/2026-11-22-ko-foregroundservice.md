---
title: (Android/System) Foreground Service 구현 — 알림과 함께
tags: [ Android, Service ]
style: fill
color: dark
description: Foreground Service의 시작 방법과 필수 알림, foregroundServiceType, 권한, Android 12 이후 시작 제한과 종료 처리를 코드로 정리합니다.
---

---

## 1. Foreground Service란?

사용자가 **진행 중임을 인지하는 작업**(음악 재생, 길 안내, 파일 다운로드)을 위한 서비스입니다.
**상태바 알림을 반드시 표시**해야 하며, 대신 시스템이 앱이 백그라운드로 가도 쉽게 종료하지 않습니다.

---

## 2. Manifest 설정

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<!-- Android 14+: 서비스 유형별 권한 -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
<!-- Android 13+: 알림을 보이려면 런타임 권한 필요 -->
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

<service
    android:name=".TrackingService"
    android:exported="false"
    android:foregroundServiceType="location" />
```

> **Android 14(API 34)부터 `foregroundServiceType`이 필수**입니다. 유형(`location`, `mediaPlayback`, `dataSync`, `camera`, `microphone` 등)에 맞는 권한도 함께 선언해야 합니다.

---

## 3. 서비스 구현

```kotlin
class TrackingService : Service() {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    override fun onBind(intent: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START -> start()
            ACTION_STOP -> stop()
        }
        return START_NOT_STICKY
    }

    private fun start() {
        createChannel()
        val notification = buildNotification("위치 기록 중")

        // startForeground는 startForegroundService 호출 후 5초 이내에 호출해야 함
        ServiceCompat.startForeground(
            this, NOTIFICATION_ID, notification,
            ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION     // Manifest의 type과 일치
        )

        scope.launch {
            while (isActive) {
                // 실제 작업
                delay(5_000)
            }
        }
    }

    private fun stop() {
        scope.cancel()
        stopForeground(STOP_FOREGROUND_REMOVE)    // 알림 제거
        stopSelf()
    }

    override fun onDestroy() {
        scope.cancel()
        super.onDestroy()
    }

    private fun buildNotification(text: String): Notification {
        val stopIntent = PendingIntent.getService(
            this, 0,
            Intent(this, TrackingService::class.java).setAction(ACTION_STOP),
            PendingIntent.FLAG_IMMUTABLE
        )
        val contentIntent = PendingIntent.getActivity(
            this, 0, Intent(this, MainActivity::class.java), PendingIntent.FLAG_IMMUTABLE
        )
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_tracking)
            .setContentTitle("트래킹")
            .setContentText(text)
            .setContentIntent(contentIntent)
            .addAction(R.drawable.ic_stop, "중지", stopIntent)
            .setOngoing(true)
            .build()
    }

    private fun createChannel() {
        val channel = NotificationChannel(CHANNEL_ID, "트래킹", NotificationManager.IMPORTANCE_LOW)
        getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
    }

    companion object {
        const val ACTION_START = "action.START"
        const val ACTION_STOP = "action.STOP"
        private const val CHANNEL_ID = "tracking"
        private const val NOTIFICATION_ID = 1001
    }
}
```

---

## 4. 서비스 시작

```kotlin
val intent = Intent(context, TrackingService::class.java).setAction(TrackingService.ACTION_START)
ContextCompat.startForegroundService(context, intent)    // 앱이 포그라운드일 때 호출
```

`startForegroundService()` 후 **5초 안에 `startForeground()`** 를 호출하지 않으면 `ForegroundServiceDidNotStartInTimeException`으로 크래시합니다. `onStartCommand` 시작 부분에서 바로 호출하세요.

---

## 5. 알림 권한 (Android 13+)

```kotlin
val launcher = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
    if (granted) startTracking()
}

if (Build.VERSION.SDK_INT >= 33 &&
    ContextCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS) != PERMISSION_GRANTED
) {
    launcher.launch(Manifest.permission.POST_NOTIFICATIONS)
} else {
    startTracking()
}
```

알림 권한이 거부되어도 Foreground Service는 동작하지만, 알림이 **작업 관리자(FGS Task Manager)** 에서만 보이게 됩니다.

---

## 6. 시작 제한 (Android 12+)

```
앱이 백그라운드일 때 startForegroundService() 호출 → ForegroundServiceStartNotAllowedException
```

**예외(허용되는 경우)**: 사용자가 알림/위젯의 버튼을 누름, 정확한 알람 수신, FCM 높은 우선순위 메시지, 앱이 보이는 상태 등. 백그라운드에서 시작해야 하면 **WorkManager의 `setForeground()`** 를 사용하세요.

```kotlin
class DownloadWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        setForeground(createForegroundInfo())      // Worker를 Foreground로 승격
        download()
        return Result.success()
    }
}
```

---

## 7. 서비스 유형 선택

| type | 용도 |
|------|------|
| `mediaPlayback` | 음악/동영상 재생 |
| `location` | 위치 추적/내비게이션 |
| `dataSync` | 데이터 전송 (Android 15에서 6시간 제한) |
| `camera` / `microphone` | 카메라/녹음 |
| `health` | 운동/건강 |
| `shortService` | 짧은 작업(약 3분) |

사용 목적과 맞지 않는 type은 **Play 정책 위반과 심사 거절** 사유가 됩니다.

---

## 8. 주의사항

- **알림을 제거할 수 없는 작업**이므로 진짜 필요할 때만 사용하세요 (남용하면 사용자가 앱을 삭제)
- 작업이 끝나면 **반드시 `stopSelf()`** 로 종료해 알림이 남지 않게 하세요
- 서비스는 메인 스레드이므로 무거운 작업은 코루틴/스레드에서 실행
- 배터리 최적화, 제조사별 백그라운드 제한은 별도로 테스트가 필요합니다

---

## 9. 정리

- Foreground Service = **알림 필수 + 시스템이 쉽게 종료하지 않는** 서비스
- Android 14+는 `foregroundServiceType`과 type별 권한이 필수
- `startForegroundService()` 후 **5초 내 `startForeground()`**
- Android 13+는 `POST_NOTIFICATIONS`, Android 12+는 백그라운드 시작 제한
- 백그라운드에서 시작이 필요하면 WorkManager `setForeground()`, 끝나면 `stopSelf()`
