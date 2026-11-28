---
title: (Android/System) BroadcastReceiver 종류와 사용법 — 정적 vs 동적
tags: [ Android, BroadcastReceiver ]
style: fill
color: dark
description: BroadcastReceiver의 매니페스트 등록과 코드 등록 차이, Android 8 이후 암시적 브로드캐스트 제한, exported 플래그, 로컬 브로드캐스트 대체 방법을 정리합니다.
---

---

## 1. BroadcastReceiver란?

시스템이나 앱이 보내는 **이벤트(브로드캐스트)** 를 받는 컴포넌트입니다.
배터리 부족, 화면 켜짐, 네트워크 변경, 부팅 완료, 앱 내부 이벤트 등을 감지합니다.

```kotlin
class MyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // 메인 스레드, 약 10초 이내에 끝내야 함
        when (intent.action) {
            Intent.ACTION_BATTERY_LOW -> { /* ... */ }
        }
    }
}
```

> `onReceive`는 **메인 스레드에서 짧게** 끝나야 합니다. 오래 걸리면 ANR이 발생하므로 무거운 작업은 WorkManager 등에 위임하세요.

---

## 2. 정적 등록 (Manifest)

앱이 실행 중이 아니어도 시스템이 앱을 깨워 수신자를 호출합니다.

```xml
<receiver android:name=".BootReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

**Android 8.0(API 26)부터는 대부분의 암시적 브로드캐스트를 Manifest로 받을 수 없습니다.** (예: `CONNECTIVITY_CHANGE`, 화면 켜짐/꺼짐)

---

## 3. 동적 등록 (코드)

앱이 **실행 중인 동안에만** 수신하고, `registerReceiver`/`unregisterReceiver`로 직접 관리합니다.

```kotlin
class MainActivity : AppCompatActivity() {

    private val batteryReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            val level = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, -1)
            updateUi(level)
        }
    }

    override fun onStart() {
        super.onStart()
        val filter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        ContextCompat.registerReceiver(
            this, batteryReceiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED
        )
    }

    override fun onStop() {
        unregisterReceiver(batteryReceiver)      // 반드시 해제 (누수 방지)
        super.onStop()
    }
}
```

> 등록과 해제는 **짝을 맞춰야** 합니다 (`onStart`/`onStop`, `onResume`/`onPause`). 해제하지 않으면 `IntentReceiverLeaked` 경고와 메모리 누수가 생깁니다.

---

## 4. 정적 vs 동적 비교

| | 정적 (Manifest) | 동적 (코드) |
|---|----------------|-------------|
| 수신 시점 | 앱 종료 상태에서도 | 앱 실행 중에만 |
| 등록 | Manifest | `registerReceiver` |
| Android 8+ 암시적 브로드캐스트 | **대부분 불가** | 가능 |
| 해제 | 불필요 | 필요 |
| 용도 | 부팅 완료, SMS 수신 등 일부 | 배터리, 연결 상태, 화면 on/off |

**Manifest에서 여전히 가능한 예**: `BOOT_COMPLETED`, `LOCALE_CHANGED`, `ACTION_MY_PACKAGE_REPLACED`, 앱이 대상인 **명시적** 브로드캐스트 등.

---

## 5. exported 플래그 — 보안 (Android 14+ 필수)

외부 앱이 내 Receiver에 이벤트를 보낼 수 있는지를 지정합니다.

```kotlin
// 동적 등록 시 Android 14+는 플래그가 필수
ContextCompat.registerReceiver(context, receiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED)   // 앱 내부 전용
ContextCompat.registerReceiver(context, receiver, filter, ContextCompat.RECEIVER_EXPORTED)       // 외부 앱도 전송 가능
```

```xml
<!-- Manifest: Android 12+는 intent-filter가 있으면 exported를 명시해야 함 -->
<receiver android:name=".MyReceiver" android:exported="false" />
```

시스템 이벤트를 받는 경우에도 외부 앱의 **가짜 브로드캐스트**를 막으려면 가능한 `NOT_EXPORTED`를 사용하고, 받은 데이터는 검증하세요.

---

## 6. 브로드캐스트 보내기

```kotlin
// 앱 내부로만 보내기 (패키지 지정으로 다른 앱이 못 받게)
val intent = Intent("com.example.ACTION_DATA_UPDATED").apply {
    setPackage(context.packageName)
    putExtra("count", 5)
}
context.sendBroadcast(intent)

// 권한이 있는 앱만 받을 수 있게
context.sendBroadcast(intent, "com.example.permission.RECEIVE_UPDATE")
```

---

## 7. LocalBroadcastManager는 Deprecated

앱 내부 컴포넌트 간 통신용이던 `LocalBroadcastManager`는 **Deprecated**입니다. 대안은 다음과 같습니다.

```kotlin
// SharedFlow / StateFlow 기반 이벤트
object AppEvents {
    private val _events = MutableSharedFlow<AppEvent>()
    val events = _events.asSharedFlow()
    suspend fun emit(event: AppEvent) = _events.emit(event)
}

// 수신
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        AppEvents.events.collect { handle(it) }
    }
}
```

앱 내부 통신은 Flow/LiveData, 시스템 이벤트만 BroadcastReceiver를 사용하는 것이 현재 방식입니다.

---

## 8. 오래 걸리는 작업 — goAsync

```kotlin
override fun onReceive(context: Context, intent: Intent) {
    val pending = goAsync()                       // onReceive 종료 후에도 잠시 유지 (약 10초)
    CoroutineScope(Dispatchers.Default).launch {
        try { quickWork() } finally { pending.finish() }
    }
}
```

10초를 넘기는 작업은 `goAsync` 대신 **WorkManager**에 위임하세요.

---

## 9. 정리

- 정적(Manifest)은 앱이 꺼져 있어도 수신하지만 **Android 8+에서 암시적 브로드캐스트는 대부분 불가**
- 동적(코드)은 실행 중에만 수신하며 **반드시 `unregisterReceiver`** 로 해제
- Android 12+/14+ 보안 정책에 따라 `exported`와 `RECEIVER_(NOT_)EXPORTED`를 명시
- `onReceive`는 메인 스레드에서 짧게, 긴 작업은 WorkManager로
- 앱 내부 통신은 `LocalBroadcastManager`(Deprecated) 대신 **Flow**를 사용
