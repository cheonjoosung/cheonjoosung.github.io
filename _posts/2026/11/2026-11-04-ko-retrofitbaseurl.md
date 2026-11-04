---
title: (Android/Retrofit) Retrofit 동적 BaseUrl 변경 패턴
tags: [ Android, Retrofit ]
style: fill
color: dark
description: Retrofit에서 런타임에 BaseUrl을 바꾸는 방법으로 Interceptor 치환, @Url 파라미터, 여러 Retrofit 인스턴스 구성 패턴을 비교해 정리합니다.
---

---

## 1. 왜 동적 BaseUrl이 필요한가?

`Retrofit.Builder().baseUrl(...)`은 **한 번 설정하면 변경할 수 없습니다.**
하지만 다음처럼 런타임에 서버 주소가 바뀌는 경우가 있습니다.

- 개발/스테이징/운영 서버를 앱 안에서 전환 (QA용 설정 화면)
- 사용자가 직접 서버 주소를 입력하는 앱 (자체 호스팅 서비스)
- 서버 선택(리전)에 따라 주소가 달라지는 경우

---

## 2. 방법 1 — 호출별 @Url

특정 요청만 전체 URL을 지정합니다. 가장 단순합니다.

```kotlin
interface DownloadApi {
    @GET
    suspend fun download(@Url url: String): ResponseBody

    @GET
    suspend fun getJson(@Url url: String, @Query("id") id: Long): Item
}

api.download("https://cdn.example.com/files/a.zip")
```

> `@Url`은 **첫 번째 파라미터**여야 하고, 주어진 값이 절대 URL이면 baseUrl을 완전히 무시합니다. 상대 경로면 baseUrl 기준으로 결합됩니다.

적합한 경우: 일부 요청만 다른 주소(CDN 파일, 페이지네이션의 `next` URL 등)를 쓸 때.

---

## 3. 방법 2 — Interceptor로 호스트 치환

모든 요청의 호스트를 **인터셉터에서 현재 설정값으로 교체**합니다. API 코드는 그대로 둡니다.

```kotlin
@Singleton
class BaseUrlProvider @Inject constructor() {
    @Volatile var baseUrl: HttpUrl = "https://api.example.com/".toHttpUrl()
}

class HostSelectionInterceptor @Inject constructor(
    private val provider: BaseUrlProvider
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val target = provider.baseUrl

        val newUrl = request.url.newBuilder()
            .scheme(target.scheme)
            .host(target.host)
            .port(target.port)
            .build()

        return chain.proceed(request.newBuilder().url(newUrl).build())
    }
}
```

```kotlin
// 사용
baseUrlProvider.baseUrl = "https://staging.example.com/".toHttpUrl()
// 이후 모든 호출이 staging 서버로 전송됨
```

**장점**: Retrofit/API 인스턴스를 다시 만들지 않고 즉시 전환, 구조가 단순합니다.
**한계**: 경로 접두사(`/v1/`)까지 바뀌는 경우는 `encodedPath` 조작이 더 필요합니다.

---

## 4. 방법 3 — Retrofit 인스턴스를 교체

주소가 바뀔 때 **새 Retrofit을 만들어** API를 다시 생성합니다.

```kotlin
@Singleton
class ApiFactory @Inject constructor(private val client: OkHttpClient) {

    private val cache = ConcurrentHashMap<String, Retrofit>()

    fun <T> create(baseUrl: String, service: Class<T>): T {
        val retrofit = cache.getOrPut(baseUrl) {
            Retrofit.Builder()
                .baseUrl(baseUrl)
                .client(client)                      // OkHttpClient는 재사용 (커넥션 풀 공유)
                .addConverterFactory(GsonConverterFactory.create())
                .build()
        }
        return retrofit.create(service)
    }
}
```

```kotlin
val api = apiFactory.create("https://${userInputHost}/", UserApi::class.java)
```

**장점**: 주소마다 경로/컨버터를 다르게 설정할 수 있어 가장 유연합니다.
**주의**: `OkHttpClient`는 **하나를 공유**하세요. Retrofit마다 새로 만들면 스레드/커넥션 풀이 중복됩니다.

---

## 5. 방법 비교

| 방법 | 전환 방식 | 적합한 경우 |
|------|----------|------------|
| `@Url` | 호출마다 지정 | 일부 요청만 다른 주소 |
| Interceptor 치환 | 전역 설정값 변경 | 환경 전환, 전체 요청 주소 변경 |
| Retrofit 교체/팩토리 | 인스턴스 생성 | 주소마다 설정이 다르거나 여러 서버 동시 사용 |

---

## 6. 사용자 입력 주소 — 검증 필수

사용자가 입력한 주소로 요청을 보낼 때는 **안전성 검증**이 필요합니다.

```kotlin
fun parseBaseUrl(input: String): HttpUrl? {
    val normalized = if (input.endsWith("/")) input else "$input/"
    val url = normalized.toHttpUrlOrNull() ?: return null     // 형식 검증
    if (url.scheme != "https") return null                    // 평문 HTTP 금지
    return url
}
```

- `baseUrl`은 **반드시 `/`로 끝나야** 합니다 (아니면 `IllegalArgumentException`)
- 운영 앱은 `https`만 허용하고, 사용자 입력 호스트로 **인증 토큰을 보내지 않도록** 주의하세요 (토큰 유출 위험)

---

## 7. 정리

- `baseUrl`은 빌드 후 변경 불가 → 필요하면 **@Url, Interceptor, 인스턴스 교체** 중 선택
- 일부 요청만 다르면 `@Url`, 환경 전환은 **Interceptor 치환**이 가장 간단
- 주소마다 설정이 다르면 팩토리로 Retrofit을 캐시하되 **OkHttpClient는 공유**
- 사용자 입력 주소는 `/` 종료, https 강제, 토큰 전송 범위를 검증
