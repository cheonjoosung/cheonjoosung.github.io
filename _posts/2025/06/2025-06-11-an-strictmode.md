---
title: Android StrictMode 완벽 가이드 - ANR을 막는 최고의 디버깅 도구
tags: [ Android ]
style: fill
color: dark
description: Android 앱 개발 중 ANR을 예방하고 병목 현상을 조기에 파악할 수 있는 StrictMode의 원리와 설정법을 명확하게 설명합니다.
---

## ✨ 개요

StrictMode는 Android에서 메인(UI) 스레드에서 발생할 수 있는 성능 저하 이슈를 탐지하는 디버깅 도구입니다.
개발 중 네트워크 요청, 디스크 접근, 메모리 누수와 같은 문제를 조기에 발견해 ANR(Application Not Responding) 을 예방하는 데 큰 역할을 합니다.

Android 앱의 성능을 관리하고, 사용자 경험을 보호하는 중요한 개발 도구로서 StrictMode는 반드시 이해하고 활용해야 할 기능입니다.



---

## 1. 🧪 왜 StrictMode가 중요한가?

ANR은 주로 메인 스레드가 오랜 시간 동안 블로킹될 경우 발생합니다. 
StrictMode는 이러한 병목을 개발 단계에서 실시간으로 탐지하여, ANR 발생 가능성을 줄이는 데 유용합니다.

| 주요 원인                            | 영향                                           |
|-------------------------------------|------------------------------------------------|
| 메인 스레드에서 디스크 접근             | I/O 지연으로 인해 UI 응답이 멈춤                   |
| 네트워크 요청을 메인 스레드에서 처리       | 서버 응답을 기다리는 동안 ANR 발생 가능성 존재       |
| 리소스 누수 (Activity, Cursor 등)       | GC 미작동으로 메모리 부족, 성능 저하 유발            |

---

## 2. ✅ StrictMode로 탐지할 수 있는 항목

StrictMode는 다음과 같은 상황을 실시간으로 감지합니다:
- UI 스레드에서의 디스크 읽기/쓰기 (detectDiskReads(), detectDiskWrites())
- UI 스레드에서의 네트워크 호출 (detectNetwork())
- 닫히지 않은 리소스 탐지 (detectLeakedClosableObjects())
- 메모리 누수 의심 객체 탐지 (detectActivityLeaks(), detectLeakedSqlLiteObjects() 등)

---

## 3. ⚙️ 설정 방법 및 적용 팁

- 무거운 작업은 반드시 백그라운드 스레드에서 처리 (e.g., Coroutine, AsyncTask, Executor 등)
- 네트워크 요청은 OkHttp, Retrofit 등 비동기 방식 사용
- BroadcastReceiver 최소한의 로직만 처리
- 프로파일러/StrictMode/ANR 로그 분석으로 병목 지점 확인

```kotlin
StrictMode.setThreadPolicy(
    StrictMode.ThreadPolicy.Builder()
        .detectAll()         // 모든 문제 탐지
        .penaltyLog()       // Logcat에 경고 출력
        .build()
)

StrictMode.setVmPolicy(
    StrictMode.VmPolicy.Builder()
        .detectLeakedClosableObjects()
        .penaltyLog()
        .build()
)
```
- 일반적으로 Application.onCreate() 내부에서 설정
- 테스트 중 특정 Activity에만 제한적으로 설정할 수도 있음

---

## 4.⚠️ 사용 시 주의사항

- StrictMode는 성능 개선을 위한 디버깅 도구입니다. 실제 배포 환경에서는 비활성화해야 합니다.
- Logcat에서 다량의 로그를 출력하므로, 디버그 환경에서만 사용해야 효율적입니다.
- StrictMode로 탐지된 경고를 해결하지 않으면 실제 사용자 환경에서 ANR, 메모리 누수, 앱 종료로 이어질 수 있습니다.

---

## 5. 결론

StrictMode는 Android 개발 중 메인 스레드 병목과 메모리 누수 위험을 사전에 감지할 수 있는 강력한 도구입니다.
다음과 같은 원칙을 실천하면 앱의 안정성과 성능을 크게 향상시킬 수 있습니다:

- 메인 스레드에서는 네트워크, 디스크 접근을 피한다.
- 장시간 실행되는 작업은 백그라운드에서 수행한다.
- StrictMode를 통해 문제를 사전에 감지하고 개선한다.

앱의 사용자 경험을 지키고, ANR을 방지하는 가장 좋은 습관은 StrictMode를 항상 디버그 빌드에 설정해 두는 것입니다.

