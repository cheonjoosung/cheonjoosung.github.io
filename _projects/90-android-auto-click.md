---
name: AutoClick - 반복 탭 입력 보조
tools: [Kotlin, Compose, Hilt, Room]
image: /auto_click/play_feature_graphic_1024x500.png
description: 반복해서 누르기 힘든 분을 위해 지정한 위치를 대신 탭해주는 앱
---

## AutoClick
- 같은 자리를 반복해서 누르기 어려운 사용자를 위한 입력 보조 앱
- 사용자가 찍은 좌표만 `dispatchGesture`로 누르고, 화면이나 다른 앱 텍스트는 읽지 않음

## Period
- 1인 프로젝트
- 2026.08.01 ~ 2026.08.10

## Dev.
**개발 환경**
- Android Studio, Kotlin 2.2.10, AGP 9.2.1
- minSdk 26 / targetSdk 36, Jetpack Compose, Material 3
- Hilt, Room, DataStore, Navigation Compose, Coroutines
- Firebase Crashlytics(release만), AdMob

**아키텍처**
- Clean Architecture + MVVM, 모듈 4개 (`:app` / `:domain` / `:data` / `:core-ui`)
- `:domain`은 순수 Kotlin. Android SDK를 안 넣어서 잘못된 import는 컴파일에서 막히게 함
- 엔진 상태: Idle → Ready → Running / Paused → Finished | Failed

**모듈 · 흐름**
![구조도](/auto_click/architecture.svg)

**구현된 기능**
- 오버레이 제어바로 마커 추가/삭제, 재생/일시정지/정지, 스크립트 저장
- 스크립트 목록에서 바로 실행. 세로/가로 좌표는 따로 저장
- 탭 간격, 유지 시간, 누르는 시간 조절 (`durationMs`로 탭/롱프레스 구분)
- 권한 안내, 문제 해결, UI 크기 조절
- 테마(시스템/라이트/다크), 앱 언어 한/영, 가로 폰·태블릿 레이아웃
- 정지는 홈, 제어바, 알림, 볼륨↓ 길게 네 군데
- 출시 준비: minify, Crashlytics 매핑, AdMob, 접근성 도구 선언

**개발하면서 정한 것**
- 클릭은 `AccessibilityService.dispatchGesture`. 루팅 없이 되는 쪽을 택함
- 다른 앱 위에 떠야 해서 Overlay + ComposeView. Activity가 아니라서 Lifecycle이랑 ViewModelStore를 직접 붙였음
- 실행 유지는 Foreground Service `specialUse` + 알림
- 재생 중에 제어바가 자기 자신을 누르는 문제가 있어서 `FLAG_NOT_TOUCHABLE`을 넣음
- Play 심사용으로 접근성 도구(`isAccessibilityTool`) + 장애 유형은 Motor만. 스토어 설명이랑 안 맞으면 거절됨
- 1회/다중 모드를 안 두고 좌표 개수로만 동작. 하나면 그 점만, 여러 개면 순서대로

**스크린샷**
![preview](/auto_click/play_feature_graphic_1024x500.png)


## Result & Learned
- 이 앱은 화면보다 접근성 서비스랑 오버레이가 본체라서, 권한/정책/제조사 절전 이슈를 생각보다 많이 봄
- 스와이프랑 공유를 넣었다가 뺐음. 모드랑 제스처 타입을 두면 좌표 개수랑 값이 따로 놀길래 `durationMs`랑 리스트만 남김. 공유 롤백할 때 Room 마이그레이션(1→2)을 실제로 겪음
- `setApplicationLocales`는 `AppCompatActivity`가 아니면 안 먹힘. 가로 폰은 폭이 넓은데 높이가 낮아서, WindowSizeClass를 폭 기준으로 보면 태블릿으로 잘못 잡힘

**Clean Architecture / DI**
- `:domain`을 java-library로 빼니까 Android import가 컴파일 에러가 남. 규칙을 문서로만 두지 않아도 됨
- domain은 Hilt를 안 넣고 `javax.inject`의 `@Inject`만 씀. ViewModel은 `AutoClickEngine`, `ScriptRepository` 인터페이스만 받고, 구현체는 `@Binds`로 app/data에 붙임. Room DB처럼 직접 만들어야 하는 것만 `@Provides`
- AccessibilityService랑 Foreground Service는 시스템이 띄워서 `@AndroidEntryPoint`가 필요했음. 그래서 Hilt를 씀. 같은 프로세스라 엔진을 싱글턴으로 두고 UI/오버레이/서비스가 StateFlow로 같이 봄
- `CrashReporter`는 debug에선 NoOp, release에선 Firebase로 소스셋만 바꿈. 재생 루프는 화면보다 오래 가서 `@ApplicationScope`에 붙였고, Compose에서 ViewModel 없이 쓰려면 EntryPoint가 필요했음
