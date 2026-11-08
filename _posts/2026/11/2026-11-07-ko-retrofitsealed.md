---
title: (Android/Retrofit) Retrofit + Kotlin Sealed class 에러 처리 패턴
tags: [ Android, Retrofit ]
style: fill
color: dark
description: Sealed class로 네트워크 결과를 Success와 여러 종류의 Failure로 모델링하고 CallAdapter로 Retrofit에 연결해 when 분기로 안전하게 처리하는 패턴을 정리합니다.
---

---

## 1. 예외 기반 처리의 한계

```kotlin
try {
    val user = api.getUser(id)
} catch (e: HttpException) { ... } catch (e: IOException) { ... }
```

- 호출하는 쪽마다 try/catch가 반복되고, **처리를 빠뜨려도 컴파일러가 알려주지 않습니다**
- 에러 종류(인증, 서버, 네트워크 등)에 따른 분기가 코드 곳곳에 흩어집니다

**Sealed class**로 결과를 타입으로 표현하면 `when`에서 모든 경우를 **컴파일 타임에 강제**할 수 있습니다.

---

## 2. 결과 모델링

```kotlin
sealed interface ApiResult<out T> {

    data class Success<T>(val data: T) : ApiResult<T>

    sealed interface Failure : ApiResult<Nothing> {
        data class Http(val code: Int, val message: String?) : Failure   // 4xx, 5xx
        data object Network : Failure                                    // 연결 불가, 타임아웃
        data class Unknown(val cause: Throwable) : Failure               // 파싱 오류 등
    }
}
```

필요하면 `Unauthorized`(401), `NotFound`(404) 같은 **앱에서 의미 있는 에러**를 별도 케이스로 추가합니다.

---

## 3. 수동 변환 — 공통 함수

```kotlin
suspend fun <T> apiCall(block: suspend () -> T): ApiResult<T> =
    try {
        ApiResult.Success(block())
    } catch (e: HttpException) {
        ApiResult.Failure.Http(e.code(), e.toApiError()?.message)
    } catch (e: IOException) {
        ApiResult.Failure.Network
    } catch (e: SerializationException) {
        ApiResult.Failure.Unknown(e)
    }

// Repository
suspend fun getUser(id: Long): ApiResult<User> = apiCall { api.getUser(id) }
```

---

## 4. CallAdapter — API 선언에서 바로 ApiResult 반환

Retrofit의 `CallAdapter`를 만들면 **API 인터페이스 반환 타입에 `ApiResult`를 직접** 쓸 수 있어 변환 코드가 사라집니다.

```kotlin
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): ApiResult<User>    // 예외 대신 결과로 반환
}
```

```kotlin
class ApiResultCallAdapterFactory : CallAdapter.Factory() {

    override fun get(returnType: Type, annotations: Array<out Annotation>, retrofit: Retrofit): CallAdapter<*, *>? {
        // suspend fun은 Call<T>로 전달됨
        if (getRawType(returnType) != Call::class.java) return null
        val resultType = getParameterUpperBound(0, returnType as ParameterizedType)
        if (getRawType(resultType) != ApiResult::class.java) return null

        val successType = getParameterUpperBound(0, resultType as ParameterizedType)
        return ApiResultCallAdapter<Any>(successType)
    }
}

private class ApiResultCallAdapter<T>(private val successType: Type) :
    CallAdapter<T, Call<ApiResult<T>>> {

    override fun responseType(): Type = successType
    override fun adapt(call: Call<T>): Call<ApiResult<T>> = ApiResultCall(call)
}

private class ApiResultCall<T>(private val delegate: Call<T>) : Call<ApiResult<T>> {

    override fun enqueue(callback: Callback<ApiResult<T>>) {
        delegate.enqueue(object : Callback<T> {
            override fun onResponse(call: Call<T>, response: Response<T>) {
                val result: ApiResult<T> = if (response.isSuccessful) {
                    response.body()?.let { ApiResult.Success(it) }
                        ?: ApiResult.Failure.Unknown(IllegalStateException("빈 응답"))
                } else {
                    ApiResult.Failure.Http(response.code(), response.message())
                }
                callback.onResponse(this@ApiResultCall, Response.success(result))   // 항상 성공으로 감싸 전달
            }

            override fun onFailure(call: Call<T>, t: Throwable) {
                val result = if (t is IOException) ApiResult.Failure.Network
                             else ApiResult.Failure.Unknown(t)
                callback.onResponse(this@ApiResultCall, Response.success(result))
            }
        })
    }

    override fun clone() = ApiResultCall(delegate.clone())
    override fun execute(): Response<ApiResult<T>> = throw UnsupportedOperationException()
    override fun isExecuted() = delegate.isExecuted
    override fun cancel() = delegate.cancel()
    override fun isCanceled() = delegate.isCanceled
    override fun request(): Request = delegate.request()
    override fun timeout(): Timeout = delegate.timeout()
}
```

```kotlin
Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addCallAdapterFactory(ApiResultCallAdapterFactory())
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

> 이 어댑터는 `suspend` 호출에서 예외 대신 결과를 돌려줍니다. 코루틴 취소는 `delegate.cancel()`로 전달되고, `execute()`는 지원하지 않아도 suspend 방식에서는 호출되지 않습니다.

---

## 5. 사용 — when으로 모든 경우 강제

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(private val api: UserApi) : ViewModel() {

    private val _state = MutableStateFlow<UiState>(UiState.Loading)
    val state = _state.asStateFlow()

    fun load(id: Long) = viewModelScope.launch {
        _state.value = when (val result = api.getUser(id)) {
            is ApiResult.Success -> UiState.Content(result.data)
            is ApiResult.Failure.Http -> when (result.code) {
                401 -> UiState.NeedLogin
                404 -> UiState.Empty
                else -> UiState.Error("서버 오류 (${result.code})")
            }
            ApiResult.Failure.Network -> UiState.Error("네트워크 연결을 확인해 주세요.")
            is ApiResult.Failure.Unknown -> UiState.Error("알 수 없는 오류가 발생했습니다.")
        }
    }
}
```

`Failure`에 새 케이스를 추가하면 **처리하지 않은 모든 `when`에서 컴파일 에러**가 나서 누락을 막습니다.

---

## 6. 결과 변환 확장 함수

```kotlin
inline fun <T, R> ApiResult<T>.map(transform: (T) -> R): ApiResult<R> = when (this) {
    is ApiResult.Success -> ApiResult.Success(transform(data))
    is ApiResult.Failure -> this           // 실패는 그대로 전달
}

inline fun <T> ApiResult<T>.onSuccess(block: (T) -> Unit): ApiResult<T> {
    if (this is ApiResult.Success) block(data); return this
}

api.getUser(1).map { it.toDomain() }       // DTO → 도메인 변환 후에도 에러는 유지
```

---

## 7. 정리

- 예외 대신 `sealed` 타입으로 **성공/실패를 값**으로 표현해 처리 누락을 컴파일러가 차단
- 실패는 `Http`, `Network`, `Unknown` 등으로 세분화하고 앱 의미가 있는 에러는 케이스로 추가
- `CallAdapter.Factory`를 만들면 API 선언에서 **`ApiResult<T>`를 직접 반환**
- ViewModel에서는 `when`으로 분기해 UI 상태로 매핑
- `map`/`onSuccess` 확장 함수로 결과 변환을 간결하게
