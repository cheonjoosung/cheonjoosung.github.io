---
title: Kotlin/코틀린 실무에서 자주 쓰는 문자열 확장함수 30선
tags: [Kotlin]
style: fill
color: dark
description: Kotlin/코틀린 실무에서 자주 쓰는 문자열 확장함수 30선 – 클린한 String 유틸 모음
---

## ✨ 개요

> 안드로이드/백엔드 공통으로 바로 가져다 쓰는 **문자열 확장함수 레시피** 모음입니다.  
> 검증·정규화·포맷·마스킹·케이스 변환·인코딩·해시까지 실전에 바로 쓰세요.

---

## 1 사용 예제

```kotlin
import android.util.Patterns
import java.net.IDN
import java.security.MessageDigest
import java.text.Normalizer
import java.util.*
import android.util.Base64

// --- 검증 / 체크 ---

/** 비어있거나 공백만 있으면 true */
val CharSequence?.isNullOrBlankEx: Boolean
    get() = this == null || this.isBlank()

/** 이메일 형식인지 (Android Patterns 활용) */
fun CharSequence?.isEmail(): Boolean =
    !isNullOrBlankEx && Patterns.EMAIL_ADDRESS.matcher(this!!).matches()

/** URL 형식인지 (http/https) */
fun CharSequence?.isHttpUrl(): Boolean =
    !isNullOrBlankEx && Patterns.WEB_URL.matcher(this!!).matches() &&
            (this!!.startsWith("http://") || this.startsWith("https://"))

/** 숫자만으로 구성? */
fun CharSequence?.isDigits(): Boolean = !isNullOrBlankEx && this!!.all { it.isDigit() }

/** 영문/숫자/_ 만 허용 (아이디 등) */
fun CharSequence?.isSlugSafe(): Boolean =
    !isNullOrBlankEx && this!!.all { it.isLetterOrDigit() || it == '_' }


// region --- 정규화 / 클린업 ---

/** 앞뒤/연속 공백 정리: 모든 공백을 단일 스페이스로 */
fun CharSequence?.normalizeWhitespace(): String =
    this?.trim()?.replace(Regex("\\s+"), " ") ?: ""

/** 제어문자 제거 (줄바꿈은 유지) */
fun CharSequence?.stripControlChars(): String =
    this?.filter { it == '\n' || it.code >= 32 } ?: ""

/** 이모지 제거 (기본 Plane 외 크게 제거) */
fun CharSequence?.removeEmoji(): String =
    this?.filter { Character.getType(it) != Character.SURROGATE } ?: ""

/** HTML 태그 제거 (간단판) */
fun CharSequence?.stripHtml(): String =
    this?.replace(Regex("<[^>]*>"), "")?.replace("&nbsp;", " ") ?: ""

// 유니코드 정규화(한글 자모 결합, 라틴 악센트 제거용)
fun CharSequence?.nfkd(): String = this?.let { Normalizer.normalize(it, Normalizer.Form.NFKD) } ?: ""

/** 악센트/다이어크리틱 제거 */
fun CharSequence?.removeDiacritics(): String =
    nfkd().replace(Regex("\\p{M}+"), "")

// endregion

// region --- 케이스 변환 / 슬러그 ---

/** lowerCamelCase → snake_case */
fun CharSequence?.toSnakeCase(): String =
    this?.replace(Regex("([a-z0-9])([A-Z])"), "$1_$2")?.lowercase(Locale.getDefault()) ?: ""

/** any → kebab-case (공백/언더바를 하이픈으로) */
fun CharSequence?.toKebabCase(): String =
    this.normalizeWhitespace().lowercase(Locale.getDefault())
        .replace(Regex("[_ ]+"), "-")
        .replace(Regex("[^a-z0-9-]"), "")

/** 슬러그: 도메인/URL 경로 등에 안전한 형태 */
fun CharSequence?.toSlug(): String =
    this.removeDiacritics().toKebabCase()

/** 첫 글자 대문자 */
fun CharSequence?.capitalizeFirst(): String =
    if (this.isNullOrBlank()) "" else this!!.replaceFirstChar { it.titlecase(Locale.getDefault()) }

/** 첫 글자 소문자 */
fun CharSequence?.decapitalizeFirst(): String =
    if (this.isNullOrBlank()) "" else this!!.replaceFirstChar { it.lowercase(Locale.getDefault()) }


// region --- 마스킹 / 포맷 ---

/** 이메일 마스킹: u***@d***.com */
fun CharSequence?.maskEmail(): String {
    if (this.isNullOrBlank() || !this.isEmail()) return this?.toString() ?: ""
    val (u, d) = this!!.split("@", limit = 2)
    val maskedUser = when {
        u.length <= 2 -> u.first() + "*"
        else -> u.first() + "*".repeat(u.length - 2) + u.last()
    }
    val domainParts = d.split(".", limit = 2)
    val maskedDomain = domainParts[0].first() + "*".repeat(maxOf(1, domainParts[0].length - 1)) +
            if (domainParts.size > 1) ".${domainParts[1]}" else ""
    return "$maskedUser@$maskedDomain"
}

/** 가운데 마스킹(주민/전화 등) */
fun CharSequence?.maskMiddle(visibleHead: Int = 3, visibleTail: Int = 2, mask: Char = '*'): String {
    val s = this?.toString() ?: return ""
    if (s.length <= visibleHead + visibleTail) return s
    val mid = s.length - visibleHead - visibleTail
    return s.take(visibleHead) + mask.toString().repeat(mid) + s.takeLast(visibleTail)
}

/** 10~11자리 한국 휴대전화 포맷 */
fun CharSequence?.formatKoreanPhone(): String {
    val digits = this?.filter(Char::isDigit) ?: return ""
    return when (digits.length) {
        10 -> "${digits.substring(0,3)}-${digits.substring(3,6)}-${digits.substring(6)}"
        11 -> "${digits.substring(0,3)}-${digits.substring(3,7)}-${digits.substring(7)}"
        else -> digits
    }
}

/** 화면 표시용 말줄임 (문자 기준) */
fun CharSequence?.ellipsize(maxLen: Int, ellipsis: String = "…"): String {
    val s = this?.toString() ?: return ""
    return if (s.length <= maxLen) s else s.take(maxLen - ellipsis.length) + ellipsis
}


// region --- 인코딩 / 해시 ---

/** Base64 인코딩 */
fun ByteArray.encodeBase64(): String = Base64.encodeToString(this, Base64.NO_WRAP)
/** Base64 디코딩 */
fun String.decodeBase64OrNull(): ByteArray? = try { Base64.decode(this, Base64.NO_WRAP) } catch (_: Exception) { null }

/** SHA-256 해시(hex) */
fun CharSequence?.sha256Hex(): String {
    val md = MessageDigest.getInstance("SHA-256")
    val bytes = md.digest((this ?: "").toByteArray(Charsets.UTF_8))
    return bytes.joinToString("") { "%02x".format(it) }
}


// region --- HTML/URL 안전 처리 ---

/** URL 안전 쿼리 인코딩 (UTF-8) */
fun String.urlEncode(): String = java.net.URLEncoder.encode(this, "UTF-8")

/** 국제화 도메인 → Punycode */
fun String.toPunycode(): String = IDN.toASCII(this)

/** 간단 태그 이스케이프 */
fun String.escapeHtmlSimple(): String = this
    .replace("&", "&amp;")
    .replace("<", "&lt;")
    .replace(">", "&gt;")
    .replace("\"", "&quot;")
    .replace("'", "&#39;")


// region --- 기타 편의 ---

/** 지정 범위 문자열 교체: [start,end) 구간을 repl로 */
fun String.replaceRangeSafe(start: Int, end: Int, repl: String): String {
    val s = start.coerceIn(0, length)
    val e = end.coerceIn(s, length)
    return substring(0, s) + repl + substring(e, length)
}

/** 다중 토큰 치환: map의 key들을 순차 치환 */
fun String.replaceTokens(vars: Map<String, String>, prefix: String = "{{", suffix: String = "}}"): String {
    var out = this
    vars.forEach { (k, v) -> out = out.replace("$prefix$k$suffix", v) }
    return out
}

/** 바이트 길이(UTF-8) */
fun CharSequence?.utf8Length(): Int = this?.toString()?.toByteArray(Charsets.UTF_8)?.size ?: 0
```

