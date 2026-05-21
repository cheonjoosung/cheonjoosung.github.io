---
title: (Kotlin/코틀린) 컴포지트 패턴(Composite Pattern) 완전 정리
tags: [ Kotlin, Android ]
style: fill
color: dark
description: 컴포지트 패턴의 개념, 트리 구조에서 단일·복합 객체를 동일하게 처리하는 방법, Android View 계층, 파일 시스템 예제까지 깊이 있게 정리합니다.
---

## 개요

- 구조 패턴(Structural Pattern) 중 **컴포지트 패턴(Composite Pattern)** 을 다룹니다.
- 컴포지트 패턴은 **객체들을 트리 구조로 구성해 단일 객체와 복합 객체를 동일하게 처리** 할 수 있게 하는 패턴입니다.
- 이 글에서는 다음을 설명합니다.
  - 컴포지트 패턴이 필요한 이유
  - Component / Leaf / Composite 구조
  - Kotlin 실전 구현
  - Android 실전 예제 (View 계층, 메뉴, 권한 그룹)

---

## 1. 왜 컴포지트 패턴이 필요한가

### ❌ 단일 파일과 폴더를 다르게 처리

```kotlin
// 파일 크기 계산 — 파일인지 폴더인지 매번 타입 확인 필요
fun calculateSize(item: Any): Long {
    return when (item) {
        is File   -> item.size                                     // 단일 파일
        is Folder -> item.children.sumOf { calculateSize(it) }    // 재귀 — 타입 분기 반복
        else      -> throw IllegalArgumentException("알 수 없는 타입")
    }
}
// 새 타입(링크, 압축파일 등)이 추가될 때마다 when 블록 수정 필요
```

---

### ✅ 컴포지트로 해결 — 동일 인터페이스로 처리

```kotlin
// 파일이든 폴더든 같은 방식으로 처리
fun calculateSize(component: FileSystemComponent): Long = component.size()

val root = Directory("root").apply {
    add(FileItem("readme.txt", 1024))
    add(Directory("src").apply {
        add(FileItem("Main.kt", 2048))
        add(FileItem("Utils.kt", 512))
    })
}

println(calculateSize(root))  // 3584 — 타입 분기 없이 동일 처리 ✅
```

---

## 2. 핵심 구조

```
Component (공통 인터페이스)
    ├── Leaf (단일 객체 — 자식 없음)
    └── Composite (복합 객체 — 자식 보유, 자식도 Component)
```

---

## 3. 기본 구현 — 파일 시스템

```kotlin
// Component — 공통 인터페이스
interface FileSystemComponent {
    val name: String
    fun size(): Long
    fun print(indent: String = "")
}

// Leaf — 단일 파일 (자식 없음)
class FileItem(
    override val name: String,
    private val fileSize: Long
) : FileSystemComponent {
    override fun size() = fileSize
    override fun print(indent: String) = println("${indent}📄 $name (${fileSize}B)")
}

// Composite — 폴더 (자식 보유)
class Directory(
    override val name: String
) : FileSystemComponent {
    private val children = mutableListOf<FileSystemComponent>()

    fun add(component: FileSystemComponent)    = children.add(component)
    fun remove(component: FileSystemComponent) = children.remove(component)

    override fun size()  = children.sumOf { it.size() }

    override fun print(indent: String) {
        println("${indent}📁 $name (${size()}B)")
        children.forEach { it.print("$indent  ") }  // 재귀 — 타입 분기 없음 ✅
    }
}
```

```kotlin
// 사용
val root = Directory("project").apply {
    add(FileItem("build.gradle", 256))
    add(FileItem("README.md", 1024))
    add(Directory("src").apply {
        add(Directory("main").apply {
            add(FileItem("MainActivity.kt", 2048))
            add(FileItem("MainViewModel.kt", 1536))
        })
        add(Directory("test").apply {
            add(FileItem("MainViewModelTest.kt", 512))
        })
    })
}

root.print()
// 📁 project (5376B)
//   📄 build.gradle (256B)
//   📄 README.md (1024B)
//   📁 src (4096B)
//     📁 main (3584B)
//       📄 MainActivity.kt (2048B)
//       📄 MainViewModel.kt (1536B)
//     📁 test (512B)
//       📄 MainViewModelTest.kt (512B)

println(root.size())  // 5376 — 전체 크기를 재귀로 계산
```

