---
title: Android 객체 직렬화 - 데이터 객체 전달하는 방법 @Parcelize 구현
tags: [Android]
style: fill
color: dark
description: Android 객체 직렬화 - 데이터 객체 전달하는 방법 @Parcelize 구현
---

## 개요

Android 앱에서 Activity나 Fragment 간 데이터를 전달할 때 객체를 직렬화하는 작업은 필수적입니다.

기존 방식으로 **Parcelable**을 구현하려면, 여러 메서드와 로직을 직접 작성해야 했습니다.

그러나 Kotlin의 @Parcelize 어노테이션을 사용하면 이러한 작업을 매우 간단하게 처리할 수 있습니다.

---

## 1. gradle (app) parcelize 추가
@Parcelize를 사용하려면 다음과 같이 Gradle 설정이 필요합니다.

```kotlin
plugins {
    id 'kotlin-parcelize' // 또는 id("kotlin-parcelize")
}
```

---

## 2. 사용자 객체 전달
@Parcelize를 사용하여 Parcelable 객체를 정의합니다.

```kotlin
import android.os.Parcelable
import kotlinx.parcelize.Parcelize

@Parcelize
data class User(
    val id: Int,
    val name: String,
    val email: String
) : Parcelable
```

---

## 3. Intent 통해 데이터 전달하고 받기 

### 3-1 Activity 간 데이터 전달하고 받기
```kotlin
// MainActivity.kt
val user = User(1, "Alice", "alice@example.com")
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("user", user)
startActivity(intent)

// DetailActivity.kt
val user = intent.getParcelableExtra<User>("user")
user?.let {
    Log.d("DetailActivity", "ID: ${it.id}, Name: ${it.name}, Email: ${it.email}")
}
```

### 3-2 Fragment 간 데이터 전달하고 받기

```kotlin
// 데이터 전송
val user = User(1, "Alice", "alice@example.com")
val fragment = DetailFragment().apply {
    arguments = Bundle().apply {
        putParcelable("user", user)
    }
}

// 데이터 수신
val user = arguments?.getParcelable<User>("user")
user?.let {
    Log.d("DetailFragment", "Name: ${it.name}, Email: ${it.email}")
}
```

---

## 4. @Parcelize 사용 시 주의사항
1. @Parcelize는 Kotlin 전용
- Kotlin으로 작성된 Android 프로젝트에서만 사용할 수 있습니다. Java에서는 사용할 수 없습니다.
2. 모든 필드가 Parcelable 가능해야 함:
- 클래스 필드에 Parcelable로 변환할 수 없는 타입이 포함되어 있다면 에러가 발생합니다.
- 예: SparseArray, HashMap 등은 사용 시 주의가 필요합니다.
3. 다른 버전과의 호환성:
- 직렬화된 데이터를 다른 앱 버전에서 읽어야 한다면, 필드 이름 변경이나 삭제 시 문제가 발생할 수 있습니다.

---