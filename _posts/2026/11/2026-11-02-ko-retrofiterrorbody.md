---
title: (Android/Retrofit) Retrofit 에러 응답 파싱 — ErrorBody 처리
tags: [ Android, Retrofit ]
style: fill
color: dark
description: Retrofit에서 4xx, 5xx 응답의 errorBody를 파싱해 서버 에러 메시지와 코드를 꺼내는 방법과 공통 처리 유틸을 정리합니다.
---

---

## 1. 문제 상황

서버는 에러일 때도 **구조화된 JSON**을 내려주는 경우가 많습니다.

```json
// HTTP 400
{ "code": "EMAIL_DUPLICATED", "message": "이미 가입된 이메일입니다." }
```

하지만 Retrofit은 성공 응답 타입(`User`)으로만 파싱하므로, 에러 본문은 **자동으로 변환되지 않습니다.** 직접 꺼내서 파싱해야 합니다.

---

## 2. errorBody 위치

```kotlin
// suspend 방식: HttpException 안에 있음
try {
    api.signUp(request)
} catch (e: HttpException) {
    val code: Int = e.code()                              // HTTP 상태 코드
    val errorBody: ResponseBody? = e.response()?.errorBody()
}

// Response<T> 방식: response 안에 있음
val response = api.signUpResponse(request)
if (!response.isSuccessful) {
    val errorBody = response.errorBody()
}
```

> `errorBody().string()`은 **한 번만 읽을 수 있습니다.** 두 번 호출하면 두 번째는 빈 문자열이 되므로 한 번 읽어 변수에 저장하세요.

---

## 3. 에러 모델 정의와 파싱

```kotlin
@Serializable
data class ApiError(
    val code: String,
    val message: String
)

private val json = Json { ignoreUnknownKeys = true }

fun HttpException.toApiError(): ApiError? =
    try {
        response()?.errorBody()?.string()?.let { json.decodeFromString<ApiError>(it) }
    } catch (e: SerializationException) {
        null          // 서버가 JSON이 아닌 HTML(502 등)을 내려준 경우
    }
```

Gson/Moshi를 쓰는 경우도 같은 방식으로 `errorBody().string()`을 해당 라이브러리로 파싱하면 됩니다.

---

## 4. Converter로 파싱하기 (Retrofit 기본 방식)

이미 등록된 ConverterFactory를 재사용해 에러 본문을 변환할 수 있습니다.

```kotlin
class ErrorParser(private val retrofit: Retrofit) {

    private val converter: Converter<ResponseBody, ApiError> =
        retrofit.responseBodyConverter(ApiError::class.java, emptyArray())

    fun parse(response: Response<*>): ApiError? =
        try {
            response.errorBody()?.let { converter.convert(it) }
        } catch (e: Exception) {
            null
        }
}
```

별도 JSON 설정을 중복하지 않아도 되는 것이 장점입니다.

---

## 5. 공통 처리 — 안전한 호출 래퍼

화면마다 try/catch를 반복하지 않도록 **한 곳에서 에러를 해석**합니다.

```kotlin
sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>
    data class Failure(val code: Int?, val message: String) : NetworkResult<Nothing>
}

suspend fun <T> safeApiCall(block: suspend () -> T): NetworkResult<T> =
    try {
        NetworkResult.Success(block())
    } catch (e: HttpException) {
        val apiError = e.toApiError()
        NetworkResult.Failure(
            code = e.code(),
            message = apiError?.message ?: "서버 오류가 발생했습니다 (${e.code()})"
        )
    } catch (e: IOException) {
        NetworkResult.Failure(null, "네트워크 연결을 확인해 주세요.")
    }

// 사용
when (val result = safeApiCall { api.signUp(request) }) {
    is NetworkResult.Success -> showWelcome(result.data)
    is NetworkResult.Failure -> showError(result.message)
}
```

> 앞서 말했듯 `CancellationException`은 잡지 않아야 합니다. 위 코드는 `HttpException`과 `IOException`만 잡아 코루틴 취소가 정상 전파됩니다.

---

## 6. 상태 코드별 처리

| 코드 | 일반적 의미 | 처리 예 |
|------|------------|--------|
| 400 | 잘못된 요청 | 서버 메시지 표시 |
| 401 | 인증 실패/만료 | 토큰 갱신 또는 로그인 화면 |
| 403 | 권한 없음 | 권한 안내 |
| 404 | 없음 | 빈 상태 화면 |
| 409 | 충돌(중복) | 중복 안내 |
| 429 | 요청 과다 | 잠시 후 재시도 |
| 5xx | 서버 오류 | 재시도 안내 |

401은 Retrofit 호출마다 처리하기보다 OkHttp `Authenticator`에서 토큰 갱신을 공통 처리하는 것이 좋습니다.

---

## 7. 주의사항

- 에러 본문이 JSON이 아닐 수 있습니다 (프록시/게이트웨이의 HTML). 파싱 실패를 항상 대비하세요
- 사용자에게 보여줄 문구와 **로그용 원본**을 구분하세요 (서버 내부 메시지를 그대로 노출하지 않기)
- 에러 본문에 민감 정보가 있을 수 있어 로그에 남길 때 주의하세요

---

## 8. 정리

- 에러 본문은 자동 변환되지 않으므로 `HttpException.response()?.errorBody()`에서 직접 파싱
- `errorBody().string()`은 **한 번만** 읽을 수 있음
- 파싱 실패(JSON이 아닌 응답)를 `null`로 안전하게 처리
- `safeApiCall` 같은 **공통 래퍼**로 에러 해석을 한 곳에 모으기
- 401은 `Authenticator`, 나머지는 상태 코드별로 UI 정책 결정
