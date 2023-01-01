---
title: Android Retrofit2 Interceptor request Header 추가 | 레트로핏2 인터셉터 헤더 추가
tags: [Android, Kotlin]
style: fill
color: dark
description: Android Retrofit2 Interceptor request Header 추가 | 레트로핏2 인터셉터 헤더 추가
---

## Android Retrofit2 Interceptor request Header 추가 변경 & response 처리
```kotlin
val retrofit: TrendNewsService = Retrofit.Builder()
    .baseUrl(basedUrl)
    .addConverterFactory(GsonConverterFactory.create(
        GsonBuilder().setLenient().create()
    ))
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
    .client(
        OkHttpClient.Builder()
            .addInterceptor(
                HttpLoggingInterceptor().apply {
                    this.level = HttpLoggingInterceptor.Level.BODY
                })
            .addInterceptor(CustomInterceptor()) // 만들어야 할 custom Interceptor
            .build()
    )
    .build()
    .create(TrendNewsService::class.java)
```
- android retrofit2 interceptor 를 사용하여 body 값을 로깅하거나 request header 를 추가 하거나 등
  다양한 방법으로 사용할 수 있다.
- CustomInterceptor() 를 만들어서 request header 변경 & response 처리 예제를 만들어 봤습니다.


```kotlin
class CustomInterceptor : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        
        // Request Header 추가
        val request = chain.request()
        request.newBuilder().addHeader("content-type", "charset=UTF-8").build()
        request.newBuilder().addHeader("what-you-want-header", "data").build()
        
        // Response 값 처리
        val response = chain.proceed(request)

        // 데이터의 정상 여부에 따라 결과를 변환하여 처리
        return if (response.code == 200) {
            val bodyString = response.body?.string()?.replace(")]}',", "")?.trim() ?: ""

            val body =
                convertedBody.toResponseBody("application/json; charset=utf-8".toMediaTypeOrNull())
            response.newBuilder().body(body).build()
        } else {
            response
        }
    }

}
```
- Interceptor 를 말 그대로 가로채기다
- request 쪽을 보면 네트워크 요청을 하기 전에 header 값을 세팅하여 보낼 수 있다.  
- response 쪽은 데이터가 왔을 때 바로 전달하는 것이 아니라 1차 가공을 한 후에 내보낼 수 있다.
- 네트워크를 요청하고 response body 값 시작이 )]}', 로 시작하면 json 형태가 아니므로 네트워크 모델로 받을 수 없다.
- response body 에서 )]}', -> 빈 문자열로 변경하여 response body 를 만들어서 내보낸다.
- response 의 여러 데이터 중 필요한 부분만 뽑아서 내보낼 수도 있을 것이다. 

## 결론
- retrofit2 Interceptor 활용법은 익히자.