---

## 2 사용 예 (스니펫)

```kotlin
val email = "user.name@example.com"
println(email.isEmail())                  // true
println(email.maskEmail())                // u*******@e*******.com

val phone = "01012345678"
println(phone.formatKoreanPhone())        // 010-1234-5678
println(phone.maskMiddle())               // 010******78

val messy = "  안녕   세상 \n  hola "
println(messy.normalizeWhitespace())      // "안녕 세상 hola"

val html = "<b>Hello</b> & welcome"
println(html.stripHtml())                 // "Hello  welcome"
println(html.escapeHtmlSimple())          // "&lt;b&gt;Hello&lt;/b&gt; &amp; welcome"

println("HelloWorld".toSnakeCase())       // hello_world
println("커피 크림".toSlug())              // keopi-keulim

println("secret".sha256Hex())             // a SHA-256 hex
println("한글".utf8Length())               // 6
```

---

## 3 안전·국제화 팁

- 검증은 방어의 일부입니다. 이메일/URL은 포맷 통과 ≠ 실제 유효. 서버 검증과 함께 쓰세요.
- 국제 도메인은 toPunycode()를 거쳐 네트워크 계층에 전달하면 안전합니다.
- XSS/HTML 인젝션 위험이 있는 텍스트는 반드시 컨텍스트 맞춤 이스케이프(여기선 단순판 제공).
- 개인정보는 maskEmail, maskMiddle처럼 표시와 저장을 분리하세요.

---

## 4 결론

- 자주 쓰는 문자열 로직을 확장함수로 모듈화하면 가독성·재사용성이 급상승합니다.
- 필요에 따라 프로젝트 규칙(아이디 정책, 슬러그 규칙, 마스킹 스펙)을 반영해 커스터마이즈 하세요.