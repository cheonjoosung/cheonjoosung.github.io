---
title: Android/안드로이드 TextView & EditText 확장함수 모음
tags: [Kotlin]
style: fill
color: dark
description: Android/안드로이드 TextView & EditText 확장함수 모음 – 입력/표시/포맷/UX 향상
---

## ✨ 개요

`TextView`와 `EditText`에서 매번 반복하는 작업을 확장함수로 정리했습니다.
- 표시: 비었을 때 GONE, HTML 표시, 하이라이트, 줄임 등
- 입력: afterTextChanged, IME 액션, 숫자 전용, 길이 제한 등
- UX: 링크 자동, 복사/붙여넣기, 에러 텍스트, 플레이스홀더 등

---

## 1 `TextView`

```kotlin
import android.content.ClipData
import android.content.ClipboardManager
import android.content.Context
import android.graphics.Typeface
import android.text.*
import android.text.method.LinkMovementMethod
import android.text.style.ForegroundColorSpan
import android.text.style.StyleSpan
import android.view.inputmethod.EditorInfo
import android.widget.EditText
import android.widget.TextView
import androidx.annotation.ColorInt
import androidx.core.text.HtmlCompat
import androidx.core.widget.addTextChangedListener
import java.util.Locale

/* ----------------------------- 표시 / 토글 ----------------------------- */

/** 비어있으면 GONE, 있으면 VISIBLE로 세팅 */
fun TextView.setTextOrGone(textOrNull: CharSequence?) {
    if (textOrNull.isNullOrBlank()) { text = ""; this.visibility = TextView.GONE }
    else { text = textOrNull; this.visibility = TextView.VISIBLE }
}

/** 비었을 때 플레이스홀더 표시(회색), 실제 값은 null 반환 */
fun TextView.setPlaceholder(textOrNull: CharSequence?, placeholder: CharSequence, @ColorInt color: Int) {
    if (textOrNull.isNullOrBlank()) {
        setTextColor(color); text = placeholder
    } else text = textOrNull
}

/** HTML 간단 표시 (+링크 터치 가능) */
fun TextView.setHtml(html: String, compact: Boolean = true) {
    text = HtmlCompat.fromHtml(
        html,
        if (compact) HtmlCompat.FROM_HTML_MODE_COMPACT else HtmlCompat.FROM_HTML_MODE_LEGACY
    )
    movementMethod = LinkMovementMethod.getInstance()
}

/** 텍스트 일부 색상/볼드 하이라이트 (최초 매칭 1개) */
fun TextView.highlightOnce(query: String, @ColorInt color: Int, bold: Boolean = false, ignoreCase: Boolean = true) {
    val src = text?.toString().orEmpty()
    val idx = src.indexOf(query, ignoreCase = ignoreCase)
    if (idx < 0) return
    val span = SpannableString(src)
    span.setSpan(ForegroundColorSpan(color), idx, idx + query.length, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
    if (bold) span.setSpan(StyleSpan(Typeface.BOLD), idx, idx + query.length, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
    text = span
}

/** 가운데 말줄임 (파일명/이메일에 유용) */
fun TextView.ellipsizeMiddle(maxLen: Int, ellipsis: String = "…") {
    val s = text?.toString().orEmpty()
    if (s.length <= maxLen) return
    val keep = maxLen - ellipsis.length
    val head = keep / 2
    val tail = keep - head
    text = s.take(head) + ellipsis + s.takeLast(tail)
}

/** 전/후 접두·접미 붙이기 (null/blank는 무시) */
fun TextView.setWithAffix(prefix: String? = null, value: CharSequence?, suffix: String? = null) {
    val v = value?.toString().orEmpty()
    text = if (v.isBlank()) "" else "${prefix.orEmpty()}$v${suffix.orEmpty()}"
}

/** 밑줄/취소선 토글 */
fun TextView.setUnderline(enable: Boolean = true) { paint.isUnderlineText = enable }
fun TextView.setStrike(enable: Boolean = true) { paint.isStrikeThruText = enable }

/* ----------------------------- 링크/길이/포맷 ----------------------------- */

/** URL/전화 자동 링크 */
fun TextView.autoLinkify(mask: Int = Linkify.WEB_URLS or Linkify.PHONE_NUMBERS) {
    autoLinkMask = mask
    linksClickable = true
    movementMethod = LinkMovementMethod.getInstance()
}

/** 최대 길이 필터 설정 */
fun TextView.setMaxLength(max: Int) {
    filters = (filters?.toMutableList() ?: mutableListOf()).apply {
        removeAll { it is InputFilter.LengthFilter }
        add(InputFilter.LengthFilter(max))
    }.toTypedArray()
}

/** 첫 글자만 대문자(로케일 반영) */
fun TextView.capitalizeFirst(locale: Locale = Locale.getDefault()) {
    val s = text?.toString().orEmpty()
    text = s.replaceFirstChar { if (it.isLowerCase()) it.titlecase(locale) else it.toString() }
}

/** 모두 소문자/대문자 (로케일 반영) */
fun TextView.toLower(locale: Locale = Locale.getDefault()) { text = text?.toString()?.lowercase(locale) }
fun TextView.toUpper(locale: Locale = Locale.getDefault()) { text = text?.toString()?.uppercase(locale) }

/* ----------------------------- 입력(EditText) ----------------------------- */

/** afterTextChanged 간단 콜백 */
inline fun EditText.afterTextChanged(crossinline block: (String) -> Unit) {
    addTextChangedListener { block(it?.toString().orEmpty()) }
}

/** before/ on/ after 모두 한번에 */
fun EditText.onTextChanged(
    before: ((CharSequence?, Int, Int, Int) -> Unit)? = null,
    on: ((CharSequence?, Int, Int, Int) -> Unit)? = null,
    after: ((Editable?) -> Unit)? = null
) = addTextChangedListener(
    onTextChanged = on,
    beforeTextChanged = before,
    afterTextChanged = after
)

/** IME 액션 처리 (DONE/SEARCH 등) */
fun EditText.onImeAction(action: Int = EditorInfo.IME_ACTION_DONE, run: () -> Unit) {
    setOnEditorActionListener { _, a, _ -> if (a == action) { run(); true } else false }
}

/** 숫자만 입력 */
fun EditText.digitsOnly() { inputType = InputType.TYPE_CLASS_NUMBER }

/** 커서 맨 끝으로 이동 */
fun EditText.moveCursorToEnd() { setSelection(text?.length ?: 0) }

/** 텍스트 세팅 + 커서 끝 */
fun EditText.setTextAndMoveEnd(value: CharSequence?) { setText(value); moveCursorToEnd() }

/** 값 검증에 따라 버튼 enable/alpha 동기화 */
fun EditText.bindEnable(target: TextView, validator: (String) -> Boolean = { it.isNotBlank() }, disabledAlpha: Float = 0.5f) {
    val refresh: (String) -> Unit = {
        val ok = validator(it)
        target.isEnabled = ok
        target.alpha = if (ok) 1f else disabledAlpha
    }
    afterTextChanged(refresh)
    refresh(text?.toString().orEmpty())
}

/** 입력값을 실시간 포맷(예: 전화번호) 하되 커서 위치 유지 */
fun EditText.formatWith(
    formatter: (String) -> String
) {
    var internal = false
    addTextChangedListener {
        if (internal) return@addTextChangedListener
        val raw = it?.toString().orEmpty()
        val formatted = formatter(raw)
        if (formatted != raw) {
            val pos = selectionStart
            internal = true
            setText(formatted)
            setSelection(minOf(formatted.length, pos + (formatted.length - raw.length)))
            internal = false
        }
    }
}

/* ----------------------------- 복사/붙여넣기/오류 ----------------------------- */

/** 텍스트를 클립보드로 복사 */
fun TextView.copyToClipboard(label: String = "text") {
    val s = text?.toString().orEmpty()
    if (s.isBlank()) return
    val cb = context.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
    cb.setPrimaryClip(ClipData.newPlainText(label, s))
}

/** 붙여넣기(있으면 맨 끝에 추가) */
fun EditText.pasteFromClipboard() {
    val cb = context.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
    val clip = cb.primaryClip
    val paste = clip?.takeIf { it.itemCount > 0 }?.getItemAt(0)?.coerceToText(context)?.toString() ?: return
    append(paste); moveCursorToEnd()
}

/** 오류 메시지 헬퍼 (TextInputLayout와 조합 용이) */
fun TextView.setErrorText(msg: CharSequence?) {
    visibility = if (msg.isNullOrBlank()) TextView.GONE else TextView.VISIBLE
    text = msg
}
```

