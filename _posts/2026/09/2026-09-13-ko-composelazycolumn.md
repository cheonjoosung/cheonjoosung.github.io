---
title: (Android/Compose) Compose LazyColumn / LazyGrid — RecyclerView 대체
tags: [ Android, Compose ]
style: fill
color: dark
description: Jetpack Compose의 LazyColumn, LazyRow, LazyGrid로 대용량 목록을 구현하는 방법과 RecyclerView와의 차이점을 정리합니다.
---

---

## 1. Lazy 컴포넌트란?

Compose에서 **LazyColumn, LazyRow, LazyGrid**는 RecyclerView의 역할을 합니다.
"Lazy"라는 이름처럼 **화면에 보이는 아이템만** 그립니다—수만 개의 데이터도 메모리 효율적으로 처리합니다.

```
기존 RecyclerView:
  - RecyclerView.Adapter 구현 (ViewHolder, onBindViewHolder 등)
  - XML 아이템 레이아웃 별도 작성

Compose LazyColumn:
  - LazyColumn { items(list) { item -> ItemComposable(item) } }
  - 단순하고 선언적
```

---

## 2. LazyColumn — 세로 스크롤 목록

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items

data class Message(val id: Int, val sender: String, val content: String)

@Composable
fun MessageList(messages: List<Message>) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),  // 목록 전체의 패딩
        verticalArrangement = Arrangement.spacedBy(8.dp)  // 아이템 사이 간격
    ) {
        // items(): 리스트 아이템 렌더링
        // key = { it.id }: 각 아이템에 고유 키를 부여 → 재구성 최적화
        items(
            items = messages,
            key = { it.id }  // 아이템 재정렬 시 애니메이션 최적화에 필수
        ) { message ->
            MessageItem(message)  // 아이템 Composable 호출
        }
    }
}

