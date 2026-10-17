---
title: (Android/Navigation) Navigation Component — Bottom Navigation 연동
tags: [ Android, Navigation ]
style: fill
color: dark
description: BottomNavigationView와 NavController를 연동하는 방법과 메뉴 ID 규칙, 탭별 백스택 보존, 하위 화면에서 바 숨기기를 정리합니다.
---

---

## 1. 구조

바텀 내비게이션의 각 탭을 NavGraph의 **최상위 Destination**으로 연결합니다.

```
BottomNavigationView
   ├─ 홈   → homeFragment
   ├─ 검색 → searchFragment
   └─ 설정 → settingsFragment
        (NavController가 선택에 맞춰 화면 전환)
```

---

## 2. 메뉴와 그래프 연결 규칙

**메뉴 아이템의 ID를 그래프 Destination ID와 동일하게** 맞추면 자동 연결됩니다.

```xml
<!-- res/menu/bottom_nav_menu.xml -->
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@+id/homeFragment"
        android:icon="@drawable/ic_home" android:title="홈" />
    <item android:id="@+id/searchFragment"
        android:icon="@drawable/ic_search" android:title="검색" />
    <item android:id="@+id/settingsFragment"
        android:icon="@drawable/ic_settings" android:title="설정" />
</menu>
```

```xml
<!-- nav_graph.xml: 위 메뉴 ID와 같은 id의 fragment 선언 -->
<fragment android:id="@+id/homeFragment" ... />
<fragment android:id="@+id/searchFragment" ... />
<fragment android:id="@+id/settingsFragment" ... />
```

---

## 3. 코드 연동

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val navHost = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        val navController = navHost.navController

        // 한 줄 연결: 탭 선택 시 이동 + 화면 이동 시 탭 선택 상태 동기화
        binding.bottomNav.setupWithNavController(navController)

        // 탭 화면들은 모두 최상위로 취급 (뒤로 가기 화살표 숨김)
        val appBarConfiguration = AppBarConfiguration(
            setOf(R.id.homeFragment, R.id.searchFragment, R.id.settingsFragment)
        )
        setupActionBarWithNavController(navController, appBarConfiguration)
    }
}
```

---

## 4. 탭별 백스택 보존

`setupWithNavController`는 기본적으로 탭 전환 시 **각 탭의 상태를 저장/복원**합니다 (`saveState`/`restoreState`).

```
홈 탭 → 상세 화면 이동 → 검색 탭 → 다시 홈 탭
   ↓
홈 탭의 상세 화면이 그대로 복원됨
```

직접 이동을 구현한다면 동일한 옵션을 줘야 합니다.

```kotlin
navController.navigate(R.id.searchFragment, null, navOptions {
    popUpTo(navController.graph.startDestinationId) { saveState = true }
    launchSingleTop = true
    restoreState = true
})
```

> 탭 재선택(이미 선택된 탭을 다시 누름) 동작은 `setOnItemReselectedListener`로 별도 처리합니다 (예: 맨 위로 스크롤).

---

## 5. 하위 화면에서 바텀바 숨기기

상세 화면 등에서는 바텀바가 보이지 않는 것이 자연스럽습니다.

```kotlin
val topLevelDestinations = setOf(
    R.id.homeFragment, R.id.searchFragment, R.id.settingsFragment
)

navController.addOnDestinationChangedListener { _, destination, _ ->
    // 최상위 탭 화면에서만 바텀바 표시
    binding.bottomNav.isVisible = destination.id in topLevelDestinations
}
```

---

## 6. 정리

- 메뉴 아이템 ID = 그래프 Destination ID 로 맞추면 `setupWithNavController` 한 줄로 연동
- 탭 전환 시 각 탭의 백스택/상태가 **자동 보존**
- 탭 화면들을 `AppBarConfiguration`의 최상위로 등록
- 하위 화면에서 바를 숨길 때는 `addOnDestinationChangedListener` 사용
- 탭이 여러 개이고 각 탭이 독립된 흐름이면 이 구조가 가장 단순
