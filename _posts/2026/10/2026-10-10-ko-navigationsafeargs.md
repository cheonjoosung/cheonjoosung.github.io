---
title: (Android/Navigation) Navigation Component — Safe Args 타입 안전 전달
tags: [ Android, Navigation ]
style: fill
color: dark
description: Navigation Safe Args 플러그인으로 화면 간 인자를 타입 안전하게 전달하고 받는 방법과 기본값, Parcelable, 결과 반환을 정리합니다.
---

---

## 1. 왜 Safe Args인가?

Bundle로 인자를 전달하면 **키 문자열 오타와 타입 불일치**가 런타임에야 드러납니다.

```kotlin
// ❌ Bundle: 키 오타, 타입 오류가 컴파일 타임에 잡히지 않음
navController.navigate(R.id.detailFragment, bundleOf("produtId" to 42))
val id = arguments?.getString("productId")  // null 또는 ClassCastException
```

Safe Args는 그래프 정의를 읽어 **전용 클래스를 생성**해 컴파일 타임에 검증합니다.

---

## 2. 설정

```kotlin
// 프로젝트 build.gradle.kts
plugins {
    id("androidx.navigation.safeargs.kotlin") version "2.8.0" apply false
}

// 앱 모듈 build.gradle.kts
plugins {
    id("androidx.navigation.safeargs.kotlin")
}
```

---

## 3. 그래프에 인자 선언

```xml
<fragment
    android:id="@+id/detailFragment"
    android:name="com.example.app.DetailFragment">

    <argument
        android:name="productId"
        app:argType="integer" />          <!-- 필수 인자 -->

    <argument
        android:name="source"
        app:argType="string"
        android:defaultValue="home" />    <!-- 기본값이 있으면 선택 인자 -->

    <argument
        android:name="product"
        app:argType="com.example.app.Product"
        app:nullable="true"
        android:defaultValue="@null" />   <!-- Parcelable 객체, null 허용 -->
</fragment>
```

---

## 4. 보내기 — Directions 클래스

빌드하면 `HomeFragmentDirections`가 생성되고, **필수 인자는 함수 파라미터**가 됩니다.

```kotlin
binding.itemClick = { productId ->
    // 필수 인자를 빠뜨리면 컴파일 에러
    val action = HomeFragmentDirections.actionHomeToDetail(productId = productId)
    findNavController().navigate(action)
}
```

---

## 5. 받기 — navArgs()

`DetailFragmentArgs`가 생성되며 `navArgs()` 위임으로 꺼냅니다.

```kotlin
class DetailFragment : Fragment(R.layout.fragment_detail) {

    // 타입이 보장된 인자 접근 (Int, String 등)
    private val args: DetailFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val id: Int = args.productId
        val source: String = args.source   // 지정 안 하면 "home"
        viewModel.load(id)
    }
}
```

> ViewModel에서는 `SavedStateHandle`로 같은 인자를 받을 수 있습니다 (`savedStateHandle.get<Int>("productId")`). Hilt ViewModel이면 별도 전달 코드가 필요 없습니다.

---

## 6. 지원 타입

| argType | Kotlin 타입 |
|---------|------------|
| `integer` | Int |
| `long`, `float`, `boolean` | Long, Float, Boolean |
| `string` | String |
| `reference` | 리소스 ID (@IdRes 등) |
| 클래스 경로 | Parcelable, Serializable, enum |
| `integer[]` 등 | 배열 (`app:argType="integer[]"`) |

---

## 7. 화면 결과 돌려받기

Safe Args는 **보내는 방향**만 담당합니다. 결과 반환은 `SavedStateHandle`이나 Fragment Result API를 사용합니다.

```kotlin
// 결과를 돌려주는 쪽 (SelectFragment)
findNavController().previousBackStackEntry
    ?.savedStateHandle?.set("selected_color", "red")
findNavController().popBackStack()

// 결과를 받는 쪽 (HomeFragment)
findNavController().currentBackStackEntry
    ?.savedStateHandle
    ?.getLiveData<String>("selected_color")
    ?.observe(viewLifecycleOwner) { color -> /* 반영 */ }
```

---

## 8. 정리

- Bundle 키 문자열 대신 **생성된 Directions/Args 클래스**를 사용해 컴파일 타임 검증
- 그래프의 `<argument>`로 타입과 기본값을 선언 (기본값 없으면 필수)
- 전송: `XxxFragmentDirections.actionXxx(...)`, 수신: `by navArgs<XxxFragmentArgs>()`
- 큰 객체 대신 **ID만 전달**하고 대상 화면에서 조회하는 것이 안전
- 결과 반환은 SavedStateHandle 또는 Fragment Result API
