---
title: Android WorkManager 완벽 가이드 - 백그라운드 작업의 정석
tags: [ Android ]
style: fill
color: dark
description: Android에서 백그라운드 작업을 안전하고 효율적으로 실행하기 위한 Jetpack WorkManager의 개념, 사용 방법, 그리고 장점과 주의사항을 정리합니다
---

## ✨ 개요

WorkManager는 Jetpack 라이브러리 중 하나로, 안정적이고 지연 가능한 백그라운드 작업을 처리하기 위한 솔루션입니다. 

앱이 종료되거나 기기가 재부팅되더라도 작업을 자동으로 재시도하거나 보장된 방식으로 실행해주는 것이 특징입니다.

---

## 1. 🧪 주요 특징

- 앱이 종료되어도 작업 실행 보장 (OS 재부팅 포함)
- 작업 조건 설정 가능 (예: 네트워크 연결 상태, 충전 중일 때 등)
- API 14 이상 모든 Android 버전 지원
- Coroutine, RxJava 등과 쉽게 연동 가능

---

## 2. ✅ 사용 방법

### 2.1 의존성 추가

```kotlin
implementation "androidx.work:work-runtime-ktx:2.10.1"
```

### 2.2 MyWork 클래스

```kotlin
import android.content.Context
import android.util.Log
import androidx.work.Worker
import androidx.work.WorkerParameters

class MyWorker(appContext: Context, params: WorkerParameters) : Worker(appContext, params) {

    override fun doWork(): Result {
        // 작업 수행
        Log.d(MyWorker::class.java.simpleName, "작업 실행됨!")
        return Result.success()
    }
}
```

### 2.3 호출방법

```kotlin
// WorkRequest 생성 & 작업 예약
val workRequest = OneTimeWorkRequestBuilder<MyWorker>().build()
WorkManager.getInstance(this).enqueue(workRequest)

// 반복 작업 (주기적) 최소 15분 이상 간격만 가능
val periodicWork = PeriodicWorkRequestBuilder<MyWorker>(15, TimeUnit.MINUTES).build()
WorkManager.getInstance(this)
    .enqueueUniquePeriodicWork("my_work", ExistingPeriodicWorkPolicy.KEEP, periodicWork)

// 제약 조건 설정
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresCharging(true)
    .build()

val work = OneTimeWorkRequestBuilder<MyWorker>()
    .setConstraints(constraints)
    .build()

// 작업 체이닝
val work1 = OneTimeWorkRequestBuilder<MyWorker>().build()
val work2 = OneTimeWorkRequestBuilder<MyWorker>().build()

WorkManager.getInstance(this)
    .beginWith(work1)
    .then(work2)
    .enqueue()
```

---

## 3. ⚠️ 주의사항

- 실시간/즉시성이 중요한 작업에는 적합하지 않음
- 정확한 실행 타이밍 보장은 어려움 (지연될 수 있음)
- 최소 실행 지연 시간: 약 10분 이상 (Doze Mode 고려 시)

---

## 4. 🧾 결론

- WorkManager는 Android에서 가장 신뢰할 수 있는 백그라운드 작업 처리 도구입니다. 
- 앱 상태, OS 상태와 무관하게 작업을 보장하며, 선언적 구성과 Jetpack 연동성을 제공해 현대적인 앱 구조에 적합합니다. 
- 신중한 설계와 조건 설정을 통해 WorkManager를 제대로 활용해보세요.