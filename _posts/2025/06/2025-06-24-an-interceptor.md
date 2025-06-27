---
title: Android Retrofit Interceptor 완벽 가이드
tags: [ Android ]
style: fill
color: dark
description: Retrofit에서 OkHttp Interceptor를 활용하여 공통 헤더, 로깅, 인증 처리 등을 구현하는 방법을 예제 중심으로 쉽게 설명합니다.
---

## ✨ 개요

Retrofit은 기본적으로 OkHttp를 기반으로 동작하기 때문에, OkHttp의 Interceptor 기능을 그대로 사용할 수 있습니다. 
Interceptor는 네트워크 요청이나 응답을 가로채어 공통 헤더 추가, 로깅, 토큰 처리 등에 활용됩니다.

---

## 1. 🧩 Interceptor의 역할

- 모든 네트워크 요청/응답을 중앙에서 제어 가능
- 공통 헤더 추가 (예: Authorization)
- 응답에 대한 공통 처리 (예: 토큰 만료 시 재로그인)
- 디버그를 위한 로그 출력

---

## 2. ⚙️ Interceptor 구현 예제

```kotlin
class AuthInterceptor : Interceptor {
  override fun intercept(chain: Interceptor.Chain): Response {
    val request = chain.request().newBuilder()
      .addHeader("Authorization", "Bearer token123")
      .build()
    return chain.proceed(request)
  }
}

val okHttpClient = OkHttpClient.Builder()
  .addInterceptor(AuthInterceptor())
  .build()

val retrofit = Retrofit.Builder()
  .baseUrl("https://api.example.com")
  .client(okHttpClient)
  .addConverterFactory(GsonConverterFactory.create())
  .build()
```

---

## 3. 📝 주의사항

- Interceptor는 모든 요청에 적용되므로, 조건부 분기 처리에 주의해야 함
- 인증 토큰 갱신 로직을 Interceptor에 작성하면 무한루프 발생 가능성 → 별도 Authenticator 권장
- 네트워크 타임아웃이 긴 요청에도 Interceptor가 영향을 줄 수 있음

---

## 4. 🧾 결론

Retrofit의 Interceptor는 네트워크 로직을 중앙에서 통합 관리하고, 코드 중복을 줄이는 데 강력한 도구입니다. 
공통 헤더나 로깅, 인증과 같은 단일 책임을 구현할 때 적극적으로 활용하면 유지보수성이 높아집니다.

| 용도          | Interceptor 활용 예                     |
| ----------- | ------------------------------------ |
| 인증 토큰 자동 추가 | 헤더에 Authorization 삽입                 |
| 요청 공통 헤더    | User-Agent, Locale, App-Version 등 삽입 |
| 응답 가공       | 응답 코드에 따른 처리                         |
| 로그 기록       | `HttpLoggingInterceptor` 사용          |
| 토큰 갱신       | `401` 응답 시 자동 재요청                    |