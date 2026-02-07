---
title: (Web) 모바일 웹에서 Android / iOS 앱 호출하는 방법 (미설치 시 스토어)
tags: [ Web ]
style: fill
color: dark
description: (Web) 모바일 웹에서 Android / iOS 앱 호출하는 방법 — 딥링크/스킴 + fallback(Play/App Store) 실무 패턴
---

## 개요

모바일 웹에서 “앱 열기” 버튼을 눌렀을 때 보통 목표는 이겁니다.
- 앱이 설치돼 있으면 앱을 바로 열고
- 설치돼 있지 않으면 스토어로 보내기
- Android/iOS는 동작 방식이 달라서, OS별 분기 + fallback 전략이 필요합니다.

요약
- `Android: intent://`(권장) 또는 스킴 호출 후 Play Store fallback
- iOS: 스킴 호출 + 타이머로 fallback (앱이 열리면 페이지가 백그라운드로 가므로 타이머 취소/무시)
- iOS는 “설치 여부를 100% 확정하는 API”가 없음 → 시간 기반 판단이 사실상 표준 
- Universal Links / App Links를 쓰면 가장 깔끔하지만, 설정 비용이 있음

---

## 1. OS 판별 함수(iPad포함)

```html
<script>
    function getMobileOS() {
      const ua = navigator.userAgent || "";
      const platform = navigator.platform || "";
    
      if (/android/i.test(ua)) return "android";
    
      // iPhone/iPad/iPod
      if (/iPhone|iPad|iPod/i.test(ua)) return "ios";
    
      // iPadOS 13+ (Mac처럼 보이는 UA)
      if (platform === "MacIntel" && navigator.maxTouchPoints > 1) return "ios";
    
      return "other";
    }
</script>
```

---

## 2. 준비: 앱/스토어 URL들

- `schemeUrl`: 예) `myapp://open?x=1`
- `androidIntentUrl`: 예) `intent://open?x=1#Intent;scheme=myapp;package=com.example.myapp;end`
- playStoreUrl: https://play.google.com/store/apps/details?id=com.example.myapp
- appStoreUrl: https://apps.apple.com/app/id123456789

> Android는 intent://를 쓰면 미설치 시 Play Store로 가는 UX가 더 안정적입니다.

---

## 3. 실무 권장: “앱 열기” 함수 (Android/iOS 분기 + fallback)

아래 코드는 가장 흔히 쓰는 패턴입니다.

```html
<script>
    function openAppOrStore() {
      const os = getMobileOS();
    
      // 프로젝트에 맞게 수정
      const schemeUrl = "myapp://open?from=web";
      const playStoreUrl = "https://play.google.com/store/apps/details?id=com.example.myapp";
      const appStoreUrl = "https://apps.apple.com/app/id123456789";
    
      // Android: intent:// 권장 (Chrome에서 잘 동작)
      const androidIntentUrl =
        "intent://open?from=web#Intent;scheme=myapp;package=com.example.myapp;end";
    
      if (os === "android") {
        // 1) intent로 시도
        window.location.href = androidIntentUrl;
    
        // 2) 일부 환경에서 intent가 막힐 수 있어 fallback 타이머(보험)
        setTimeout(() => {
          window.location.href = playStoreUrl;
        }, 1200);
    
        return;
      }
    
      if (os === "ios") {
        // iOS는 intent 스킴이 아니라 일반 scheme 호출 + 타이머 fallback
        const start = Date.now();
    
        // 앱 열기 시도
        window.location.href = schemeUrl;
    
        // 설치되어 앱이 열리면 브라우저가 백그라운드로 가며
        // 타이머가 실행되지 않거나, 돌아왔을 때 시간이 많이 지났을 가능성이 큼
        setTimeout(() => {
          const elapsed = Date.now() - start;
          // 너무 짧게 지나면(=앱이 안 열린 가능성) 스토어로 이동
          // (기기/브라우저에 따라 튜닝 필요: 800~1500ms)
          if (elapsed < 1600) {
            window.location.href = appStoreUrl;
          }
        }, 1200);
    
        return;
      }
    
      alert("모바일 기기에서 열어주세요.");
    }
</script>
```

### iOS fallback 판단이 “시간 기반”인 이유
- iOS Safari는 웹에서 “앱 설치 여부”를 확정적으로 알 수 있는 표준 API가 없습니다.
- 그래서 앱 호출 → 일정 시간 후에도 페이지가 그대로면 스토어로 이동이 현실적인 표준입니다.

---

## 4. Android `intent://` URL 만들기 템플릿

```text
intent://<HOST_PATH_AND_QUERY>#Intent;
scheme=<SCHEME>;
package=<PACKAGE_NAME>;
end

intent://open?from=web#Intent;scheme=myapp;package=com.example.myapp;end
```

- scheme=myapp → 앱 스킴
- package=com.example.myapp → Android 패키지명

> 추가 파라미터(예: S.browser_fallback_url=)로 fallback을 더 정교하게 만들 수도 있지만, 기본 형태만으로도 대부분 충분합니다.

---

## 5. 더 안정적으로 만들고 싶다면: Universal Links / App Links

가장 좋은 UX는 이겁니다.
- 링크 클릭 → 설치됨: 앱이 자동으로 열림
- 링크 클릭 → 미설치: 웹 페이지 유지 또는 스토어 안내

iOS: Universal Links
- 도메인 + apple-app-site-association 파일 필요
- 앱에 Associated Domains 설정 필요

Android: App Links
- 도메인 + assetlinks.json 필요
- 앱에 intent-filter + autoVerify 설정 필요

> 설정은 번거롭지만, “스킴 + 타이머” 방식보다 차단/경고가 안정적입니다.