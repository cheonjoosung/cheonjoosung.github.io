---
title: (Android/안드로이드) Retrofit2 & OkHttp 이해하기 + 네트워크 프로토콜 기본 개념 정리 
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Retrofit2 & OkHttp 이해하기 + 네트워크 프로토콜 기본 개념 정리
---

## ✨ 개요

Android에서 네트워크 통신을 구현할 때 가장 많이 사용되는 조합은 Retrofit2 + OkHttp이다.
Retrofit2는 “API 인터페이스 작성 + 응답 파싱 자동화”에 특화된 HTTP 클라이언트이고,
OkHttp는 네트워크 레이어를 담당하는 고성능 HTTP 엔진이다.

이 포스팅에서는 Android 개발자가 반드시 알아야 할
Retrofit2, OkHttp, 그리고 HTTP·TCP·TLS 같은 네트워크 프로토콜 개념까지 한 번에 정리한다.

---

## 1. 요약

- Retrofit2: API 인터페이스 정의만 하면 JSON 파싱까지 자동으로 처리해주는 고수준 HTTP 클라이언트
- OkHttp: Retrofit이 내부적으로 사용하는 저수준 네트워크 라이브러리(커넥션, 캐싱, 인터셉터, TLS 처리)
- Retrofit = “편의성”, OkHttp = “성능/세부제어”
- 네트워크는 기본적으로 TCP → TLS → HTTP(REST) 순서로 이루어진다
- HTTPS는 HTTP를 TLS 위에 올린 프로토콜이며, 중간자 공격(MITM)을 방지한다

---

## 2. Retrofit2란 무엇인가?

Retrofit2는 Square에서 만든 타입 안전(type-safe) REST 클라이언트이다.
가장 큰 장점은 “인터페이스만 작성하면 요청/응답이 완성된다”는 점이다.

✔ 주요 특징
1) 인터페이스 기반 API 설계

 ```kotlin
interface ApiService {
    @GET("users")
    suspend fun getUsers(): List<User>
}
```

- HTTP 요청 생성
- JSON 파싱(Gson/Moshi)
- Retrofit → OkHttp를 통해 네트워크 호출이 모두 수행된다.

2) Coroutine 지원 (suspend)

비동기 처리를 Callback 지옥 없이 해결.

```kotlin
val users = api.getUsers()  // suspend
```

3) Converter 선택 가능

- Gson
- Moshi
- Kotlin Serialization 등

4) OkHttp 기반

- Retrofit 자체에는 네트워크 엔진이 없다.
- HTTP 요청 실제 송수신은 OkHttpClient가 담당한다.

---

## 3. OkHttp란 무엇인가?

OkHttp는 안드로이드에서 사실상 표준으로 쓰이는 고성능 HTTP 클라이언트 라이브러리이다.

✔ 주요 기능
- Connection Pooling 
  + TCP 커넥션을 재활용해서 요청 속도 향상(keep-alive)
- HTTP/2 지원 (멀티플렉싱)
  + 한 커넥션으로 여러 요청 병렬 처리 → 지연 시간 감소
- Interceptor 지원
  + 네트워크 요청/응답을 가로채서 로그, 헤더 추가, 인증 토큰 갱신 등 처리 가능
  + ```kotlin
    class LoggingInterceptor : Interceptor {
        override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        println("Request: ${request.url}")
        return chain.proceed(request)
        }
    }
    ```
- Cache 지원
  + 오프라인 캐싱, 재요청 최소화
- TLS(HTTPS) 자동 처리
- Timeouts 제어

---

## 4. Retrofit + OkHttp

```kotlin
val okHttp = OkHttpClient.Builder()
    .addInterceptor(LoggingInterceptor())
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(10, TimeUnit.SECONDS)
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .client(okHttp)
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val api = retrofit.create(ApiService::class.java)
```

---

## 5. Retrofit/OkHttp 구조

```text
Retrofit
   ↓ (API Interface → Request 변환)
OkHttp
   ↓ (TCP 연결, TLS, HTTP 처리)
네트워크
```
Retrofit 은 설계/파싱/비동기 처리 담당, OKHttp는 실제 네트워크 I/O 담당

---

## 6. Retrofit/OkHttp 네트워크 흐름 요약

```text
App 에서 서버로 요청 시 흐름

Retrofit API 호출
  ↓
OkHttp Request 생성
  ↓
TCP 커넥션 연결
  ↓
TLS 핸드셰이크 (HTTPS이면)
  ↓
HTTP 요청 전송
  ↓
HTTP 응답 수신
  ↓
Converter(Gson/Moshi)로 JSON → 객체 변환
  ↓
코루틴 or 콜백으로 결과 반환
```

---

## 7. 실무 팁 (중요!)

- OkHttp Interceptor 활용하기
  + access token 자동 주입
  + 401 발생 시 refresh token 재발급 처리
  + 통신 로그 debugging
- Retrofit + Coroutine + Sealed Class 조합으로 에러 처리
  + ```kotlin
    sealed class ApiResult<out T> {
        data class Success<T>(val data: T): ApiResult<T>()
        data class Error(val throwable: Throwable): ApiResult<Nothing>()
    }
    ```
- HTTPS 인증서 고정(Pinning) 적용
  + 보안 강화 → 금융/보안 앱에서는 필수
- 타임아웃 반드시 명시
  + 네트워크 이슈로 인한 무한 대기 방지
- BaseResponse + 공통 에러 처리 패턴 구축
  + 서버 스펙에 맞춰 Response Wrapper 사용