---

## 4. sealed class로 구현 — Kotlin 스타일

```kotlin
sealed class MenuItem {
    abstract val title: String
    abstract fun render(depth: Int = 0)

    // Leaf — 단일 메뉴 항목
    data class Item(
        override val title: String,
        val action: () -> Unit,
        val isEnabled: Boolean = true
    ) : MenuItem() {
        override fun render(depth: Int) {
            val indent = "  ".repeat(depth)
            val state  = if (isEnabled) "" else " (비활성)"
            println("$indent- $title$state")
        }
    }

    // Composite — 서브메뉴를 가진 그룹
    class Group(
        override val title: String,
        private val items: MutableList<MenuItem> = mutableListOf()
    ) : MenuItem() {
        fun add(item: MenuItem) = items.add(item)

        override fun render(depth: Int) {
            println("${"  ".repeat(depth)}▶ $title")
            items.forEach { it.render(depth + 1) }
        }

        fun findItem(title: String): MenuItem? {
            return items.firstOrNull { it.title == title }
                ?: items.filterIsInstance<Group>()
                    .firstNotNullOfOrNull { it.findItem(title) }
        }
    }
}
```

```kotlin
// 앱 메뉴 구성
val menu = MenuItem.Group("메인 메뉴").apply {
    add(MenuItem.Item("홈") { navigateTo("home") })
    add(MenuItem.Group("설정").apply {
        add(MenuItem.Item("프로필 설정") { navigateTo("profile") })
        add(MenuItem.Group("알림 설정").apply {
            add(MenuItem.Item("푸시 알림") { togglePush() })
            add(MenuItem.Item("이메일 알림") { toggleEmail() })
        })
        add(MenuItem.Item("테마 설정") { navigateTo("theme") })
    })
    add(MenuItem.Item("고객센터") { navigateTo("support") })
    add(MenuItem.Item("로그아웃", isEnabled = isLoggedIn) { logout() })
}

menu.render()
// ▶ 메인 메뉴
//   - 홈
//   ▶ 설정
//     - 프로필 설정
//     ▶ 알림 설정
//       - 푸시 알림
//       - 이메일 알림
//     - 테마 설정
//   - 고객센터
//   - 로그아웃
```

---

## 5. Android 실전 예제 ① — 권한 그룹

```kotlin
sealed class Permission {
    abstract val name: String
    abstract fun isGranted(context: Context): Boolean

    // Leaf — 단일 권한
    data class Single(
        override val name: String,
        val manifestPermission: String
    ) : Permission() {
        override fun isGranted(context: Context) =
            ContextCompat.checkSelfPermission(context, manifestPermission) ==
                    PackageManager.PERMISSION_GRANTED
    }

    // Composite — 권한 그룹 (모두 허용돼야 granted)
    class Group(
        override val name: String,
        private val permissions: List<Permission>
    ) : Permission() {
        override fun isGranted(context: Context) =
            permissions.all { it.isGranted(context) }

        fun getAllManifestPermissions(): List<String> =
            permissions.flatMap { permission ->
                when (permission) {
                    is Single -> listOf(permission.manifestPermission)
                    is Group  -> permission.getAllManifestPermissions()
                }
            }
    }
}
```

```kotlin
// 권한 트리 구성
val cameraPermissions = Permission.Group(
    name        = "카메라 관련",
    permissions = listOf(
        Permission.Single("카메라",       Manifest.permission.CAMERA),
        Permission.Single("오디오",       Manifest.permission.RECORD_AUDIO),
        Permission.Single("외부 저장소",  Manifest.permission.WRITE_EXTERNAL_STORAGE)
    )
)

val locationPermissions = Permission.Group(
    name        = "위치 관련",
    permissions = listOf(
        Permission.Single("정확한 위치", Manifest.permission.ACCESS_FINE_LOCATION),
        Permission.Single("대략적 위치", Manifest.permission.ACCESS_COARSE_LOCATION)
    )
)

val appPermissions = Permission.Group(
    name        = "앱 전체 권한",
    permissions = listOf(cameraPermissions, locationPermissions)
)

// 단일 인터페이스로 그룹/단일 권한 동일하게 처리
if (!appPermissions.isGranted(this)) {
    requestPermissions(
        appPermissions.getAllManifestPermissions().toTypedArray(),
        REQUEST_CODE
    )
}
```

