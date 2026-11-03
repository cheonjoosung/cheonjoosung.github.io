---
title: (Android/Retrofit) Retrofit 파일 업로드 — Multipart, ProgressBody
tags: [ Android, Retrofit ]
style: fill
color: dark
description: Retrofit으로 이미지와 파일을 Multipart로 업로드하는 방법과 RequestBody 생성, 여러 파일 전송, 업로드 진행률 표시 구현을 정리합니다.
---

---

## 1. Multipart란?

파일과 텍스트 필드를 **하나의 HTTP 요청에 여러 파트로 나눠** 보내는 형식입니다.

```
POST /upload
Content-Type: multipart/form-data; boundary=xxx

--xxx
Content-Disposition: form-data; name="title"
내 사진
--xxx
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg
(바이너리 데이터)
--xxx--
```

---

## 2. API 선언

```kotlin
interface UploadApi {

    @Multipart
    @POST("upload")
    suspend fun upload(
        @Part file: MultipartBody.Part,                       // 파일 파트
        @Part("title") title: RequestBody                     // 텍스트 파트
    ): UploadResponse

    // 여러 파일
    @Multipart
    @POST("upload/multi")
    suspend fun uploadMany(
        @Part files: List<MultipartBody.Part>
    ): UploadResponse
}
```

---

## 3. 파일을 Part로 만들기

```kotlin
fun File.toPart(partName: String = "file"): MultipartBody.Part {
    val body = asRequestBody("image/jpeg".toMediaType())      // 파일 → RequestBody
    return MultipartBody.Part.createFormData(partName, name, body)
}

val file = File(cacheDir, "photo.jpg")
val title = "내 사진".toRequestBody("text/plain".toMediaType())

api.upload(file.toPart(), title)
```

> MIME 타입은 확장자에서 유추할 수 있습니다. `MimeTypeMap.getSingleton().getMimeTypeFromExtension(file.extension)`

---

## 4. Uri(갤러리 선택)에서 업로드

Android 10+에서는 파일 경로가 아닌 **Uri**를 받습니다. 경로 변환 대신 `ContentResolver`로 스트림을 읽는 `RequestBody`를 만듭니다.

```kotlin
class UriRequestBody(
    private val contentResolver: ContentResolver,
    private val uri: Uri,
    private val mimeType: String?
) : RequestBody() {

    override fun contentType() = mimeType?.toMediaTypeOrNull()

    override fun contentLength(): Long =
        contentResolver.query(uri, arrayOf(OpenableColumns.SIZE), null, null, null)?.use {
            if (it.moveToFirst()) it.getLong(0) else -1L
        } ?: -1L

    override fun writeTo(sink: BufferedSink) {
        contentResolver.openInputStream(uri)?.source()?.use { sink.writeAll(it) }
    }
}

fun Uri.toPart(resolver: ContentResolver, name: String = "file"): MultipartBody.Part {
    val mime = resolver.getType(this)
    val fileName = "upload_${System.currentTimeMillis()}"
    return MultipartBody.Part.createFormData(name, fileName, UriRequestBody(resolver, this, mime))
}
```

메모리에 전체를 올리지 않고 **스트리밍**으로 전송하므로 큰 파일에도 안전합니다.

---

## 5. 업로드 진행률 — ProgressRequestBody

`RequestBody`를 감싸 **전송된 바이트를 추적**합니다.

```kotlin
class ProgressRequestBody(
    private val delegate: RequestBody,
    private val onProgress: (written: Long, total: Long) -> Unit
) : RequestBody() {

    override fun contentType() = delegate.contentType()
    override fun contentLength() = delegate.contentLength()

    override fun writeTo(sink: BufferedSink) {
        val total = contentLength()
        val countingSink = object : ForwardingSink(sink) {
            private var written = 0L
            override fun write(source: Buffer, byteCount: Long) {
                super.write(source, byteCount)
                written += byteCount
                onProgress(written, total)
            }
        }
        val buffered = countingSink.buffer()
        delegate.writeTo(buffered)
        buffered.flush()                     // 남은 버퍼 전송
    }
}
```

```kotlin
val body = ProgressRequestBody(file.asRequestBody(mediaType)) { written, total ->
    if (total > 0) progressFlow.value = (written * 100 / total).toInt()
}
val part = MultipartBody.Part.createFormData("file", file.name, body)
```

> 진행률 콜백은 **OkHttp 스레드**에서 호출됩니다. UI 갱신은 `StateFlow`에 값을 쓰거나 메인 스레드로 전달하세요. 또한 `HttpLoggingInterceptor(Level.BODY)`가 켜져 있으면 본문을 두 번 쓰게 되어 진행률이 이중으로 올라가므로 업로드 시에는 로그 레벨을 낮추세요.

---

## 6. 대용량·이미지 업로드 팁

- 사진은 **업로드 전에 리사이즈/압축**해서 전송량을 줄이세요 (원본 수십 MB는 비효율)
- 장시간/대용량 업로드는 앱이 종료되어도 이어지도록 **WorkManager**와 함께 사용하세요
- 서버 파트 이름(`"file"`)은 서버 명세와 정확히 일치해야 합니다
- 타임아웃은 업로드용 OkHttpClient의 `writeTimeout`을 넉넉히 설정하세요

---

## 7. 정리

- `@Multipart` + `@Part`로 파일/텍스트를 한 요청에 전송
- 파일은 `asRequestBody` → `MultipartBody.Part.createFormData`
- Android 10+ 갤러리 Uri는 `ContentResolver` 스트림을 쓰는 **커스텀 RequestBody**로 전송
- 진행률은 `RequestBody`를 감싸 `ForwardingSink`로 바이트를 집계 (로깅 레벨 주의)
- 큰 파일은 압축과 WorkManager를 함께 고려
