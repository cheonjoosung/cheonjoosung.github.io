---
title: (Android/Navigation) Navigation Component 기초 — NavGraph, NavController
tags: [ Android, Navigation ]
style: fill
color: dark
description: Jetpack Navigation Component의 NavHost, NavGraph, NavController 구성과 Fragment 간 화면 이동 방법을 정리합니다.
---

---

## 1. Navigation Component란?

앱 내 화면(Destination) 이동을 **하나의 그래프로 선언적으로 관리**하는 Jetpack 라이브러리입니다.
`FragmentTransaction`을 직접 다루는 대신, 이동 경로를 XML 그래프에 정의하고 `NavController`로 이동합니다.

```
NavHost (화면이 표시되는 컨테이너)
   ↑ 보유
NavController (이동 명령 담당)
   ↑ 참조
NavGraph (화면과 이동 경로 정의: res/navigation/nav_graph.xml)
```

**의존성**:
```kotlin
implementation("androidx.navigation:navigation-fragment-ktx:2.8.0")
implementation("androidx.navigation:navigation-ui-ktx:2.8.0")
```

---

## 2. NavGraph 정의

`res/navigation/nav_graph.xml`에 화면과 이동 경로(action)를 선언합니다.

```xml
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">  <!-- 앱 시작 화면 -->

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.app.HomeFragment"
        android:label="홈">
        <!-- action: 홈 → 상세 이동 경로 -->
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.app.DetailFragment"
        android:label="상세" />
</navigation>
```

---

## 3. NavHostFragment 배치

Activity 레이아웃에 화면이 표시될 컨테이너를 둡니다.

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:defaultNavHost="true"
    app:navGraph="@navigation/nav_graph" />
```

> `defaultNavHost="true"`: 시스템 뒤로 가기 버튼을 이 NavHost가 가로채 처리합니다.

---

## 4. NavController로 이동

```kotlin
class HomeFragment : Fragment(R.layout.fragment_home) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // findNavController(): 이 Fragment가 속한 NavHost의 컨트롤러 획득
        val navController = findNavController()

        binding.detailButton.setOnClickListener {
            // action ID로 이동 (그래프에 정의된 경로만 이동 가능)
            navController.navigate(R.id.action_home_to_detail)
        }
    }
}

// 뒤로 가기
findNavController().popBackStack()   // 스택에서 현재 화면 제거
findNavController().navigateUp()     // Up 버튼 동작 (계층 구조상 상위로)
```

---

## 5. 툴바와 연동 — AppBarConfiguration

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var appBarConfiguration: AppBarConfiguration

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val navHost = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        val navController = navHost.navController

        // 최상위 화면(뒤로 가기 화살표 대신 햄버거/없음 표시)
        appBarConfiguration = AppBarConfiguration(setOf(R.id.homeFragment))

        // 그래프의 label을 툴바 제목으로 자동 표시 + Up 버튼 자동 처리
        setupActionBarWithNavController(navController, appBarConfiguration)
    }

    override fun onSupportNavigateUp(): Boolean {
        val navController = findNavController(R.id.nav_host_fragment)
        return navController.navigateUp(appBarConfiguration) || super.onSupportNavigateUp()
    }
}
```

---

## 6. 정리

| 구성 요소 | 역할 |
|----------|------|
| `NavGraph` | 화면과 이동 경로를 정의한 XML |
| `NavHost` | 현재 화면을 표시하는 컨테이너 |
| `NavController` | `navigate()`, `popBackStack()` 등 이동 수행 |
| `action` | 화면 간 이동 경로 + 인자, 애니메이션 설정 |

- 이동 경로를 그래프에 모아 **화면 흐름을 한눈에 파악**
- `FragmentTransaction`, 백스택 처리를 라이브러리가 대신 관리
- Compose를 쓴다면 같은 개념을 Compose Navigation으로 사용