---

## 6. Android 실전 예제 ② — UI 컴포넌트 가격 계산

```kotlin
// 주문서 항목 — 단품과 세트 메뉴를 동일하게 처리
interface OrderItem {
    val name: String
    fun totalPrice(): Int
    fun describe(indent: String = ""): String
}

data class SingleItem(
    override val name: String,
    val price: Int,
    val quantity: Int = 1
) : OrderItem {
    override fun totalPrice() = price * quantity
    override fun describe(indent: String) = "${indent}$name x$quantity = ${totalPrice()}원"
}

class ComboItem(
    override val name: String,
    private val items: List<OrderItem>,
    private val discountRate: Double = 0.0
) : OrderItem {
    override fun totalPrice(): Int {
        val subtotal = items.sumOf { it.totalPrice() }
        return (subtotal * (1 - discountRate)).toInt()
    }

    override fun describe(indent: String): String {
        val sb = StringBuilder("${indent}[콤보] $name — ${totalPrice()}원\n")
        items.forEach { sb.append(it.describe("$indent  ") + "\n") }
        return sb.toString().trimEnd()
    }
}
```

```kotlin
// 주문 구성
val order = listOf<OrderItem>(
    SingleItem("불고기 버거", 5500),
    ComboItem(
        name         = "패밀리 세트",
        discountRate = 0.1,
        items = listOf(
            SingleItem("치킨 버거", 5000, quantity = 2),
            SingleItem("감자튀김", 2000, quantity = 2),
            SingleItem("콜라",    1500, quantity = 2)
        )
    ),
    SingleItem("아이스크림", 1000)
)

order.forEach { println(it.describe()) }
println("총 합계: ${order.sumOf { it.totalPrice() }}원")

// [출력]
// 불고기 버거 x1 = 5500원
// [콤보] 패밀리 세트 — 15300원
//   치킨 버거 x2 = 10000원
//   감자튀김 x2 = 4000원
//   콜라 x2 = 3000원
// 아이스크림 x1 = 1000원
// 총 합계: 21800원
```

---

## 7. 컴포지트 패턴 적용 판단 기준

```
✔ 데이터 구조가 트리 형태인가?
   → 파일 시스템, 카테고리, 조직도, 메뉴

✔ 단일 객체와 복합 객체를 동일하게 처리하고 싶은가?
   → 타입 체크(is, instanceof) 없이 처리하고 싶을 때

✔ 재귀적으로 작업을 수행해야 하는가?
   → 전체 크기 계산, 전체 출력, 전체 활성화 여부 등
```

---

## 8. 정리

| 항목 | 내용 |
|------|------|
| 목적 | 트리 구조에서 단일·복합 객체를 동일하게 처리 |
| 구성 | Component(인터페이스) / Leaf(단말) / Composite(중간 노드) |
| 핵심 효과 | 타입 분기 없이 재귀 처리 가능 |
| Kotlin 도구 | `sealed class`로 타입 안전하게 구현 |
| Android 사례 | View 계층, 메뉴 트리, 권한 그룹, 주문 구성 |

- 컴포지트 패턴은 **"하나와 여럿을 구별하지 않고 동일하게 다루는 것"** 이 핵심입니다.
- `is` 타입 체크가 재귀 구조에 반복된다면 컴포지트 패턴 적용을 고려하세요.

---

## 참고

- Design Patterns — GoF (Gang of Four)
- [어댑터 패턴 포스팅 보기](/posts/2026-05-06-adapter)
- [데코레이터 패턴 포스팅 보기](/posts/2026-05-07-decorator)
