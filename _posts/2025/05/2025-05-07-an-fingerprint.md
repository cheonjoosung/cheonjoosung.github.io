---
title: Android BiometricPrompt로 지문/Face 인증 구현하기 (앱 잠금 보안 처리)
tags: [Android]
style: fill
color: dark
description: Android BiometricPrompt API를 이용해 지문/Face 인증으로 앱 잠금 처리하는 방법을 예제와 함께 소개합니다.
---

## ✨ 개요

대부분 앱에서는 사용자 인증을 강화하기 위해 **생체 인증(Biometric Authentication)** 을 자주 사용합니다.  
Android에서는 Jetpack의 `BiometricPrompt` API를 통해 지문, 얼굴, PIN 등 **시스템 UI 기반 인증 처리**가 가능합니다.

---

## 1. ✅ 의존성 추가 & 조건

### 1.1 의존성

`build.gradle.kts` 또는 `build.gradle`에 아래를 추가하세요.

```kotlin
// Material Components
implementation("androidx.biometric:biometric:1.1.0")
```

### 1.2 사용 전 사전 조건

- Android 6.0 (API 23) 이상
- AndroidX Biometric 라이브러리 필요
- 기기에 생체(지문/얼굴) 인증 등록되어 있어야 함

---

## 2. ✅ 지문 인증 실행 코드

### 2.1 인증 코드

```kotlin
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import androidx.biometric.BiometricManager
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import com.example.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = MainActivity::class.java.simpleName

    private lateinit var biometricPrompt: BiometricPrompt
    private lateinit var promptInfo: BiometricPrompt.PromptInfo

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // 생체 인증 초기화
        initBioAuth()

        with(binding) {
            // 생체 인증 요청
            bioAuthButton.setOnClickListener {
                biometricPrompt.authenticate(promptInfo)
            }

            // 생체 인증 가능 여부
            bioAuthCheckSupportButton.setOnClickListener {
                checkBiometricSupport()
            }
        }

    }

    private fun initBioAuth() {
        biometricPrompt = BiometricPrompt(
            this, ContextCompat.getMainExecutor(this),
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    super.onAuthenticationSucceeded(result)
                    binding.bioAuthResultTextView.text = "✅ 인증 성공! 환영합니다"
                }

                override fun onAuthenticationFailed() {
                    super.onAuthenticationFailed()
                    binding.bioAuthResultTextView.text = "❌ 인증 실패"
                }

                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    super.onAuthenticationError(errorCode, errString)
                    val result = "⚠ 오류: errCode=$errorCode errString=$errString"
                    binding.bioAuthResultTextView.text = result
                }
            })

        promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("생체 인증")
            .setSubtitle("민감 정보 보호를 위해 인증이 필요합니다.")
            .setNegativeButtonText("취소")
            .build()
    }

    private fun checkBiometricSupport() {
        val biometricManager = BiometricManager.from(this)

        val result =
            when (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_WEAK)) {
                BiometricManager.BIOMETRIC_SUCCESS -> {
                    "생체 인증 사용 가능"
                }

                BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> {
                    "기기에 생체 인증 하드웨어 없음"
                }

                BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> {
                    "생체 센서 사용 불가"
                }

                BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> {
                    "생체 등록되어 있지 않습니다."
                }

                else -> {
                    "기타 오류"
                }
            }

        binding.bioAuthResultTextView.text = result
    }

}
```

- 생체인증
  + 지문/얼굴 2개 모두 등록된 경우 하드웨어 프롬프트에서 탭 으로 구분하여 인증 가능
  + 지문/얼굴 중 1개만 등록된 경우 등록된 생체인증만 수행 가능

- 생체 기기 체크
  + BIOMETRIC_STRONG	고급 생체 인증 (지문, 얼굴)
  + BIOMETRIC_WEAK	약한 생체 인증 (일부 얼굴 인식, 단순 센서 등)
  + DEVICE_CREDENTIAL	패턴, PIN, 비밀번호 등 기기 잠금 방식
  + BIOMETRIC_STRONG or DEVICE_CREDENTIAL	가장 많이 사용되는 안전한 조합

### 2.2 layout 코드

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   xmlns:tools="http://schemas.android.com/tools"
                                                   android:id="@+id/main"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="match_parent"
                                                   tools:context=".MainActivity">

    <Button
            android:id="@+id/bioAuthButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="생체 인증 요청"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

    <Button
            android:id="@+id/bioAuthCheckSupportButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="생체 인증 가능 여부 요청"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@id/bioAuthButton" />


    <TextView
            android:id="@+id/bioAuthResultTextView"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:hint="생체 인증 가능 여부 버튼을 눌러주세요!!"
            android:text=""
            android:textSize="18sp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@id/bioAuthCheckSupportButton" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## 3.✅ BiometricPrompt 요약 및 정리

| 항목                  | 설명                                                                                 |
|-----------------------|--------------------------------------------------------------------------------------|
| ✅ 지원 대상           | Android 6.0 (API 23) 이상, 지문/얼굴 등 생체 인증 등록된 기기                      |
| 🧩 다이얼로그 자동 처리 | 시스템 제공 인증 UI 사용 → 일관된 사용자 경험 제공                                 |
| 📱 인증 수단 분류       | BIOMETRIC_WEAK (지문, 얼굴), BIOMETRIC_STRONG, DEVICE_CREDENTIAL(PIN/패턴 등)     |
| 🔁 콜백 처리           | 성공(onAuthenticationSucceeded), 실패, 에러 콜백 제공                              |
| 🔐 인증 강도 설정       | `.setAllowedAuthenticators()`로 지문/PIN 등 인증 방식 선택 가능                     |
| 🚫 제한 상황 처리       | 인증 하드웨어 없음, 등록 안 됨 등의 상황을 BiometricManager로 사전 확인 가능        |
| 🔓 앱 잠금 활용         | 설정 화면, 민감 정보 진입 시 인증 걸어두기 용도로 많이 사용                        |
| 🛠 유지보수성           | AndroidX Jetpack 구성 → 향후 업데이트에도 안정적인 호환 가능                        |

--- 

## 4.🧠 결론

- BiometricPrompt는 안드로이드에서 가장 안전하고 표준적인 생체 인증 방법입니다.
- UI 구성도 간단하고 시스템과 통합되어 보안성도 높습니다.
- 사용자 경험 개선 + 민감 정보 보호까지 동시에 챙길 수 있어 적극 활용을 추천합니다.