---
title: Android 안전한 공유 시트 열기 – ShareCompat 한 줄로 끝내기 (+파일 URI/권한 완벽 가이드)
tags: [Android]
style: fill
color: dark
description: Android 안전한 공유 시트 열기 – ShareCompat 한 줄로 끝내기 (+파일 URI/권한 완벽 가이드)
---

## ✨ 개요

안드로이드에서 이미지를 다른 앱으로 공유할 때 가장 간단하고 안전한 방법은 **`ShareCompat.IntentBuilder`**를 쓰는 것입니다.  
한 줄로 공유 시트를 띄울 수 있고, `content://` 권한 처리까지 깔끔하게 해결됩니다.

> 핵심 한 줄:  
> `ShareCompat.IntentBuilder(context).setType("image/*").setStream(uri).startChooser()`
---

## 1. 언제 ShareCompat를 써야 할까?

- **이미지/파일/텍스트 공유**를 빠르게 구현하고 싶을 때
- **file:// 노출 금지(FileUriExposedException)**를 피하고 싶을 때
- **권한 플래그(READ URI)**를 자동으로 붙여 안전하게 공유하고 싶을 때

`ShareCompat`는 `androidx.core`에 포함되어 있으며, 내부적으로 **권한 플래그**와 **액션 선택**을 안전하게 구성해줍니다.

---

## 2. 사용법

#### Gradle 의존성 - 프로젝트가 `androidx`를 쓴다면 보통 이미 포함되어 있습니다.
```gradle
dependencies {
    implementation "androidx.core:core-ktx:1.13.1" // or newer
}
- FLAG_SECURE가 켜진 창은 캡처 불가(보안화면).
```

#### 단일 이미지 공유

```kotlin
fun shareImage(context: Context, uri: Uri, title: String = "공유하기") {
    ShareCompat.IntentBuilder(context)
        .setType("image/*")
        .setStream(uri)                       // content:// 형태 권장
        .setChooserTitle(title)
        .startChooser()
}
```
- setType("image/*"): 이미지 전용 앱 위주로 목록이 좁혀집니다.
- setStream(uri): 반드시 content://(FileProvider/MediaStore) URI 사용.
- 💡 Fragment라면 requireContext()를 넘겨도 됩니다.

#### 여러 장 이미지 공유

```kotlin
fun shareImages(context: Context, uris: List<Uri>, title: String = "사진 공유") {
    ShareCompat.IntentBuilder(context)
        .setType("image/*")
        .also { builder -> uris.forEach { builder.addStream(it) } }
        .setChooserTitle(title)
        .startChooser()
}
```

### 텍스트 + 이미지 함께 공유

```kotlin
fun shareImageWithText(context: Context, uri: Uri, text: String) {
    ShareCompat.IntentBuilder(context)
        .setType("image/*")
        .setText(text)
        .setStream(uri)
        .setChooserTitle("공유하기")
        .startChooser()
}
```

#### MediaStore 에서 바로 가져와 공유하기

```kotlin
fun shareMediaStoreImage(context: Context, mediaStoreUri: Uri) {
    ShareCompat.IntentBuilder(context)
        .setType("image/*")
        .setStream(mediaStoreUri)
        .setChooserTitle("공유하기")
        .startChooser()
}
```
- 이미 MediaStore에 있는 항목은 원래가 content URI입니다.
- Android 10+의 Scoped Storage 환경에서도 문제없이 동작합니다.

---

## 3. 안정성 높이기 : FileProvider로 content URI 만들기

갤러리/다운로드 폴더의 파일을 직접 공유할 때는 file:// 대신 FileProvider로 변환하세요.

#### `AndroidManifest.xml`

```xml
<application ...>
    <provider
        android:name="androidx.core.content.FileProvider"
        android:authorities="${applicationId}.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
    </provider>
</application>
```
- 아이템 수가 많으면 아주 큰 비트맵이 생성되어 OOM 위험. 섹션 별로 나눠서 캡처하거나 PDF로 내보내는 전략 고려.

#### `res/xml/file_paths.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path name="images" path="Pictures/" />
    <cache-path name="cache" path="shared/" />
</paths>
```

#### File -> content URL 변환

```kotlin
fun fileToContentUri(context: Context, file: File): Uri =
    FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
```
- 이렇게 얻은 content URI를 setStream(uri)에 넣으면 끝!

---

## 4. 결론

- 권한 플래그: ShareCompat는 보통 FLAG_GRANT_READ_URI_PERMISSION을 자동으로 붙입니다. 만약 직접 인텐트를 만들어 쓸 땐 반드시 추가하세요.
- MIME 타입 정확히: PNG만 공유면 "image/png", 혼합이면 "image/*".
- 앱 존재 확인: 이례적으로 공유 대상이 아예 없다면 예외가 날 수 있으니 try/catch로 감싸 안정성 확보.
- 대용량/비공개 파일: 캐시 디렉터리(cache/)에 임시 복사 후 FileProvider로 공유하면 접근성/보안에 유리.
- 보안 화면(FLAG_SECURE): 캡처가 막힌 화면의 비트맵을 저장·공유하려는 경우 정책 검토가 필요.
> 안전성까지 고려하면, 실무에서는 FileProvider + ShareCompat 조합이 사실상 정답입니다.