---
title: Android Photo Picker 사용법 (권한 없이 이미지 선택하기)
tags: [Android]
style: fill
color: dark
description: Android 13+에서 도입된 Photo Picker를 사용하여 권한 없이 이미지를 선택하는 방법과 기존 방식과의 차이점을 소개합니다.
---
---

## ✨ 개요

Android 13(API 33)부터 Google은 **새로운 시스템 UI 기반의 이미지/영상 선택기**인 **Photo Picker**를 도입했습니다.

기존 방식(`ACTION_PICK`, `GetContent`)과 달리,**권한 없이도 사진을 선택할 수 있는 보안 친화적인 방식**으로 사용자의 민감한 파일 접근을 최소화합니다.

---

## 1. ✅ Photo Picker란?

- OS에서 제공하는 **표준 이미지/영상 선택 UI**
- 앱이 저장소를 직접 읽지 않고도 사용자의 미디어 파일을 선택 가능
- `READ_EXTERNAL_STORAGE` 또는 `READ_MEDIA_IMAGES` 권한 불필요

---

## 2. ✅ 기존 방식과 비교

| 항목                    | 기존 방식 (`ACTION_PICK`)     | Photo Picker (Android 13+)    |
|-------------------------|-------------------------------|-------------------------------|
| UI 제공 주체            | 앱 또는 OS 제공 (`Intent`)     | **OS 전용 UI 제공**             |
| 저장소 권한 필요 여부   | ✅ 필요함 (`READ_EXTERNAL_STORAGE`) | ❌ 필요 없음                    |
| Android 지원 범위       | 전체 버전                     | Android 13(API 33)+ 이상        |
| 사진 외 다른 파일 선택  | 가능 (확장자 제한 가능)         | 이미지/비디오에 특화됨          |
| 사용자 보안             | 앱이 전체 저장소 접근          | **앱은 선택된 파일에만 접근 가능** |

---

## 3. 🔥 코드 예제 – Photo Picker 사용

### 3.1 기본 사용 (단일 이미지 선택)

```kotlin
import androidx.activity.result.contract.ActivityResultContracts
import androidx.activity.result.PickVisualMediaRequest

val photoPickerLauncher = registerForActivityResult(
  ActivityResultContracts.PickVisualMedia()
) { uri ->
  uri?.let {
    binding.imageView.setImageURI(it)
  }
}

fun singlePhotoPicker() {
  photoPickerLauncher.launch(
    PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
  )
}
```
PickVisualMedia.ImageOnly 외에도 .VideoOnly, .ImageAndVideo 사용 가능

### 3.2 여러 개 선택 (Multiple Photo Picker)

```kotlin
import androidx.activity.result.contract.ActivityResultContracts
import androidx.activity.result.PickVisualMediaRequest

val multiPicker = registerForActivityResult(
  ActivityResultContracts.PickMultipleVisualMedia()
) { uris ->
  uris.forEach { uri ->
    Log.d("PhotoPicker", "선택된 이미지: $uri")
  }
}

fun multiPhotoPicker() {
  multiPicker.launch(
    PickVisualMediaRequest.Builder()
      .setMediaType(ActivityResultContracts.PickVisualMedia.ImageOnly)
      .build()
  )
}
```

---

## 4. 🚫 권한이 필요 없는 이유?

기존에는 권한이 필요헀으나
```xml
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
```
Photo Picker는 OS 자체 UI로 파일을 선택하고 앱에는 선택된 URI만 전달되기 때문에, 사용자 저장소 접근 권한이 전혀 필요하지 않습니다.

---

## 5. 결론

- Android 13 이상에서는 Photo Picker를 우선적으로 사용하는 것이 권장됩니다.
- 권한 없이 이미지 선택 가능 → UX 향상 + 보안 강화
- 하위 버전은 기존 방식과 함께 버전별 분기 처리로 대응

