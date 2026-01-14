---
title: (Android/안드로이드) OS 기본 공유 UI로 이미지 공유하기
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) OS 기본 공유 UI로 이미지 공유하기 Intent.ACTION_SEND + FileProvider 정석 가이드
---

## 1. 안드로이드 앱에서 이미지를 공유할 때는 앱별 SDK를 직접 호출하지 않고,

- OS가 제공하는 기본 공유 UI(Chooser) 를 사용하는 것이 가장 안전하고 범용적입니다.
  - Bitmap / 파일 이미지를 OS 기본 공유 창으로 전달
  - Android 7.0+ 보안 정책(FileProvider) 대응
  - 가장 흔한 실수와 주의점

---

## 2. 요약

- 이미지 공유는 Intent.ACTION_SEND
- Uri.fromFile() ❌ → FileProvider 필수
- MIME 타입은 image/* 
- EXTRA_STREAM + FLAG_GRANT_READ_URI_PERMISSION 필수
- OS 공유창은 Intent.createChooser() 사용

---

## 3. 기본 개념: OS 기본 공유란?

- 아래와 같은 시스템 공유 패널을 말합니다.
  - 카카오톡
  - 메시지
  - Gmail
  - Drive
  - 사진 앱
  - 기타 설치된 앱들
- Intent 기반으로 OS가 자동 구성합니다.

---

## 4. 전체 흐름

- Bitmap → 파일로 저장 (cache 또는 files)
- FileProvider를 통해 content:// Uri 생성
- ACTION_SEND Intent 생성
- Intent.createChooser()로 공유 실행

---

## 5. Bitmap 파일 저장

```kotlin
fun saveBitmapToCache(context: Context, bitmap: Bitmap): File {
    val cacheDir = File(context.cacheDir, "images").apply { mkdirs() }
    val file = File(cacheDir, "share_image.png")

    FileOutputStream(file).use { out ->
        bitmap.compress(Bitmap.CompressFormat.PNG, 100, out)
    }
    return file
}

```

- cacheDir를 쓰는 이유
- 공유용 임시 파일
- 앱 종료 시 OS가 정리 가능
- 권장 방식


---

## 6. FileProvider 설정

```xml
<provider
        android:name="androidx.core.content.FileProvider"
        android:authorities="${applicationId}.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">

    <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths" />
</provider>
```

---

## 7. 파일 -> content Uri 변환

```kotlin
 fun shareImage(context: Context, imageUri: Uri) {
    val shareIntent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, imageUri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }

    context.startActivity(
        Intent.createChooser(shareIntent, "이미지 공유")
    )
}
```

---

## 8. OS 기본 공유 Intent 생성 & 호출

```kotlin
fun shareBitmap(context: Context, bitmap: Bitmap) {
    val file = saveBitmapToCache(context, bitmap)
    val uri = getContentUri(context, file)
    shareImage(context, uri)
}

fun shareImage(context: Context, imageUri: Uri) {
    val shareIntent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, imageUri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }

    context.startActivity(
        Intent.createChooser(shareIntent, "이미지 공유")
    )
}
```

- type = "image/*"
- EXTRA_STREAM → 이미지 Uri
- FLAG_GRANT_READ_URI_PERMISSION → 상대 앱에 읽기 권한 부여

---

## 9. 여러 장의 이미지 보내는 법

```kotlin
val uris: ArrayList<Uri> = arrayListOf(uri1, uri2)

Intent(Intent.ACTION_SEND_MULTIPLE).apply {
    type = "image/*"
    putParcelableArrayListExtra(Intent.EXTRA_STREAM, uris)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
```