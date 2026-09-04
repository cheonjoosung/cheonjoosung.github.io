---
title: (Kotlin/코틀린) 행동 패턴 — 템플릿 메서드 패턴 (Template Method)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 템플릿 메서드 패턴을 구현하는 방법과 abstract class, hook 메서드, 함수 타입 활용 패턴을 정리합니다.
---

---

## 1. 템플릿 메서드 패턴이란?

**알고리즘의 뼈대(순서)를 부모 클래스에 정의하고, 세부 단계를 서브클래스가 구현**하는 패턴입니다.
"무엇을 하는지(순서)"는 부모가 결정하고, "어떻게 하는지(구현)"는 자식이 결정합니다.

```
AbstractClass
  templateMethod() {
    step1()  ← 공통 (부모가 직접 구현)
    step2()  ← 추상 (서브클래스가 반드시 구현)
    step3()  ← hook (서브클래스가 선택적으로 오버라이드)
  }
```

- **추상 메서드(abstract fun)**: 서브클래스에서 반드시 구현해야 하는 단계
- **Hook 메서드(open fun)**: 기본 구현이 있어 선택적으로 오버라이드

---

## 2. 기본 구현

데이터 처리 파이프라인을 예로 들겠습니다. 읽기→처리→저장 순서는 모든 프로세서가 공통으로 따르지만, 각 단계의 구현은 다릅니다.

```kotlin
abstract class DataProcessor {
    // 템플릿 메서드: 알고리즘의 뼈대를 정의
    // 서브클래스가 이 순서를 바꿀 수 없음 (Kotlin에서 기본적으로 open이 아님)
    fun process() {
        readData()       // 1단계: 데이터 읽기 (서브클래스 구현)
        processData()    // 2단계: 데이터 처리 (서브클래스 구현)
        writeData()      // 3단계: 결과 저장 (서브클래스 구현)
        if (shouldLog()) log()  // 4단계: 로그 (hook - 선택적)
    }

    // abstract: 서브클래스에서 반드시 구현해야 함
    protected abstract fun readData()
    protected abstract fun processData()
    protected abstract fun writeData()

    // Hook 메서드: 기본값(false)이 있어 오버라이드하지 않아도 동작
    protected open fun shouldLog() = false
    protected open fun log() = println("처리 완료")
}

// CSV 처리: 세 단계를 CSV에 맞게 구현
class CsvProcessor : DataProcessor() {
    override fun readData()    = println("CSV 파일 읽기")
    override fun processData() = println("CSV 파싱 및 변환")
    override fun writeData()   = println("데이터베이스에 저장")
    // hook 오버라이드: 이 프로세서는 로그를 남김
    override fun shouldLog()   = true
}

// JSON 처리: 세 단계를 JSON에 맞게 구현
class JsonProcessor : DataProcessor() {
    override fun readData()    = println("JSON API 읽기")
    override fun processData() = println("JSON 파싱 및 매핑")
    override fun writeData()   = println("캐시에 저장")
    // shouldLog()를 오버라이드하지 않음 → 기본값 false → 로그 없음
}

CsvProcessor().process()
// CSV 파일 읽기 → CSV 파싱 및 변환 → 데이터베이스에 저장 → 처리 완료

JsonProcessor().process()
// JSON API 읽기 → JSON 파싱 및 매핑 → 캐시에 저장 (로그 없음)
```

> **포인트**: 템플릿 메서드(`process()`)가 알고리즘의 실행 순서를 통제합니다. 서브클래스는 단계의 "내용"만 바꿀 수 있고 "순서"는 바꿀 수 없습니다.

---

## 3. 실전 예제 — 음료 제조

고전적인 패턴 교과서 예제입니다. 음료 제조 순서는 동일하지만 각 단계의 구현이 음료마다 다릅니다.