@Composable
fun MessageItem(message: Message) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(2.dp)
    ) {
        Column(modifier = Modifier.padding(12.dp)) {
            Text(
                text = message.sender,
                style = MaterialTheme.typography.labelMedium,
                color = MaterialTheme.colorScheme.primary
            )
            Spacer(modifier = Modifier.height(4.dp))
            Text(
                text = message.content,
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

---

## 3. 다양한 items 함수

LazyColumn 안에서 아이템을 구성하는 DSL 함수들입니다.

```kotlin
@Composable
fun MixedList(users: List<User>, products: List<Product>) {
    LazyColumn {
        // item { }: 단일 아이템 (헤더, 푸터 등)
        item {
            Text(
                text = "사용자 목록",
                style = MaterialTheme.typography.headlineSmall,
                modifier = Modifier.padding(16.dp)
            )
        }

        // items(list) { }: 리스트 아이템 반복
        items(users, key = { it.id }) { user ->
            UserRow(user)
        }

        // item { }: 구분선
        item { Divider(modifier = Modifier.padding(vertical = 8.dp)) }

        item {
            Text(
                text = "상품 목록",
                style = MaterialTheme.typography.headlineSmall,
                modifier = Modifier.padding(16.dp)
            )
        }

        // itemsIndexed: 인덱스도 함께 받음
        itemsIndexed(products) { index, product ->
            ProductRow(index = index + 1, product = product)
        }

        // item { }: 하단 로딩 인디케이터 (더 불러오기 구현 시)
        item {
            Box(
                modifier = Modifier.fillMaxWidth().padding(16.dp),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator()
            }
        }
    }
}
```

---

## 4. LazyRow — 가로 스크롤 목록

```kotlin
@Composable
fun CategoryRow(categories: List<Category>) {
    LazyRow(
        contentPadding = PaddingValues(horizontal = 16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(categories, key = { it.id }) { category ->
            CategoryChip(category)
        }
    }
}

@Composable
fun CategoryChip(category: Category) {
    var selected by remember { mutableStateOf(false) }

    FilterChip(
        selected = selected,
        onClick = { selected = !selected },
        label = { Text(category.name) },
        leadingIcon = if (selected) {
            { Icon(Icons.Default.Check, contentDescription = null) }
        } else null
    )
}
```

---

## 5. LazyGrid — 그리드 목록

```kotlin
import androidx.compose.foundation.lazy.grid.GridCells
import androidx.compose.foundation.lazy.grid.LazyVerticalGrid

@Composable
fun PhotoGrid(photos: List<Photo>) {
    LazyVerticalGrid(
        // GridCells.Fixed(n): 항상 n열
        // GridCells.Adaptive(minSize): 최소 크기를 유지하며 열 수 자동 조정
        columns = GridCells.Adaptive(minSize = 120.dp),
        contentPadding = PaddingValues(8.dp),
        horizontalArrangement = Arrangement.spacedBy(4.dp),
        verticalArrangement = Arrangement.spacedBy(4.dp)
    ) {
        items(photos, key = { it.id }) { photo ->
            PhotoThumbnail(photo)
        }
    }
}

@Composable
fun PhotoThumbnail(photo: Photo) {
    AsyncImage(  // Coil 라이브러리: 이미지를 비동기로 로드
        model = photo.url,
        contentDescription = photo.description,
        contentScale = ContentScale.Crop,  // 이미지를 잘라서 꽉 채움
        modifier = Modifier
            .aspectRatio(1f)               // 정사각형 비율 유지
            .clip(RoundedCornerShape(8.dp))
    )
}
```

---

## 6. 스크롤 상태 제어

```kotlin
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.foundation.lazy.LazyListState

@Composable
fun ScrollControlExample() {
    // LazyListState: 스크롤 위치, 첫 번째 보이는 아이템 정보 등을 제공
    val listState = rememberLazyListState()
    val coroutineScope = rememberCoroutineScope()

    val items = (1..50).map { "아이템 $it" }

    Box {
        LazyColumn(state = listState) {
            items(items) { item ->
                Text(
                    text = item,
                    modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp)
                )
            }
        }

        // 맨 위로 버튼: 스크롤이 어느 정도 내려갔을 때만 표시
        // derivedStateOf: listState.firstVisibleItemIndex가 바뀔 때만 재계산
        val showButton by remember {
            derivedStateOf { listState.firstVisibleItemIndex > 3 }
        }

        if (showButton) {
            FloatingActionButton(
                onClick = {
                    // animateScrollToItem: 애니메이션과 함께 특정 인덱스로 스크롤
                    coroutineScope.launch { listState.animateScrollToItem(0) }
                },
                modifier = Modifier.align(Alignment.BottomEnd).padding(16.dp)
            ) {
                Icon(Icons.Default.KeyboardArrowUp, "맨 위로")
            }
        }
    }
}
```

> **derivedStateOf**: `listState.firstVisibleItemIndex`는 스크롤할 때마다 바뀝니다. `derivedStateOf`로 감싸면 "3 초과"라는 결과가 바뀔 때만 재구성됩니다. 성능 최적화에 중요합니다.

---

## 7. 무한 스크롤 (페이지네이션)

```kotlin
@Composable
fun InfiniteScrollList(viewModel: ProductViewModel = viewModel()) {
    val products by viewModel.products.collectAsState()
    val isLoading by viewModel.isLoading.collectAsState()
    val listState = rememberLazyListState()

    // 마지막 아이템이 보이면 다음 페이지 로드
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisibleIndex = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index
            val totalItems = listState.layoutInfo.totalItemsCount
            // 마지막에서 5개 이내에 도달하면 로드
            lastVisibleIndex != null && lastVisibleIndex >= totalItems - 5
        }
    }

    // shouldLoadMore가 true가 될 때 다음 페이지 요청
    LaunchedEffect(shouldLoadMore) {
        if (shouldLoadMore && !isLoading) viewModel.loadNextPage()
    }

    LazyColumn(state = listState) {
        items(products, key = { it.id }) { product ->
            ProductItem(product)
        }
        // 로딩 중일 때 하단에 인디케이터 표시
        if (isLoading) {
            item {
                Box(Modifier.fillMaxWidth().padding(16.dp), Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}
```

---

## 8. 정리

| 컴포넌트 | 역할 | RecyclerView 비교 |
|----------|------|-----------------|
| `LazyColumn` | 세로 스크롤 목록 | LinearLayoutManager(VERTICAL) |
| `LazyRow` | 가로 스크롤 목록 | LinearLayoutManager(HORIZONTAL) |
| `LazyVerticalGrid` | 세로 그리드 | GridLayoutManager |
| `LazyHorizontalGrid` | 가로 그리드 | - |

- `key = { it.id }` 설정: 아이템 재정렬 시 애니메이션 최적화 + 재구성 최소화
- `contentPadding`: 스크롤해도 패딩이 유지됨 (일반 `Modifier.padding`과 다름)
- `rememberLazyListState()`: 스크롤 위치 제어, 첫 번째 보이는 아이템 추적에 사용
- `derivedStateOf`: 스크롤 관련 조건식은 반드시 `derivedStateOf`로 감싸야 성능 문제 방지
