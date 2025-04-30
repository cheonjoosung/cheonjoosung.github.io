---
title: Android 화면 캡처 방지 설정 방법 (FLAG_SECURE 사용법)
tags: [Android]
style: fill
color: dark
description: 민감한 정보를 다룰 때 필요한 Android 화면 캡처 방지 설정 방법을 FLAG_SECURE로 알아봅니다
---
---

## ✨ 개요

Android 앱에서 **민감한 정보를 화면에 표시할 때**,  
사용자가 화면을 캡처하거나 녹화하지 못하도록 방지하는 보안 기능이 필요할 수 있습니다.

이를 위해 Android에서는 **`FLAG_SECURE`** 플래그를 제공합니다.


---

## 1. ✅ FLAG_SECURE란?

`FLAG_SECURE`은 Android의 보안 플래그로,  
이 플래그가 설정된 화면은 다음 기능이 **자동 차단됩니다.**

- **스크린샷 캡처 금지**
- **화면 녹화 차단**
- **앱 미리보기 화면 블러 처리 (최근 앱 목록)**

---

## 2. ✅ 사용 방법

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    // 화면 캡처 방지 설정
    window.setFlags(
        WindowManager.LayoutParams.FLAG_SECURE,
        WindowManager.LayoutParams.FLAG_SECURE
    )

    setContentView(R.layout.activity_main)
}
```

- 반드시 setContentView() 이전에 호출해야 정상 적용됩니다.
- Activity 전체가 보안 처리됩니다.

---

## 3. 🔥 특정 Fragment만 보안 적용하고 싶은 경우

### 3.1 Fragment 에서 직접 적용

```kotlin
class SecureFragment : Fragment() {

    override fun onResume() {
        super.onResume()

        // 해당 Fragment가 보일 때만 보안 적용
        requireActivity().window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
    }

    override fun onPause() {
        super.onPause()

        // 이 Fragment가 사라질 때 보안 해제
        requireActivity().window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
    }
}

```

- onResume()에서 적용, onPause()에서 해제
- 단점: Fragment 간 빠른 전환 시 간혹 깜빡이거나 적용 지연 발생 가능


### 3.2 Fragment 전환 이벤트 감지 → FLAG_SECURE 관리

Fragment를 여러 개 사용하는 구조라면, Fragment 전환을 감지해서 FLAG_SECURE를 컨트롤하는 것도 추천됩니다

```kotlin
navController.addOnDestinationChangedListener { _, destination, _ ->
    if (destination.id == R.id.sensitiveFragment) {
        window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
    } else {
        window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
    }
}
```
- Fragment 마다 코드 적용없이 한 화면에서 일괄적으로 관리 가능

---

## 4. 🔥 설정 전/후 비교

| 항목                      | FLAG_SECURE 미적용 상태             | FLAG_SECURE 적용 상태             |
|---------------------------|-------------------------------------|-----------------------------------|
| 스크린샷 캡처             | 가능 (캡처 이미지 저장됨)          | 불가능 (검은 화면 또는 무효 처리) |
| 화면 녹화                 | 가능 (전체 화면 녹화 가능)         | 대부분 실패 또는 블랙 화면 녹화   |
| 최근 앱 미리보기 화면     | 앱 화면이 그대로 미리보기로 노출   | 블랙 또는 앱 아이콘만 표시됨      |
| 민감 정보 유출 위험       | 높음                               | 매우 낮음                         |

---

## 5. 주의사항 및 결론

- 주의사항
  + 시스템 UI(예: 알림 바, 최근 앱 등)에는 영향을 주지 않습니다.
  + 일부 루팅 기기에서는 우회 가능성이 존재합니다.
  + FLAG_SECURE을 사용해도 데이터 암호화나 서버 보안을 대체할 수는 없습니다.
- 결론
  + 민감한 정보를 다루는 결제, 인증, 금융, 건강 정보 앱 등에서는 반드시 FLAG_SECURE 설정을 통해 스크린샷과 화면 녹화를 방지해야 합니다.
  + 📱 Android에서 보안 UI를 구현하려면 FLAG_SECURE는 기본 중 기본입니다.

