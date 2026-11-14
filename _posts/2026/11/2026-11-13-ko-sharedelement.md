---
title: (Android/UI) Shared Element Transition — 자연스러운 화면 이동
tags: [ Android, Animation ]
style: fill
color: dark
description: Shared Element Transition으로 목록에서 상세 화면으로 이어지는 자연스러운 이동을 Activity, Fragment, Navigation, Compose에서 구현하는 방법을 정리합니다.
---

---

## 1. Shared Element Transition이란?

두 화면에 **공통으로 존재하는 요소**(썸네일 이미지, 제목 등)가 한 화면에서 다른 화면으로 **이어져 이동하는** 것처럼 보이게 하는 전환입니다.

```
목록 화면                         상세 화면
┌────────────┐                    ┌────────────────────┐
│ [썸네일] 제목│  ── 탭 ──▶       │  [   큰 이미지   ]  │
└────────────┘     썸네일이       │  제목              │
                   자연스럽게 확대  └────────────────────┘
```

사용자가 "어디서 왔는지" 맥락을 유지해 주는 효과가 있습니다.

---

## 2. 핵심 개념 — transitionName

두 화면의 공유 요소는 **같은 `transitionName`** 으로 짝지어집니다.

```xml
<!-- 목록 아이템 -->
<ImageView
    android:id="@+id/thumbnail"
    android:transitionName="@string/transition_thumbnail" />   <!-- 고정 이름 -->

<!-- 상세 화면 -->
<ImageView
    android:id="@+id/heroImage"
    android:transitionName="@string/transition_thumbnail" />   <!-- 같은 이름 -->
```

목록은 **아이템마다 이름이 고유**해야 하므로 코드에서 동적으로 지정합니다.

```kotlin
// 어댑터 onBindViewHolder
ViewCompat.setTransitionName(binding.thumbnail, "thumb_${item.id}")
```

---

## 3. Fragment + Navigation Component

```kotlin
// 목록 Fragment — 클릭 시 공유 요소 전달
adapter.onItemClick = { item, sharedView ->
    val extras = FragmentNavigatorExtras(sharedView to "thumb_${item.id}")
    findNavController().navigate(
        ListFragmentDirections.actionListToDetail(item.id),
        extras
    )
}
```

```kotlin
// 상세 Fragment
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    sharedElementEnterTransition = MaterialContainerTransform().apply {
        duration = 350
        scrimColor = Color.TRANSPARENT
    }
}

override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    // 목록과 같은 transitionName 지정
    ViewCompat.setTransitionName(binding.heroImage, "thumb_${args.itemId}")

    // 이미지 로딩이 끝날 때까지 전환 시작을 지연 (중요)
    postponeEnterTransition()
    Glide.with(this).load(url)
        .listener(object : RequestListener<Drawable> {
            override fun onResourceReady(...): Boolean { startPostponedEnterTransition(); return false }
            override fun onLoadFailed(...): Boolean { startPostponedEnterTransition(); return false }
        })
        .into(binding.heroImage)
}
```

> **`postponeEnterTransition()`** 을 빠뜨리면 이미지가 아직 로드되지 않은 상태에서 전환이 시작돼 **끊기거나 번쩍이는** 현상이 생깁니다. 로딩 성공/실패 양쪽에서 `startPostponedEnterTransition()`을 호출해야 합니다.

---

## 4. 목록으로 돌아올 때 (Return Transition)

뒤로 가기 시 요소가 원래 자리로 돌아가려면 목록 쪽에서도 준비가 필요합니다.

```kotlin
// 목록 Fragment onViewCreated
postponeEnterTransition()
binding.recyclerView.doOnPreDraw { startPostponedEnterTransition() }   // 리스트가 그려진 후 시작
```

`RecyclerView`의 아이템이 그려지기 전에 전환이 시작되면 돌아갈 위치를 찾지 못합니다.

---

## 5. Activity 전환

```kotlin
val options = ActivityOptions.makeSceneTransitionAnimation(
    this,
    Pair.create(thumbnail, "thumb"),                       // (뷰, transitionName)
    Pair.create(titleView, "title")
)
startActivity(Intent(this, DetailActivity::class.java), options.toBundle())
```

```kotlin
// 상세 Activity
window.sharedElementEnterTransition = MaterialContainerTransform()
```

---

## 6. MaterialContainerTransform — 컨테이너 변형

카드가 **상세 화면 전체로 확장**되는 Material 전환입니다. 이미지뿐 아니라 컨테이너(카드)를 통째로 이어줍니다.

```kotlin
// 시작: 목록 카드, 끝: 상세 루트 뷰
sharedElementEnterTransition = MaterialContainerTransform().apply {
    drawingViewId = R.id.nav_host_fragment          // 전환이 그려질 컨테이너
    scrimColor = Color.TRANSPARENT
    setAllContainerColors(MaterialColors.getColor(requireView(), R.attr.colorSurface))
}
```

---

## 7. Jetpack Compose (1.7+)

Compose에서는 `SharedTransitionLayout`과 `sharedElement` Modifier를 사용합니다.

```kotlin
@Composable
fun App() {
    SharedTransitionLayout {
        val nav = rememberNavController()
        NavHost(nav, startDestination = "list") {
            composable("list") {
                ListScreen(
                    onClick = { nav.navigate("detail/$it") },
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable
                )
            }
            composable("detail/{id}") { entry ->
                DetailScreen(
                    id = entry.arguments?.getString("id")!!,
                    sharedTransitionScope = this@SharedTransitionLayout,
                    animatedVisibilityScope = this@composable
                )
            }
        }
    }
}

// 양쪽 화면의 이미지에 같은 key로 지정
with(sharedTransitionScope) {
    Image(
        painter = painter,
        contentDescription = null,
        modifier = Modifier.sharedElement(
            rememberSharedContentState(key = "image-$id"),    // 같은 key가 짝
            animatedVisibilityScope = animatedVisibilityScope
        )
    )
}
```

---

## 8. 자주 겪는 문제

| 증상 | 원인/해결 |
|------|----------|
| 전환이 번쩍이며 안 이어짐 | 이미지 로딩 전에 시작 → `postponeEnterTransition` |
| 뒤로 갈 때 사라짐 | 목록이 그려지기 전에 시작 → `doOnPreDraw` |
| 엉뚱한 아이템이 움직임 | transitionName이 중복 → 아이템별 **고유 이름** |
| 상태바/툴바가 깜빡임 | `excludeTarget(android.R.id.statusBarBackground, true)` |
| 다른 화면 요소가 겹침 | 전환 중에는 `elevation`, `clipChildren` 확인 |

---

## 9. 정리

- 두 화면의 요소를 **같은 `transitionName`** (Compose는 같은 key)으로 연결
- Fragment는 `FragmentNavigatorExtras`, Activity는 `makeSceneTransitionAnimation`
- 이미지 로딩은 `postponeEnterTransition`/`startPostponedEnterTransition`으로 **시작 시점 지연**
- 돌아올 때를 위해 목록도 그려진 후 전환 시작
- 카드 확장 형태는 `MaterialContainerTransform`, Compose 1.7+는 `SharedTransitionLayout`
