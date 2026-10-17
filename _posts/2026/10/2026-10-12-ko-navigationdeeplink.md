---
title: (Android/Navigation) Navigation Component — Deep Link 처리
tags: [ Android, Navigation ]
style: fill
color: dark
description: Navigation Component의 Explicit/Implicit Deep Link 설정과 URI 인자 매핑, PendingIntent 알림 연동, 테스트 방법을 정리합니다.
---

---

## 1. Deep Link란?

앱의 **특정 화면으로 바로 진입**하게 해주는 링크입니다.
알림 클릭, 웹 링크, 다른 앱에서의 호출 등에서 사용합니다.

| 종류 | 설명 | 예시 |
|------|------|------|
| Explicit | 앱 내부에서 PendingIntent로 특정 화면 지정 | 알림 클릭 |
| Implicit | URI로 외부에서 진입 | `https://example.com/product/42` |

---

## 2. Implicit Deep Link — 그래프에 선언

```xml
<fragment
    android:id="@+id/detailFragment"
    android:name="com.example.app.DetailFragment">

    <argument android:name="productId" app:argType="integer" />

    <!-- {productId}가 위 argument로 자동 매핑 -->
    <deepLink app:uri="https://example.com/product/{productId}" />
    <deepLink app:uri="myapp://product/{productId}" />
</fragment>
```

Manifest에서 해당 Activity가 NavGraph를 사용한다고 알려줍니다.

```xml
<activity android:name=".MainActivity" android:exported="true">
    <nav-graph android:value="@navigation/nav_graph" />
</activity>
```

> `<nav-graph>`가 빌드 시 위 deepLink들을 **intent-filter로 자동 확장**합니다. 직접 작성하지 않아도 됩니다.

---

## 3. 인자 매핑 규칙

```
URI: https://example.com/product/42?source=push

{productId} → path 인자 (integer)
?source=    → 쿼리 파라미터는 같은 이름의 argument(선택)로 매핑
```

```kotlin
class DetailFragment : Fragment() {
    private val args: DetailFragmentArgs by navArgs()
    // 딥링크로 들어와도 일반 이동과 동일하게 args.productId 사용
}
```

---

## 4. Explicit Deep Link — 알림에서 화면으로

```kotlin
val pendingIntent = NavDeepLinkBuilder(context)
    .setGraph(R.navigation.nav_graph)
    .setDestination(R.id.detailFragment)
    .setArguments(bundleOf("productId" to 42))
    .createPendingIntent()

val notification = NotificationCompat.Builder(context, CHANNEL_ID)
    .setContentTitle("할인 상품")
    .setContentIntent(pendingIntent)   // 클릭 시 상세 화면으로
    .setAutoCancel(true)
    .build()
```

`NavDeepLinkBuilder`는 시작 화면부터 대상까지의 **백스택을 합성**해, 뒤로 가기 시 홈을 거쳐 앱이 자연스럽게 동작합니다.

---

## 5. 앱 링크(App Links) 검증

`https` 링크가 브라우저 선택 창 없이 바로 앱으로 열리게 하려면 도메인 검증이 필요합니다.

```xml
<!-- 그래프의 deepLink에 autoVerify를 위해 Manifest에서 설정 -->
<nav-graph android:value="@navigation/nav_graph" />
```

```
1. https://example.com/.well-known/assetlinks.json 에 앱 패키지와 서명 SHA256 등록
2. deepLink 의 scheme을 https, host를 해당 도메인으로 선언
```

---

## 6. 테스트

```bash
# 에뮬레이터/기기에서 딥링크 실행
adb shell am start -a android.intent.action.VIEW \
  -d "myapp://product/42" com.example.app
```

```kotlin
// Navigation 테스트
val navController = TestNavHostController(context)
navController.setGraph(R.navigation.nav_graph)
navController.navigate(Uri.parse("myapp://product/42"))
assertEquals(R.id.detailFragment, navController.currentDestination?.id)
```

---

## 7. 주의사항

- 외부에서 오는 URI는 **신뢰하지 말고 인자를 검증**하세요 (존재하지 않는 ID, 권한 없는 화면)
- 로그인이 필요한 화면은 딥링크 진입 후 인증 상태를 확인해 로그인 화면으로 보낸 뒤 복귀시킵니다
- `exported="true"`인 Activity는 외부에서 호출되므로 민감한 동작을 노출하지 마세요

---

## 8. 정리

- 그래프의 `<deepLink app:uri="...{arg}" />`로 URI와 인자를 매핑
- Manifest의 `<nav-graph>`가 intent-filter를 자동 생성
- 알림 등 앱 내부 진입은 `NavDeepLinkBuilder`로 PendingIntent 생성 (백스택 자동 구성)
- `adb shell am start`로 간단히 테스트
- 외부 입력이므로 인자 검증과 인증 처리가 필수
