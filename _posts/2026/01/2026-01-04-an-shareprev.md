---
title: (Android/안드로이드) OS 공유창에서 이미지 미리보기(Preview) 나오게 하기
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) OS 공유창에서 이미지 미리보기(Preview) 나오게 하기 - ACTION_SEND + EXTRA_STREAM에서 프리뷰가 안 뜨는 이유와 해결법
---

## 1. 이미지를 OS 기본 공유창(chooser/sharesheet)으로 넘길 때 아래 코드가 정석처럼 보입니다.

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

- 그런데 실제로는 기기/OS 버전/공유 대상 앱에 따라 공유 패널 상단에 이미지 프리뷰가 안 보이는 경우가 있습니다.
  - “프리뷰를 직접 넣는 옵션이 있는가?”
  - 프리뷰가 뜨지 않는 대표 원인
  - 프리뷰가 뜨게 만드는 실무 해법(ClipData 포함)
  - FileProvider / ContentProvider 메타데이터 팁

---

## 2. 요약

- Android Sharesheet는 개발자가 “프리뷰 이미지를 따로 지정”하는 공식 API가 거의 없습니다.
- 프리뷰는 OS가 EXTRA_STREAM의 Uri를 읽어 자동 생성합니다.
- 따라서 프리뷰가 안 뜨면 대부분 Uri 접근 권한/메타데이터(파일명, MIME)/ClipData 누락 문제입니다.
- 실무에서는 FLAG_GRANT_READ_URI_PERMISSION + ClipData 설정이 가장 효과적입니다.

---

## 3. “미리보기 이미지를 넣는 옵션”이 따로 있을까?

- 결론적으로, 공유 패널의 이미지 프리뷰는 OS가 자동 생성하는 영역이며 개발자가 “프리뷰용 썸네일 Bitmap”을 직접 주입하는 표준 옵션은 없다
- 프리뷰가 보이려면 OS가 EXTRA_STREAM에 들어있는 Uri를 정상적으로 읽어야 합니다.

---

## 4. 프리뷰가 안 뜨는 대표 원인 4가지

- Uri 권한이 부족함 (가장 흔함)
  - content:// Uri를 받는 앱(또는 OS)이 읽을 권한이 없음
  - 결과적으로 미리보기를 생성 못하고 텍스트만 노출
- EXTRA_STREAM은 넣었는데 ClipData가 없음
  - 일부 앱/OS 조합에서는 FLAG_GRANT_READ_URI_PERMISSION만으로 충분하지 않고 ClipData로 명시적인 Uri 권한을 부여해야 프리뷰가 뜹니다.
- MIME 타입이 부정확함
  - type="*/*" 또는 누락 시 프리뷰 처리가 제한될 수 있음
  - 이미지면 image/* 또는 정확한 image/png, image/jpeg 권장
- ContentProvider가 파일 정보를 제대로 못 줌
  - OS가 프리뷰 생성을 위해 다음 정보를 조회할 수 있습니다.
  - DISPLAY_NAME (파일명)
  - MIME type
  - size
- FileProvider 사용 시 대체로 OK지만, 커스텀 Provider에서는 메타데이터 제공이 필요합니다.

---

## 5. ClipData까지 포함해 공유하기

```kotlin
fun shareImageWithPreview(context: Context, imageUri: Uri) {
    val shareIntent = Intent(Intent.ACTION_SEND).apply {
        type = "image/*"
        putExtra(Intent.EXTRA_STREAM, imageUri)

        // 읽기 권한 부여
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)

        // 중요: ClipData로 Uri를 명시하면 일부 기기/앱에서 프리뷰가 정상 노출됨
        clipData = ClipData.newUri(context.contentResolver, "shared_image", imageUri)
    }

    context.startActivity(Intent.createChooser(shareIntent, "이미지 공유"))
}
```

- Intent flag는 “권한 부여 의도”이지만, 일부 수신 앱은 EXTRA_STREAM만 보고 파일을 여는 과정에서 권한을 못 받는 경우가 있습니다.
- ClipData는 OS가 URI grant를 적용하는 경로에서 더 직접적으로 작동하는 케이스가 있습니다.


---

## 6. FileProvider 설정이 프리뷰에 영향을 줄까?

- 영향이 있습니다. 특히 다음이 중요합니다.
- 공유 Uri는 file://가 아니라 content:// 여야 함
- FileProvider를 쓰면 OS가 파일 정보를 얻기 쉬워 프리뷰 생성에 유리

```xml
<!-- Manifest -->
<provider
        android:name="androidx.core.content.FileProvider"
        android:authorities="${applicationId}.fileprovider"
        android:exported="false"
        android:grantUriPermissions="true">
    <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/file_paths" />
</provider>

<!-- file_paths.xml -->
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <cache-path name="cache" path="." />
</paths>

```

---

## 7. 프리뷰가 꼭 떠야 한다”면 이것도 같이 점검

- 파일 확장자/파일명 제공
  - OS나 일부 앱이 파일명을 기반으로 MIME을 추정하기도 합니다.
  - 유용 파일을 저장할 때 확장자를 명확히 주세요.
    - share_image.png
    - share_image.jpg
- MIME을 더 정확히 지정
  - `type = "image/png"`
- 여러 장 공유 시 ACTION_SEND_MULTIPLE
  - 프리뷰가 다르게 동작할 수 있으니, 여러 장이면 아래로 분기합니다.
  - ```kotlin
    Intent(Intent.ACTION_SEND_MULTIPLE).apply {
        type = "image/*"
        putParcelableArrayListExtra(Intent.EXTRA_STREAM, uris)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        clipData = ClipData.newUri(context.contentResolver, "shared_images", uris.first())
    }
    ```