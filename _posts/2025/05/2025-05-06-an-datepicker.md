---
title: Android MaterialDatePicker로 날짜 선택 다이얼로그 만들기
tags: [Android]
style: fill
color: dark
description: MaterialDatePicker를 활용한 날짜 선택 다이얼로그 구현 방법과 사용 예시를 소개합니다. 간단한 날짜 포맷 처리 방법도 함께 알아보세요.
---

## ✨ 개요

앱에서 날짜 선택 기능이 필요한 경우 `MaterialDatePicker`는 가장 모던하고 직관적인 방법입니다.  
구글의 **Material Components 라이브러리**를 기반으로 하며, 간단하게 다이얼로그 형태로 구현할 수 있습니다.

> 생년월일, 예약 날짜, 달력 선택 등 다양한 UI에서 활용됩니다.

---

## 1. ✅ 의존성 추가

`build.gradle.kts` 또는 `build.gradle`에 아래를 추가하세요.

```kotlin
// Material Components
implementation("com.google.android.material:material:1.11.0")
```

---

## 2. ✅ Date Picker 사용법

### 2.1 기본 사용법

```kotlin
import com.google.android.material.datepicker.MaterialDatePicker
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

val datePicker =
    MaterialDatePicker.Builder.datePicker()
        .setTitleText("날짜를 선택하세요")
        .setSelection(MaterialDatePicker.todayInUtcMilliseconds())
        .build()

datePicker.show(supportFragmentManager, "date_picker")

datePicker.addOnPositiveButtonClickListener { selection ->
    // 선택된 날짜: milliseconds (Long)
    val selectedDate = Date(selection)
    val format = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
    val formatted = format.format(selectedDate)

    binding.textViewDate.text = formatted
}
```

### 2.2 커스텀 초기 날짜 지정

```kotlin
import com.google.android.material.datepicker.MaterialDatePicker
import java.text.SimpleDateFormat
import java.util.Calendar
import java.util.Date
import java.util.Locale

private fun showDatePickerWithCustom() {
    val calendar = Calendar.getInstance().apply {
        set(2022, Calendar.JANUARY, 1)
    }

    val datePicker = MaterialDatePicker.Builder.datePicker()
        .setTitleText("생년월일 선택")
        .setSelection(calendar.timeInMillis)
        .build()


    datePicker.show(supportFragmentManager, "date_picker")

    datePicker.addOnPositiveButtonClickListener { selection ->
        // 선택된 날짜: milliseconds (Long)
        val selectedDate = Date(selection)
        val format = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
        val formatted = format.format(selectedDate)

        Log.e(TAG, "formatted=$formatted")
    }

}
```

### 2.3 날짜 범위 선택 (DateRangePicker)

```kotlin
import com.google.android.material.datepicker.MaterialDatePicker
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

private fun showDatePickerWithRange() {
    val dateRangePicker = MaterialDatePicker.Builder.dateRangePicker()
        .setTitleText("날짜 범위를 선택하세요")
        .build()

    dateRangePicker.show(supportFragmentManager, "date_range_picker")

    dateRangePicker.addOnPositiveButtonClickListener { selection ->
        val startMillis = selection.first
        val endMillis = selection.second

        val dateFormat = SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
        val startDate = dateFormat.format(Date(startMillis ?: 0))
        val endDate = dateFormat.format(Date(endMillis ?: 0))

        Log.e(TAG, "선택된 기간: $startDate ~ $endDate")
    }
}
```


---

## 3.✅ 장점요약

## ✅ MaterialDatePicker의 장점 요약

| 항목                     | 장점 설명                                                                 |
|--------------------------|----------------------------------------------------------------------------|
| 🎨 최신 UI 지원          | Material Design 가이드라인 기반, 모던한 UI 제공                            |
| 📆 다양한 선택 방식       | 단일 날짜, 날짜 범위, 커스텀 기간 등 다양한 DatePicker 지원                |
| 🧩 Fragment 기반 구조     | DialogFragment 기반으로 라이프사이클에 안전                                |
| 🌐 다국어 자동 지원       | 시스템 로케일에 따라 요일, 월 등 자동 번역                                 |
| 🛠️ 커스터마이징 가능     | 타이틀, 초기 날짜, 포맷 등 다양한 옵션 지정 가능                          |
| 🔄 날짜 변경 콜백 제공    | addOnPositiveButtonClickListener로 간단한 이벤트 처리                       |
| ✅ Jetpack 공식 지원      | Jetpack Material 라이브러리로 안정적이고 유지보수 쉬움                     |

--- 

## 4.🧠 결론

- MaterialDatePicker는 기존 DatePickerDialog보다 훨씬 직관적이고,
- Fragment 라이프사이클에 안전하며, 스타일도 최신 Google 디자인 가이드에 부합합니다.