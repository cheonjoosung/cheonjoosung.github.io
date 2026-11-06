---
title: (Android/OkHttp) OkHttp Cache 전략 — 오프라인 캐시
tags: [ Android, OkHttp ]
style: fill
color: dark
description: OkHttp의 HTTP 캐시 설정과 Cache-Control 헤더 처리, 오프라인 시 캐시 사용, 네트워크 인터셉터를 이용한 캐시 정책 제어를 정리합니다.
---

---

## 1. OkHttp 캐시란?

HTTP 응답을 디스크에 저장해 **같은 요청을 네트워크 없이** 재사용하는 기능입니다.
서버가 내려주는 `Cache-Control`, `ETag`, `Last-Modified` 헤더를 따라 동작합니다.

```
요청 → [캐시 유효?] ─ 예 → 캐시 응답 반환 (네트워크 사용 안 함)
                    └ 아니오 → 서버 요청 → 응답을 캐시에 저장
```

---

## 2. 캐시 설정

```kotlin
val cacheDir = File(context.cacheDir, "http_cache")
val cache = Cache(cacheDir, 10L * 1024 * 1024)     // 10MB

val client = OkHttpClient.Builder()
    .cache(cache)
    .build()
```

> **캐시는 기본으로 꺼져 있습니다.** `Cache`를 지정하지 않으면 아무것도 저장되지 않습니다. 서버가 캐시를 허용하는 헤더를 주어야 저장됩니다.

---

## 3. 서버 헤더가 결정한다

| 헤더 | 의미 |
|------|------|
| `Cache-Control: max-age=60` | 60초 동안은 네트워크 없이 캐시 사용 |
| `Cache-Control: no-cache` | 저장은 하되 **매번 서버에 재검증** |
| `Cache-Control: no-store` | 저장 금지 |
| `ETag` / `Last-Modified` | 만료 후 재검증 (변경 없으면 304, 본문 재전송 없음) |

서버가 이 헤더를 주지 않는다면 OkHttp는 캐시를 활용하지 못합니다. 이때 클라이언트에서 정책을 덧씌웁니다.

---

## 4. 네트워크 인터셉터로 서버 응답 헤더 덮어쓰기

서버를 수정할 수 없을 때 **응답의 캐시 헤더를 클라이언트에서 지정**합니다.

```kotlin
class CacheControlInterceptor(private val maxAgeSec: Int) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        return response.newBuilder()
            .removeHeader("Pragma")
            .header("Cache-Control", "public, max-age=$maxAgeSec")
            .build()
    }
}

val client = OkHttpClient.Builder()
    .cache(cache)
    .addNetworkInterceptor(CacheControlInterceptor(60))   // 반드시 networkInterceptor
    .build()
```

> 응답 캐시 헤더 수정은 `addInterceptor`가 아니라 **`addNetworkInterceptor`** 여야 합니다. 캐시 저장 직전의 실제 네트워크 응답을 건드려야 하기 때문입니다.

---

## 5. 오프라인일 때 오래된 캐시 허용

기본 동작은 만료된 캐시를 쓰지 않습니다. 오프라인이면 **만료돼도 캐시를 쓰도록** 요청을 바꿉니다.

```kotlin
class OfflineCacheInterceptor(private val context: Context) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()
        if (!isOnline(context)) {
            request = request.newBuilder()
                .cacheControl(
                    CacheControl.Builder()
                        .maxStale(7, TimeUnit.DAYS)       // 7일까지 만료된 캐시 허용
                        .build()
                )
                .build()
        }
        return chain.proceed(request)
    }
}

val client = OkHttpClient.Builder()
    .cache(cache)
    .addInterceptor(OfflineCacheInterceptor(context))      // 일반 interceptor
    .addNetworkInterceptor(CacheControlInterceptor(60))
    .build()
```

| 인터셉터 | 시점 | 용도 |
|---------|------|------|
| `addInterceptor` | 요청 전/응답 후 (캐시 전) | 요청에 `maxStale` 등 지정 |
| `addNetworkInterceptor` | 실제 네트워크 구간 | 응답 캐시 헤더 덮어쓰기 |

---

## 6. 요청별 캐시 제어

```kotlin
interface ArticleApi {
    // 이 요청만 항상 서버에서 새로 받기
    @Headers("Cache-Control: no-cache")
    @GET("articles/latest")
    suspend fun latest(): List<Article>

    // 강제 새로고침 (pull-to-refresh)
    @GET("articles")
    suspend fun list(@Header("Cache-Control") cacheControl: String? = null): List<Article>
}

api.list()                                                        // 캐시 정책에 따름
api.list(CacheControl.FORCE_NETWORK.toString())                   // 새로고침
```

- `CacheControl.FORCE_NETWORK`: 캐시 무시, 항상 네트워크
- `CacheControl.FORCE_CACHE`: 캐시만 사용 (없으면 504)

---

## 7. 주의사항

- **POST 응답은 캐시되지 않습니다.** GET 위주의 조회 API에 적합합니다
- 인증이 필요한 응답을 캐시할 때 **사용자 간 데이터 혼선**이 없는지 확인하세요 (`Vary: Authorization`, 로그아웃 시 `cache.evictAll()`)
- 민감한 데이터는 `no-store`로 저장을 막고, 로그아웃 시 캐시를 비우세요
- 실시간성이 중요한 데이터에 긴 `max-age`를 주면 오래된 화면이 보입니다

---

## 8. 오프라인 앱은 Room이 더 적합할 수 있다

OkHttp 캐시는 **HTTP 응답 단위**의 투명한 캐시입니다. 오프라인에서 목록 검색/정렬/수정이 필요하다면 데이터를 DB에 저장하는 방식(Room + Repository)이 더 알맞습니다. 간단한 조회 화면의 재진입 속도와 오프라인 조회 정도는 OkHttp 캐시로 충분합니다.

---

## 9. 정리

- `Cache(dir, size)`를 설정해야 동작, 서버의 `Cache-Control`/`ETag` 헤더를 따름
- 서버를 못 바꿀 땐 **`addNetworkInterceptor`로 응답 헤더 덮어쓰기**
- 오프라인에서는 요청에 `maxStale`을 지정해 **만료된 캐시도 사용**
- 요청별로 `no-cache`, `FORCE_NETWORK`로 새로고침 제어
- GET만 캐시, 민감 데이터는 `no-store`와 로그아웃 시 `evictAll()`
