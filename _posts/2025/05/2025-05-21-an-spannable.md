---
title: 안드로이드 TextView Spannable - 특정 단어 강조 및 링크 처리하기
tags: [Android]
style: fill
color: dark
description: Android TextView에서 Spannable을 활용해 특정 단어 강조, 색상 변경, 클릭 이벤트 등 실전 예제로 정리합니다.
---

## ✨ 개요

`TextView`는 텍스트 전체에 스타일을 적용할 수 있을 뿐만 아니라, **Spannable**을 활용해 **텍스트 내 특정 단어에만** 동적으로 스타일링이나 클릭 이벤트를 지정할 수 있습니다.

이번 포스트에서는 `SpannableString`을 사용해 **특정 단어 강조, 색상 변경, 밑줄, 링크, 클릭 이벤트** 등을 적용하는 실전 예제를 정리합니다.

---

## 1. ✅ TextView Programmatically 기능

- 취소선, 밑줄, 볼드, 기울임, 색상 변경, 배경색 변경, 글자 크기 변경

---

## 2. ✅ TextView Programmatically 코드

### 2.1 Spannable 기본 코드

```kotlin
private fun setSpannable(textView: TextView) {
  val text = "문의는 고객센터 또는 이메일로 주세요."
  val spannable = SpannableString(text)
  val keyword = "고객센터"

  val start = text.indexOf(keyword)
  val end = start + keyword.length

  spannable.setSpan(
    ForegroundColorSpan(Color.BLUE), // 글자색 변경
    start, end,
    Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
  )
}
```
- "고객센터" 텍스트만 파란색으로 강조됨

### 2.2 Spannable 클릭 이벤트 추가 (ClickableSpan)

```kotlin
private fun setSpannable(textView: TextView) {
  val text = "문의는 고객센터 또는 이메일로 주세요."
  val spannable = SpannableString(text)
  val keyword = "고객센터"

  val start = text.indexOf(keyword)
  val end = start + keyword.length

  spannable.setSpan(
    object : ClickableSpan() { // 클릭 이벤트 추가
      override fun onClick(widget: View) {
        Toast.makeText(applicationContext, "고객센터 클릭됨", Toast.LENGTH_SHORT).show()
      }
    },
    start, end,
    Spanned.SPAN_EXCLUSIVE_EXCLUSIVE
  )

  textView.text = spannable
  textView.movementMethod = LinkMovementMethod.getInstance() // 없는 경우 클릭 이벤트 작동 안함
}
```
- movementMethod가 없으면 클릭 이벤트가 작동하지 않음

---

## 3.✅  **다양한 Span 예제 모음**

### 3.1 굵은 글씨 (styleSpan)

```kotlin
spannable.setSpan(StyleSpan(Typeface.BOLD), start, end, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
```

### 3.2 밑줄 (UnderlineSpan)

```kotlin
spannable.setSpan(UnderlineSpan(), start, end, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
```

### 3.3 링크처럼 보이게 만들기

```kotlin
spannable.setSpan(object : ClickableSpan() {
    override fun onClick(widget: View) {
        // 원하는 동작 실행
    }

    override fun updateDrawState(ds: TextPaint) {
        super.updateDrawState(ds)
        ds.color = Color.BLUE
        ds.isUnderlineText = true
    }
}, start, end, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
```

### 3.4 여러 단어에 다중 Span 적용 예제

```kotlin
private fun setSpannable(textView: TextView) {
  val text = "이용약관과 개인정보처리방침에 동의해주세요"
  val spannable = SpannableString(text)

  spanWord(text, "이용약관", spannable) {
    Toast.makeText(applicationContext, "이용약관 클릭됨", Toast.LENGTH_SHORT).show()
  }
  spanWord(text, "개인정보처리방침", spannable) {
    Toast.makeText(applicationContext, "개인정보 클릭됨", Toast.LENGTH_SHORT).show()
  }

  textView.text = spannable
  textView.movementMethod = LinkMovementMethod.getInstance()

}

private fun spanWord(
  text: String,
  word: String,
  spannable: Spannable,
  onClick: () -> Unit
) {
  val s = text.indexOf(word)
  val e = s + word.length
  spannable.setSpan(object : ClickableSpan() {
    override fun onClick(widget: View) {
      onClick()
    }

    override fun updateDrawState(ds: TextPaint) {
      ds.isUnderlineText = true
      ds.color = Color.BLUE
    }
  }, s, e, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
}
```

### 3.5 자동 링크 적용 (전화번호 이메일 등)

```kotlin
textView.autoLinkMask = Linkify.ALL
textView.text = "문의: 010-1234-5678, email@example.com"
textView.movementMethod = LinkMovementMethod.getInstance()
```
- 웹 링크 : Linkify.WEB_URLS
- 전화번호 : Linkify.PHONE_NUMBERS
- 이메일 : Linkify.EMAIL_ADDRESSES
- 모두 : Linkify.ALL

---

## 4.🧠 **결론**

- TSpannableString은 텍스트 내 특정 영역에만 스타일이나 동작을 적용할 수 있어, 하이라이트 표시, 링크 텍스트, 약관 동의 텍스트 등에서 매우 자주 사용됩니다.
  + 단어 강조: ForegroundColorSpan, StyleSpan, UnderlineSpan
  + 클릭 이벤트: ClickableSpan
  + 다중 적용: 위치 계산 후 반복 적용
  + movementMethod 설정 필수!
- 커스텀 UI 없이도 인터랙티브한 텍스트를 만들 수 있다는 점에서 실무 활용도가 매우 높습니다.