---

## 2 사용 예

```kotlin
// 1) 표시/하이라이트/줄임
title.setTextOrGone(user.name)
desc.setHtml("<b>안내</b>: <a href='https://example.com'>자세히</a>")
badge.highlightOnce("NEW", color = 0xFF2962FF.toInt(), bold = true)
fileName.apply { text = "very_long_filename.png"; ellipsizeMiddle(20) }

// 2) 입력/검증
searchEdit.onImeAction(EditorInfo.IME_ACTION_SEARCH) { doSearch(searchEdit.text.toString()) }
phoneEdit.digitsOnly()
phoneEdit.formatWith { raw -> raw.filter(Char::isDigit).let { if (it.length == 11) "${it.substring(0,3)}-${it.substring(3,7)}-${it.substring(7)}" else it } }
idEdit.bindEnable(loginButton) { it.length >= 3 && pwEdit.text?.length ?: 0 >= 8 }

// 3) 링크/길이
content.autoLinkify()
nick.setMaxLength(12)

// 4) 클립보드/에러
copyView.setOnClickListener { resultText.copyToClipboard("result") }
errorText.setErrorText("아이디를 입력하세요")
```

---

## 3 팁 및 주의사항

- `setHtml`은 신뢰된 콘텐츠에만 사용(외부 입력은 서버/클라이언트에서 반드시 sanitize).
- `formatWith`는 무한 루프 방지를 위해 내부 플래그 사용—포맷 함수는 빠르게 동작하도록.
- `autoLinkify` 사용 시 클릭 충돌이 생기면 `movementMethod/clickable` 조합을 조정하세요.
- `bindEnable`는 단일 `EditText` 기준입니다. 다수 입력을 묶으려면 각 `EditText.afterTextChanged`에서 공통 검사 함수를 호출하세요.