```kotlin
abstract class Beverage {
    // 최종 메서드: 서브클래스가 순서를 바꿀 수 없음
    fun prepare() {
        boilWater()        // 공통 1단계: 물 끓이기
        brew()             // 다른 2단계: 차/커피 우리기
        pourInCup()        // 공통 3단계: 컵에 따르기
        // hook: customerWantsCondiments()가 true인 경우만 첨가물 추가
        if (customerWantsCondiments()) addCondiments()
    }

    // private: 서브클래스도 오버라이드 불가 (완전히 고정된 단계)
    private fun boilWater() = println("물 끓이기")
    private fun pourInCup() = println("컵에 따르기")

    // 서브클래스가 반드시 구현
    protected abstract fun brew()
    protected abstract fun addCondiments()

    // hook: 기본은 true (첨가물 추가), 서브클래스가 false로 오버라이드 가능
    protected open fun customerWantsCondiments() = true
}

class Tea : Beverage() {
    override fun brew()           = println("차 우려내기")
    override fun addCondiments()  = println("레몬 추가")
    // customerWantsCondiments() → 기본 true → 레몬 추가
}

class Coffee : Beverage() {
    override fun brew()          = println("필터로 커피 추출")
    override fun addCondiments() = println("설탕과 우유 추가")
    // 블랙커피: hook을 false로 → 첨가물 건너뜀
    override fun customerWantsCondiments() = false
}

Tea().prepare()
// 물 끓이기 → 차 우려내기 → 컵에 따르기 → 레몬 추가

Coffee().prepare()
// 물 끓이기 → 필터로 커피 추출 → 컵에 따르기  (첨가물 건너뜀)
```

---

## 4. 함수 타입으로 대체 (Kotlin 스타일)

상속 대신 **함수를 파라미터로 주입**하면 클래스 계층 없이 동일한 효과를 낼 수 있습니다.
재사용성이 높고 테스트하기 쉽습니다.

```kotlin
// 보고서 생성 뼈대: 타이틀 출력 → 데이터 행 추가 → 푸터 출력
// 각 단계를 함수 타입으로 주입받아 유연하게 교체
fun buildReport(
    title: String,
    fetchData: () -> List<String>,         // 데이터 조회 방법 주입
    formatRow: (String) -> String,         // 행 포맷 방법 주입
    footer: (() -> String)? = null         // 선택적 푸터 (hook과 동일한 역할)
): String {
    val sb = StringBuilder()
    sb.appendLine("=== $title ===")
    // fetchData()로 데이터 조회 후 formatRow()로 각 행 포맷
    fetchData().forEach { sb.appendLine(formatRow(it)) }
    // footer가 주어진 경우에만 출력
    footer?.let { sb.appendLine(it()) }
    return sb.toString()
}

// 사용: 뼈대는 고정, 세부 구현만 람다로 주입
val report = buildReport(
    title = "판매 보고서",
    fetchData = { listOf("사과 100개", "바나나 50개", "오렌지 30개") },
    formatRow = { "  - $it" },   // 들여쓰기 포맷
    footer = { "총 3개 품목" }
)
println(report)
// === 판매 보고서 ===
//   - 사과 100개
//   - 바나나 50개
//   - 오렌지 30개
// 총 3개 품목
```

> **언제 상속 vs 함수 주입을 선택?**: 단계가 많고 서브클래스가 상태를 가진다면 상속 방식이 적합합니다. 단계가 단순하고 재사용/테스트가 중요하다면 함수 주입 방식이 더 유연합니다.

---

## 5. 단계 조합 — 파이프라인 패턴

`fold`를 이용해 데이터를 단계별로 변환하는 파이프라인을 만들 수 있습니다.

```kotlin
// 파이프라인: 데이터에 여러 변환 단계를 순서대로 적용
class Pipeline<T>(private var data: T) {
    private val steps = mutableListOf<(T) -> T>()

    // 단계 추가 (메서드 체이닝 지원)
    fun addStep(step: (T) -> T): Pipeline<T> {
        steps.add(step)
        return this  // this 반환으로 체이닝 가능
    }

    // 실행: fold로 초기값(data)에 각 단계를 순서대로 적용
    // fold(초기값) { 누적값, 단계 → 단계(누적값) }
    fun execute(): T = steps.fold(data) { acc, step -> step(acc) }
}

// 문자열 정제 파이프라인: trim → lowercase → 쉼표 제거
val result = Pipeline("  Hello, World!  ")
    .addStep { it.trim() }            // "  Hello, World!  " → "Hello, World!"
    .addStep { it.lowercase() }       // "Hello, World!" → "hello, world!"
    .addStep { it.replace(",", "") }  // "hello, world!" → "hello world!"
    .execute()
println(result)  // "hello world!"
```

---

## 6. 정리

- 템플릿 메서드: 알고리즘 **순서(뼈대)는 부모**, **내용(구현)은 자식**
- `abstract fun`: 서브클래스가 반드시 구현해야 하는 필수 단계
- `open fun` (Hook): 기본 구현 제공, 서브클래스가 선택적으로 오버라이드
- Kotlin에서는 **함수 타입 파라미터**로 상속 없이 동일한 효과 달성 가능
- 뼈대 메서드를 오버라이드하지 못하게 하려면 `final` 또는 `private`으로 선언
