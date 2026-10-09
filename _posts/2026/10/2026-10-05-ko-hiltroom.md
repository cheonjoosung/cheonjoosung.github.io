---
title: (Android/Hilt) Hilt + Room 연동
tags: [ Android, Hilt, Room ]
style: fill
color: dark
description: Hilt로 Room Database와 DAO를 주입하는 방법과 Repository 연결, 테스트용 인메모리 DB 교체 패턴을 정리합니다.
---

---

## 1. 연동 구조

Room의 `Database`는 `Room.databaseBuilder()`로 생성하므로 `@Inject constructor`를 쓸 수 없습니다.
따라서 **Module의 `@Provides`** 로 제공하고, DAO는 Database에서 꺼내 제공합니다.

```
Context → AppDatabase → UserDao → UserRepository → ViewModel
 (Hilt)    (@Provides)  (@Provides)  (@Binds)       (@HiltViewModel)
```

---

## 2. Room 구성 요소

```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun observeAll(): Flow<List<UserEntity>>

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun findById(id: String): UserEntity?

    @Upsert
    suspend fun upsert(user: UserEntity)
}

@Database(entities = [UserEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

---

## 3. DatabaseModule

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton  // DB 인스턴스는 앱 전체에서 하나만 — 필수!
    fun provideDatabase(
        @ApplicationContext context: Context  // Hilt가 기본 제공하는 Application Context
    ): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .fallbackToDestructiveMigration()
            .build()

    // DAO는 Database에서 꺼내서 제공 — 별도 @Singleton 불필요 (Database가 싱글턴이므로)
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao = database.userDao()
}
```

> **Database는 반드시 @Singleton**: Room DB 인스턴스 생성은 비용이 크고, 여러 인스턴스가 같은 파일에 접근하면 데이터 불일치가 생길 수 있습니다.

---

## 4. Repository 연결

```kotlin
interface UserRepository {
    fun observeUsers(): Flow<List<User>>
    suspend fun saveUser(user: User)
}

class UserRepositoryImpl @Inject constructor(
    private val userDao: UserDao,  // Hilt가 DatabaseModule에서 찾아 주입
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) : UserRepository {

    override fun observeUsers(): Flow<List<User>> =
        userDao.observeAll().map { list -> list.map { it.toDomain() } }

    override suspend fun saveUser(user: User) = withContext(ioDispatcher) {
        userDao.upsert(user.toEntity())
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

ViewModel은 `UserRepository` 인터페이스만 알면 됩니다.

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    repository: UserRepository
) : ViewModel() {
    val users = repository.observeUsers()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}
```

---

## 5. 테스트: 인메모리 DB로 교체

운영용 `DatabaseModule`을 테스트에서 **인메모리 DB**로 바꿔 끼울 수 있습니다 (자세한 내용은 Hilt Testing 글 참고).

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [DatabaseModule::class]  // 운영 모듈을 대체
)
object TestDatabaseModule {
    @Provides
    @Singleton
    fun provideInMemoryDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()
            .build()
}
```

---

## 6. 정리

- Room DB는 생성자에 `@Inject`를 붙일 수 없으므로 `@Provides`로 제공
- `AppDatabase`는 반드시 `@Singleton`, DAO는 Database에서 꺼내 제공
- 흐름: `Context → Database → DAO → Repository → ViewModel` 모두 Hilt가 자동 연결
- Repository는 `@Binds`로 인터페이스와 구현체 매핑
- 테스트에서는 `@TestInstallIn`으로 인메모리 DB로 교체
