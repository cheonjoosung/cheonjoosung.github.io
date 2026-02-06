---
title: (Web) 모바일 웹에서 Android / iOS 구분하는 방법
tags: [ Web ]
style: fill
color: dark
description: (Web) 모바일 웹에서 Android / iOS 구분하는 방법 - UserAgent부터 실무 패턴까지 정리
---

## 개요

모바일 웹을 개발하다 보면 다음과 같은 요구가 자주 생깁니다.
- Android / iOS에 따라 스토어 링크 분기
- 앱 설치 유도 배너 분기
- 앱 스킴 호출 시 OS별 분기
- WebView ↔ Native 브릿지 처리
- 이때 가장 기본적이면서도 널리 쓰이는 방법이 UserAgent 기반 OS 판별입니다.

요약
- 가장 보편적인 방법: navigator.userAgent
- Android: Android 문자열 포함
- iOS: iPhone, iPad, iPod 
- iPadOS(13+)는 Mac처럼 보이는 UA 주의
- 서버/클라이언트 모두에서 처리 가능

---

## 1. 가장 기본적인 방법: UserAgent로 판별 (클라이언트)

```html
function getMobileOS() {
  const ua = navigator.userAgent || navigator.vendor || window.opera;

  if (/android/i.test(ua)) {
    return "android";
  }

  if (/iPad|iPhone|iPod/.test(ua)) {
    return "ios";
  }

  return "other";
}

// 사용 방법
const os = getMobileOS();

if (os === "android") {
    // Android 처리
} else if (os === "ios") {
    // iOS 처리
}
```

---

## 2. Android / iOS UserAgent 예시

Android UserAgent 예시

```text
Mozilla/5.0 (Linux; Android 14; Pixel 7)
AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/120.0.0.0 Mobile Safari/537.36
```

iOS UserAgent 예시

```text
Mozilla/5.0 (iPhone; CPU iPhone OS 17_2 like Mac OS X)
AppleWebKit/605.1.15 (KHTML, like Gecko)
Version/17.2 Mobile/15E148 Safari/604.1
```

---

## 3. ⚠️ iPadOS 13+ 주의사항 (중요)

iPadOS 13부터 Safari 기본 UA가 Mac과 거의 동일해졌습니다.

```text
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15)
AppleWebKit/605.1.15 (KHTML, like Gecko)
Version/17.0 Mobile/15E148 Safari/604.1
```

iPadOS까지 고려한 판별 코드

```text
function isIOS() {
  const ua = navigator.userAgent;
  const platform = navigator.platform;

  // iPhone / iPad / iPod
  if (/iPhone|iPad|iPod/.test(ua)) return true;

  // iPadOS 13+ (Mac처럼 보이는 경우)
  if (platform === "MacIntel" && navigator.maxTouchPoints > 1) {
    return true;
  }

  return false;
}
```