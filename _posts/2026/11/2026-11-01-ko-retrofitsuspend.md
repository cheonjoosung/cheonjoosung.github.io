---
title: (Android/Retrofit) Retrofit suspend vs Call 비교
tags: [ Android, Retrofit ]
style: fill
color: dark
description: Retrofit의 Call 방식과 suspend 함수 방식의 동작 차이, 에러 처리, 취소, 코루틴 연동 시 장단점을 비교해 정리합니다.
---

---

## 1. 두 가지 선언 방식

Retrofit은 API 인터페이스의 반환 타입에 따라 호출 방식이 달라집니다.

```kotlin
interface UserApi {
    // 1. Call 방식 — 콜백 기반 (코루틴 이전 방식)
    @GET("users/{id}")
    fun getUserCall(@Path("id") id: Long): Call<User>

    // 2. suspend 방식 — 코루틴 기반 (Retrofit 2.6+)
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): User
}
```

---

## 2. Call 방식

`Call`은 **실행 시점을 직접 제어**하는 객체입니다. 선언만으로는 요청이 나가지 않습니다.

```kotlin
val call = api.getUserCall(1)

// 비동기 실행 (콜백)
call.enqueue(object : Callback<User> {
    override fun onResponse(call: Call<User>, response: Response<User>) {
        if (response.isSuccessful) {
            val user = response.body()          // null 가능 (body가 없는 응답)
        } else {
            val code = response.code()          // 4xx, 5xx도 onResponse로 들어옴
        }
    }
    override fun onFailure(call: Call<User>, t: Throwable) {
        // 네트워크 오류(IOException) 등 — HTTP 응답 자체가 없는 경우
    }
})

// 취소
call.cancel()

// 동기 실행 (메인 스레드에서 호출 금지)
val response = call.execute()
```

> **콜백 지옥**: 요청 A의 결과로 B를 호출하는 순차 흐름이 중첩 콜백으로 이어져 가독성과 에러 처리가 나빠집니다.

---

## 3. suspend 방식

일반 코드처럼 **순차적으로** 작성할 수 있습니다.

```kotlin
class UserRepository(private val api: UserApi) {

    suspend fun loadProfile(id: Long): Profile {
        val user = api.getUser(id)                 // 결과가 올 때까지 일시 중단
        val posts = api.getPosts(user.id)          // 앞 결과를 사용해 이어서 호출
        return Profile(user, posts)
    }
}

// 병렬 호출도 간단
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user = async { api.getUser(1) }
    val feed = async { api.getFeed() }
    Dashboard(user.await(), feed.await())
}
```

Retrofit이 내부적으로 `Call.enqueue`를 사용하며, 스레드는 OkHttp의 디스패처가 관리하므로 **메인 스레드에서 호출해도 안전**합니다 (`withContext(IO)`가 필요 없음).

---

## 4. 에러 처리 차이

| 상황 | Call | suspend (`User` 반환) |
|------|------|----------------------|
| 2xx 성공 | `onResponse`, `isSuccessful` | 값 반환 |
| 4xx/5xx | `onResponse`, `isSuccessful == false` | `HttpException` **예외** |
| 네트워크 오류 | `onFailure` | `IOException` **예외** |
| 파싱 오류 | `onFailure` | `JsonDataException` 등 예외 |

```kotlin
// suspend: try/catch로 한 곳에서 처리
try {
    val user = api.getUser(id)
} catch (e: HttpException) {
    val code = e.code()                      // 4xx, 5xx
} catch (e: IOException) {
    // 네트워크 연결 문제
}
```

---

## 5. suspend + Response<T> — 응답 전체 받기

예외 대신 **상태 코드와 헤더를 직접 다루고 싶다면** `Response<T>`로 감쌉니다.

```kotlin
@GET("users/{id}")
suspend fun getUserResponse(@Path("id") id: Long): Response<User>

val response = api.getUserResponse(1)
if (response.isSuccessful) {
    val user = response.body()!!
    val etag = response.headers()["ETag"]    // 헤더 접근
} else {
    val errorJson = response.errorBody()?.string()   // 에러 본문 접근
}
```

이 방식은 4xx/5xx에서 **예외가 발생하지 않고** `isSuccessful`로 분기합니다. (네트워크 오류는 여전히 예외)

---

## 6. 취소 동작

```kotlin
val job = viewModelScope.launch {
    val user = api.getUser(1)        // 코루틴이 취소되면 진행 중인 HTTP 요청도 취소됨
}
job.cancel()
```

suspend 방식은 코루틴 취소와 **자동으로 연동**됩니다. `Call` 방식은 직접 `call.cancel()`을 호출하고 생명주기에 맞춰 관리해야 합니다.

> 취소되면 `CancellationException`이 발생합니다. `catch (e: Exception)`으로 넓게 잡을 때는 이 예외를 삼키지 말고 다시 던지세요.

---

## 7. 언제 무엇을 쓰나

| | Call | suspend |
|---|------|---------|
| 가독성 | 콜백 중첩 | **순차 코드** |
| 취소 | 수동 | **코루틴 자동 연동** |
| 병렬 처리 | 복잡 | `async/await` |
| 요청 재사용/재시도 | `call.clone()` | 함수 재호출 |
| 요청 지연 실행 | **가능 (객체로 전달)** | 불가 (호출 즉시 실행) |

`Call`은 요청을 객체로 들고 다니며 나중에 실행하거나 복제해야 하는 특수한 경우, 또는 코루틴을 쓰지 않는 레거시 코드에서만 유지하고 **새 코드는 suspend**를 기본으로 합니다.

---

## 8. 정리

- `Call`은 콜백 기반, `suspend`는 코루틴 기반으로 순차적이고 간결
- suspend는 **4xx/5xx가 `HttpException`**, 네트워크 오류가 `IOException`으로 던져짐
- 상태 코드/헤더가 필요하면 `suspend fun ...(): Response<T>`
- suspend는 코루틴 **취소와 자동 연동**되고 메인 스레드 호출도 안전
- 신규 코드는 suspend를 기본으로 선택
