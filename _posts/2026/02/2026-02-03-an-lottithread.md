---
title: (Android/안드로이드) LottieAnimation & 메인 스레드(UI Thread) 사용 정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) LottieAnimation & 메인 스레드(UI Thread) 사용 정리 - “왜 끊기나?”, “어디서 돌리나?”, “실무에서 어떻게 안전하게 쓰나”
---

## 개요

Lottie는 “벡터 애니메이션을 쉽게” 보여주지만, 막상 프로젝트에 넣으면 이런 이슈를 자주 만납니다.
- 애니메이션이 버벅이거나 멈춤
- Dialog/BottomSheet에서 dismiss 후에도 계속 도는 느낌
- start/stop을 백그라운드에서 호출하다가 스레드 오류
- 로띠 JSON이 무거운 경우 초기 로딩이 느림
- 이 글은 Lottie가 메인 스레드와 어떤 관계인지, 그리고 실무에서 성능/안정성을 어떻게 챙길지를 정리합니다.

요약
- LottieView(LottieAnimationView)는 UI View → start/stop/setAnimation 등은 메인 스레드에서 호출해야 안전
- 애니메이션 “렌더링”은 결국 **UI 프레임(Choreographer)**과 맞물림 → 메인 스레드가 막히면 끊김
- 무거운 JSON 파싱/로딩은 백그라운드에서(Lottie의 async 로딩 활용)
- Dialog/Progress에서 누수 방지: onStart/onStop 또는 onResume/onPause에서 명확히 재생/정지

---

## 1. Lottie는 “어느 스레드에서” 동작하나?

- LottieAnimationView는 View다
  - View의 상태 변경(가시성, 애니메이션 시작/정지, progress 변경)은 원칙적으로 **메인 스레드(UI thread)**에서 수행해야 합니다.
  - 백그라운드에서 호출하면 대표적으로:
    - CalledFromWrongThreadException
    - “가끔은 되는데 가끔은 안 되는” 레이스 컨디션같은 문제가 생길 수 있습니다.
- ✅ 결론: start/stop/cancel/progress 변경은 메인 스레드

---

## 2. 왜 메인 스레드가 막히면 Lottie가 끊기나?

Lottie는 결과적으로 화면에 프레임을 그려야 합니다.
- Android UI는 다음 루프를 따릅니다.
  - 메인 스레드가 프레임 타이밍에 맞춰 View를 measure/layout/draw
  - 애니메이션도 이 흐름에 포함됨
- 즉, 메인 스레드가 바쁘면 프레임 드랍 → Lottie도 프레임 드랍

흔한 원인
- JSON 파싱을 메인에서 수행
- 큰 Bitmap decode / 파일 I/O를 메인에서 수행
- WebView heavy 작업, RecyclerView bind 과부하
- 과도한 로그/동기 네트워크/DB 쿼리

---

## 3. 로딩/파싱은 메인에서 하면 안 되는가?

- “애니메이션 시작/정지”는 메인
- “리소스 로딩/파싱”은 가능한 비동기(백그라운드) 권장
  - Lottie는 composition(애니메이션 구성) 로딩을 비동기로 처리할 수 있습니다.
  - rawRes 예시
    - ```kotlin
      binding.lottieView.setAnimation(R.raw.loading)
      binding.lottieView.repeatCount = LottieDrawable.INFINITE
      binding.lottieView.playAnimation() // 메인 스레드
      ```
  - URL 로딩은 상황 따라 무겁고 실패 요인이 많음
  - 가능하면 앱 번들(assets/raw)로 포함하는 게 안정적입니다.

---

## 4. 메인 스레드 안전 호출 패턴(실무용)

“Lottie start/stop을 어디서 호출할지”가 흔한 문제라서, 아래처럼 래핑하면 안전합니다.

```kotlin
fun LottieAnimationView.safePlay() {
    if (Looper.myLooper() == Looper.getMainLooper()) {
        playAnimation()
    } else {
        post { playAnimation() }
    }
}

fun LottieAnimationView.safeStopAndClear() {
    if (Looper.myLooper() == Looper.getMainLooper()) {
        cancelAnimation()
        progress = 0f
    } else {
        post {
            cancelAnimation()
            progress = 0f
        }
    }
}
```

---

## 5. Dialog/Progress에서 Lottie를 쓸 때 “안 꺼지는 것처럼 보이는” 이유

- dismiss 되기 전에 다음 프레임이 이미 예약됨
  - 애니메이션은 프레임 콜백을 타고 돌기 때문에,
  - dismiss 타이밍과 애니메이션 프레임 스케줄이 엇갈리면 “한 박자 늦게” 보일 수 있습니다.
- View가 detach 되었는데 애니메이션을 계속 돌리는 케이스
  - Dialog lifecycle과 View lifecycle이 섞이면 재생/정지 시점이 꼬일 수 있어요.

✅ 추천: Dialog/Fragment의 라이프사이클에 맞춰 명확히 stop

```kotlin
override fun onStart() {
    super.onStart()
    binding.lottieView.safePlay()
}

override fun onStop() {
    binding.lottieView.safeStopAndClear()
    super.onStop()
}
```