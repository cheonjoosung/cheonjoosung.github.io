---
title: Android TextInputLayout 오류 메시지 표시 방법 (setError 사용법)
tags: [ Android ]
style: fill
color: dark
description: TextInputLayout을 활용한 EditText 유효성 검사 및 오류 메시지 출력 방법을 예제와 함께 정리합니다.
---

## ✨ 개요

`TextInputLayout`은 `EditText`를 감싸는 뷰로, 에러 메시지, 힌트 애니메이션, 카운터 등을 쉽게 구현할 수 있는 Android Material Design 구성요소입니다.

이 포스팅에서는 `TextInputLayout.setError()`를 사용하여 **사용자 입력 오류 메시지를 표시하는 방법**과 **실전 사용 예제**를 소개합니다.


---

## 1. ✅ TextInputLayout 구성 예시

```xml
<com.google.android.material.textfield.TextInputLayout
        android:id="@+id/usernameInputLayout"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="사용자 이름"
        app:errorEnabled="true">

    <com.google.android.material.textfield.TextInputEditText
            android:id="@+id/usernameEditText"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"/>
    
</com.google.android.material.textfield.TextInputLayout>
```

- errorEnabled="true"를 반드시 추가해야 에러 메시지가 표시됩니다.
- 내부에 반드시 TextInputEditText를 사용해야 제대로 작동합니다.

---

## 2. ✅ 에러 메시지 숨기기

```kotlin
button.setOnClickListener {
    val username = usernameEditText.text.toString()
    if (username.isBlank()) {
        usernameInputLayout.error = "사용자 이름을 입력하세요."
    } else {
        usernameInputLayout.error = null // 에러 제거
        Toast.makeText(applicationContext, "정상 입력됨: $username", Toast.LENGTH_SHORT)
            .show()
    }
}

// 에러 메시지 숨기기
button2.setOnClickListener {
    usernameInputLayout.error = null
    usernameInputLayout.isErrorEnabled = false
}
```
- 사용자가 오류를 수정한 경우 위와 같이 에러를 제거합니다.

---

## 3. 🔍 팁

| 상황                 | 처리 방법                                        |
| ------------------ | -------------------------------------------- |
| 빈 입력일 경우           | `setError("필수 입력입니다")`                       |
| 정규식 검사 실패          | `setError("이메일 형식이 아닙니다")`                   |
| 유효할 때              | `setError(null)` 또는 `isErrorEnabled = false` |
| 포커스 이동 시 실시간 검사 적용 | `setOnFocusChangeListener` 활용 가능             |

---

## 4.🧠 **결론**

- TextInputLayout은 EditText의 기능을 확장하여 더 직관적인 UI와 오류 처리를 제공합니다.
- 특히 setError() 기능은 사용자에게 피드백을 주고 입력을 유도하는 데 매우 유용합니다.
- LiveData, TextWatcher, focus listener와 함께 사용하면 즉각적인 유효성 검사 처리도 가능해져 UX가 향상됩니다.

✅ 사용자 입력을 다루는 모든 앱에서 필수적인 구성요소입니다. 꼭 익혀두세요!