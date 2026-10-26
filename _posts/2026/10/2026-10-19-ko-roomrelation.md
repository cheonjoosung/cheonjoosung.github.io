---
title: (Android/Room) Room Relation — one-to-many, many-to-many
tags: [ Android, Room ]
style: fill
color: dark
description: Room에서 일대다, 다대다 관계를 @Relation, @Embedded, Junction으로 표현하고 조회하는 방법과 중첩 관계, 성능 주의점을 정리합니다.
---

---

## 1. Room은 객체 간 참조를 직접 저장하지 않는다

Room 엔티티는 다른 엔티티를 필드로 가질 수 없습니다. 대신 **외래 키(ID)** 로 연결하고, 조회 시 `@Relation`으로 합쳐서 가져옵니다.

```kotlin
// ❌ Room은 엔티티 안에 다른 엔티티 리스트를 저장할 수 없음
@Entity
data class User(val id: Long, val posts: List<Post>)
```

---

## 2. 일대다 (One-to-Many)

사용자 한 명이 여러 게시글을 가지는 경우입니다.

```kotlin
@Entity(tableName = "users")
data class User(
    @PrimaryKey val userId: Long,
    val name: String
)

@Entity(
    tableName = "posts",
    foreignKeys = [ForeignKey(
        entity = User::class,
        parentColumns = ["userId"],
        childColumns = ["authorId"],
        onDelete = ForeignKey.CASCADE      // 사용자 삭제 시 게시글도 삭제
    )],
    indices = [Index("authorId")]           // 외래 키에는 인덱스 권장 (조인 성능)
)
data class Post(
    @PrimaryKey val postId: Long,
    val authorId: Long,                     // User와 연결되는 외래 키
    val title: String
)
```

조회용 클래스(**엔티티가 아닌 데이터 클래스**)를 만듭니다.

```kotlin
data class UserWithPosts(
    @Embedded val user: User,               // 부모 컬럼을 그대로 포함
    @Relation(
        parentColumn = "userId",            // 부모의 키
        entityColumn = "authorId"           // 자식의 외래 키
    )
    val posts: List<Post>
)

@Dao
interface UserDao {
    @Transaction                            // 두 번의 쿼리를 일관되게 실행
    @Query("SELECT * FROM users")
    fun observeUsersWithPosts(): Flow<List<UserWithPosts>>
}
```

> `@Relation`은 내부적으로 **쿼리 두 번**(부모 조회 → 자식 조회)을 실행하므로 `@Transaction`을 붙여 중간에 데이터가 바뀌어도 일관되게 읽게 합니다.

---

## 3. 일대일 (One-to-One)

```kotlin
data class UserWithProfile(
    @Embedded val user: User,
    @Relation(parentColumn = "userId", entityColumn = "ownerId")
    val profile: Profile?                   // 단일 객체 (없을 수 있으면 nullable)
)
```

---

## 4. 다대다 (Many-to-Many)

학생과 수업처럼 양쪽 모두 여러 개를 가지는 관계는 **연결(교차) 테이블**이 필요합니다.

```kotlin
@Entity(tableName = "students")
data class Student(@PrimaryKey val studentId: Long, val name: String)

@Entity(tableName = "courses")
data class Course(@PrimaryKey val courseId: Long, val title: String)

// 연결 테이블: 두 키를 복합 기본 키로
@Entity(
    tableName = "student_course",
    primaryKeys = ["studentId", "courseId"],
    indices = [Index("courseId")]
)
data class StudentCourseCrossRef(
    val studentId: Long,
    val courseId: Long
)
```

```kotlin
// 학생 → 수강 과목들
data class StudentWithCourses(
    @Embedded val student: Student,
    @Relation(
        parentColumn = "studentId",
        entityColumn = "courseId",
        associateBy = Junction(StudentCourseCrossRef::class)   // 연결 테이블 지정
    )
    val courses: List<Course>
)

// 반대 방향: 과목 → 수강 학생들
data class CourseWithStudents(
    @Embedded val course: Course,
    @Relation(
        parentColumn = "courseId",
        entityColumn = "studentId",
        associateBy = Junction(StudentCourseCrossRef::class)
    )
    val students: List<Student>
)
```

`Junction`에 연결 테이블을 지정하면 Room이 조인을 처리합니다. 양방향 조회는 **각각 별도 클래스**로 만듭니다.

---

## 5. 저장은 직접 — Room이 자동 삽입하지 않는다

관계 객체를 한 번에 insert하는 기능은 없습니다. 부모/자식/연결 테이블을 각각 저장합니다.

```kotlin
@Dao
interface SchoolDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertStudent(student: Student)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertCourse(course: Course)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun enroll(ref: StudentCourseCrossRef)

    @Transaction
    suspend fun enrollStudent(student: Student, courses: List<Course>) {
        insertStudent(student)
        courses.forEach {
            insertCourse(it)
            enroll(StudentCourseCrossRef(student.studentId, it.courseId))
        }
    }
}
```

---

## 6. 중첩 관계

`@Relation`의 대상 클래스가 다시 관계 클래스일 수 있습니다.

```kotlin
data class PostWithComments(
    @Embedded val post: Post,
    @Relation(parentColumn = "postId", entityColumn = "postOwnerId")
    val comments: List<Comment>
)

data class UserWithPostsAndComments(
    @Embedded val user: User,
    @Relation(
        entity = Post::class,               // 중간 엔티티를 명시
        parentColumn = "userId",
        entityColumn = "authorId"
    )
    val posts: List<PostWithComments>
)
```

---

## 7. 성능 주의

- `@Relation`은 **자식 전체를 메모리에 로드**합니다. 자식이 수천 개면 부담이 되므로 페이징이나 별도 쿼리를 사용하세요
- 목록 화면에서 자식 개수만 필요하다면 `COUNT`를 직접 쿼리하는 것이 훨씬 효율적입니다
- 외래 키 컬럼에는 `@Index`를 붙여 조인 속도를 확보하세요

---

## 8. 정리

- 엔티티는 외래 키(ID)로 연결하고, 조회용 클래스에서 `@Embedded` + `@Relation`으로 합침
- 일대다: `parentColumn` ↔ `entityColumn`, 다대다: `associateBy = Junction(...)`
- 다대다는 **연결 테이블(복합 PK)** 이 필수, 양방향은 각각 클래스 작성
- 관계 조회에는 `@Transaction`, 삽입은 부모/자식/연결을 직접 저장
- 자식이 많으면 전체 로드 비용에 주의
