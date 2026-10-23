---
title: (Android/Paging) Paging3 + Room — 오프라인 캐시 전략
tags: [ Android, Paging, Room ]
style: fill
color: dark
description: Paging3 RemoteMediator로 네트워크 데이터를 Room에 캐시하고 DB를 단일 진실 공급원으로 사용하는 오프라인 지원 페이징 구현을 정리합니다.
---

---

## 1. 왜 Room과 함께 쓰나?

네트워크만 쓰면 오프라인에서 목록이 비고, 화면 재진입마다 재요청합니다.
**DB를 단일 진실 공급원(SSOT)** 으로 두면 오프라인에서도 목록이 보이고 로딩이 빨라집니다.

```
          ┌──────────── RemoteMediator ────────────┐
          │  (필요할 때 서버에서 가져와 DB에 저장)  │
   서버 ──┘                                         ▼
                                                   Room DB
                                                     │ PagingSource (Room 자동 생성)
                                                     ▼
                                                   Pager → UI
```

UI는 **항상 DB만 관찰**하고, 서버 데이터는 `RemoteMediator`가 DB에 채워 넣습니다.

---

## 2. Room 구성

Room이 `PagingSource`를 직접 반환할 수 있습니다.

```kotlin
@Entity(tableName = "articles")
data class ArticleEntity(
    @PrimaryKey val id: Long,
    val title: String,
    val content: String
)

// 다음 페이지 키를 저장하는 테이블
@Entity(tableName = "remote_keys")
data class RemoteKey(
    @PrimaryKey val articleId: Long,
    val prevKey: Int?,
    val nextKey: Int?
)

@Dao
interface ArticleDao {
    @Query("SELECT * FROM articles ORDER BY id DESC")
    fun pagingSource(): PagingSource<Int, ArticleEntity>   // Room이 구현 생성

    @Upsert
    suspend fun upsertAll(items: List<ArticleEntity>)

    @Query("DELETE FROM articles")
    suspend fun clearAll()
}

@Dao
interface RemoteKeyDao {
    @Query("SELECT * FROM remote_keys WHERE articleId = :id")
    suspend fun remoteKeyById(id: Long): RemoteKey?

    @Upsert
    suspend fun upsertAll(keys: List<RemoteKey>)

    @Query("DELETE FROM remote_keys")
    suspend fun clearAll()
}
```

---

## 3. RemoteMediator — 서버 → DB

`load()`는 **언제 어떤 페이지를 서버에서 가져올지** 결정합니다.

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRemoteMediator(
    private val api: ArticleApi,
    private val db: AppDatabase
) : RemoteMediator<Int, ArticleEntity>() {

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, ArticleEntity>
    ): MediatorResult {

        val page = when (loadType) {
            LoadType.REFRESH -> 1                         // 처음부터
            LoadType.PREPEND -> return MediatorResult.Success(endOfPaginationReached = true)
            LoadType.APPEND -> {
                // 마지막 아이템의 키에서 다음 페이지를 알아냄
                val lastItem = state.lastItemOrNull()
                    ?: return MediatorResult.Success(endOfPaginationReached = false)
                val key = db.remoteKeyDao().remoteKeyById(lastItem.id)
                key?.nextKey
                    ?: return MediatorResult.Success(endOfPaginationReached = true)
            }
        }

        return try {
            val response = api.getArticles(page, state.config.pageSize)
            val endReached = !response.hasNext

            // 삭제와 삽입을 하나의 트랜잭션으로 (중간 상태 노출 방지)
            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    db.remoteKeyDao().clearAll()
                    db.articleDao().clearAll()
                }
                val keys = response.items.map {
                    RemoteKey(it.id, prevKey = if (page == 1) null else page - 1,
                              nextKey = if (endReached) null else page + 1)
                }
                db.remoteKeyDao().upsertAll(keys)
                db.articleDao().upsertAll(response.items.map { it.toEntity() })
            }
            MediatorResult.Success(endOfPaginationReached = endReached)
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }

    // 캐시가 충분히 신선하면 시작 시 새로고침 생략
    override suspend fun initialize(): InitializeAction {
        val cacheTimeout = TimeUnit.HOURS.toMillis(1)
        val lastUpdated = db.metaDao().lastUpdated() ?: 0
        return if (System.currentTimeMillis() - lastUpdated < cacheTimeout)
            InitializeAction.SKIP_INITIAL_REFRESH      // 캐시 그대로 사용
        else
            InitializeAction.LAUNCH_INITIAL_REFRESH    // 서버에서 새로 가져옴
    }
}
```

---

## 4. Pager 연결

```kotlin
@OptIn(ExperimentalPagingApi::class)
class ArticleRepository @Inject constructor(
    private val api: ArticleApi,
    private val db: AppDatabase
) {
    fun getArticles(): Flow<PagingData<Article>> = Pager(
        config = PagingConfig(pageSize = 20),
        remoteMediator = ArticleRemoteMediator(api, db),
        pagingSourceFactory = { db.articleDao().pagingSource() }  // UI는 DB만 바라봄
    ).flow.map { pagingData -> pagingData.map { it.toDomain() } }
}
```

ViewModel/UI 코드는 네트워크 전용 방식과 **동일**합니다 (`cachedIn` + `collectAsLazyPagingItems`).

---

## 5. 동작 흐름

```
1. 앱 시작 → DB의 기존 데이터를 즉시 표시 (오프라인 OK)
2. initialize() 판단 → 오래됐으면 REFRESH로 서버 데이터를 DB에 저장
3. 스크롤 끝에 도달 → APPEND로 다음 페이지를 DB에 저장
4. DB가 바뀌면 Room의 PagingSource가 무효화 → UI 자동 갱신
```

---

## 6. 주의사항

- **REFRESH 시 삭제+삽입은 반드시 한 트랜잭션**: 분리하면 화면이 깜빡이거나 빈 목록이 보입니다
- 에러가 나도 DB의 기존 데이터는 계속 보이므로, `loadState.mediator`로 에러만 별도 표시합니다
- 서버와 DB 스키마가 다르면 `toEntity()`/`toDomain()` 매퍼로 분리
- 정렬 기준이 서버 순서와 다르면 키 계산이 어긋날 수 있으니 서버 순서를 DB에도 보존 (`ORDER BY` 컬럼 또는 순번 저장)

---

## 7. 정리

- **DB = 단일 진실 공급원**, 서버 데이터는 `RemoteMediator`가 DB에 채움
- Room의 DAO가 `PagingSource`를 직접 반환 → `Pager(remoteMediator, pagingSourceFactory)`
- 다음 페이지 키는 `remote_keys` 테이블에 저장해 APPEND 시 사용
- `initialize()`로 캐시 유효 시간을 관리해 불필요한 새로고침 방지
- REFRESH의 삭제/삽입은 `withTransaction`으로 묶기
