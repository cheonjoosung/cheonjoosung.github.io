---
title: (Kotlin/코틀린) 행동 패턴 — 책임 연쇄 패턴 (Chain of Responsibility)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 책임 연쇄 패턴을 구현하는 방법과 로그 레벨 처리, HTTP 미들웨어, 결재 승인 체인 실전 예제를 정리합니다.
---

---

## 1. 책임 연쇄 패턴이란?

**요청을 처리할 수 있는 객체를 연결해 순서대로 처리 기회를 부여**하는 패턴입니다.
각 핸들러는 요청을 처리하거나, 처리하지 못하면 다음 핸들러로 전달합니다.
요청을 보내는 쪽은 어떤 핸들러가 처리할지 알 필요가 없습니다.

```
요청 → Handler1 → Handler2 → Handler3 → null(처리 불가)
          ↓           ↓           ↓
       처리 or     처리 or     처리 or
       다음으로    다음으로    끝(null)
```

---

## 2. 기본 구현

`setNext()`로 핸들러를 연결하고, `handle()`에서 처리 여부를 결정합니다.
`setNext()`가 연결된 핸들러를 반환해 체이닝 문법을 지원합니다.

```kotlin
abstract class Handler {
    // 다음 핸들러 — null이면 체인의 끝
    private var next: Handler? = null

    // 다음 핸들러를 설정하고 그 핸들러를 반환 → 체이닝 가능
    // small.setNext(medium).setNext(large) 처럼 연결
    fun setNext(handler: Handler): Handler {
        next = handler
        return handler  // 반환해야 체이닝 가능
    }

    fun handle(request: Int) {
        if (canHandle(request)) {
            process(request)         // 이 핸들러가 처리 가능하면 직접 처리
        } else {
            // 처리 불가 → 다음 핸들러에게 전달 (없으면 처리 불가 출력)
            next?.handle(request) ?: println("처리 불가: $request")
        }
    }

    protected abstract fun canHandle(request: Int): Boolean
    protected abstract fun process(request: Int)
}

class SmallHandler  : Handler() {
    override fun canHandle(r: Int) = r < 10     // 10 미만만 처리
    override fun process(r: Int) = println("소형 핸들러: $r 처리")
}

class MediumHandler : Handler() {
    override fun canHandle(r: Int) = r < 100    // 100 미만만 처리
    override fun process(r: Int) = println("중형 핸들러: $r 처리")
}

class LargeHandler  : Handler() {
    override fun canHandle(r: Int) = r < 1000   // 1000 미만만 처리
    override fun process(r: Int) = println("대형 핸들러: $r 처리")
}

// 체인 구성: small → medium → large 순서로 연결
val small = SmallHandler()
val medium = MediumHandler()
val large = LargeHandler()
small.setNext(medium).setNext(large)  // medium의 next가 large가 됨

// 항상 체인의 첫 번째(small)에게 요청을 보냄
small.handle(5)    // small이 처리 (5 < 10)
small.handle(50)   // small 패스 → medium이 처리 (50 < 100)
small.handle(500)  // small 패스 → medium 패스 → large가 처리
small.handle(5000) // 모두 패스 → 처리 불가
```

---

## 3. 실전 예제 — 로그 레벨 처리

로그 레벨에 따라 다른 출력 대상으로 보내는 전형적인 사용 예입니다.
한 메시지가 **여러 핸들러에서 동시에 처리**될 수도 있습니다(처리 후 다음으로 전달).

```kotlin
enum class LogLevel { DEBUG, INFO, WARN, ERROR }  // 숫자가 클수록 심각

abstract class Logger(private val level: LogLevel) {
    private var next: Logger? = null

    fun setNext(logger: Logger): Logger { next = logger; return logger }

    fun log(msgLevel: LogLevel, message: String) {
        // 이 로거의 레벨 이상의 메시지만 처리 (예: WARN 로거는 WARN, ERROR만 처리)
        if (msgLevel >= level) write(message)
        // 처리 여부와 관계없이 다음 로거에도 전달 (모든 핸들러가 볼 수 있음)
        next?.log(msgLevel, message)
    }

    protected abstract fun write(message: String)
}

class ConsoleLogger : Logger(LogLevel.DEBUG) {  // DEBUG 이상 모두 처리
    override fun write(msg: String) = println("[콘솔] $msg")
}

class FileLogger    : Logger(LogLevel.WARN) {   // WARN 이상만 처리
    override fun write(msg: String) = println("[파일] $msg")
}

class AlertLogger   : Logger(LogLevel.ERROR) {  // ERROR만 처리
    override fun write(msg: String) = println("[알림] $msg ← 즉시 대응 필요!")
}

val console = ConsoleLogger()
console.setNext(FileLogger()).setNext(AlertLogger())

console.log(LogLevel.DEBUG, "디버그 메시지")
// [콘솔] 디버그 메시지  (콘솔만 DEBUG 이상 → 파일/알림은 레벨 미달)

console.log(LogLevel.WARN, "경고 발생")
// [콘솔] 경고 발생  (DEBUG 이상)
// [파일] 경고 발생  (WARN 이상)

console.log(LogLevel.ERROR, "시스템 오류")
// [콘솔] 시스템 오류  (DEBUG 이상)
// [파일] 시스템 오류  (WARN 이상)
// [알림] 시스템 오류 ← 즉시 대응 필요!  (ERROR)
```

> **처리 후 계속 vs 처리 후 중단**: 기본 구현에서는 처리하면 중단했지만, 여기서는 처리 후에도 다음으로 계속 전달합니다. 요구사항에 따라 선택합니다.

---

## 4. 실전 예제 — 결재 승인 체인

직급별 결재 한도를 체인으로 표현합니다. 현실에서 자주 볼 수 있는 시나리오입니다.

```kotlin
data class PurchaseRequest(val amount: Int, val purpose: String)

// 각 결재자가 한도(limit)를 가짐
abstract class Approver(protected val name: String, protected val limit: Int) {
    private var next: Approver? = null

    fun setNext(approver: Approver): Approver { next = approver; return approver }

    fun approve(request: PurchaseRequest) {
        when {
            // 이 결재자 한도 이내 → 직접 승인
            request.amount <= limit ->
                println("[$name] ${request.purpose}: ${request.amount}원 승인")
            // 한도 초과, 다음 결재자가 있음 → 상위 결재자에게 전달
            next != null ->
                next!!.approve(request)
            // 한도 초과, 더 이상 결재자 없음 → 승인 불가
            else ->
                println("${request.purpose}: ${request.amount}원 — 승인 권한자 없음")
        }
    }
}

class TeamLead : Approver("팀장", 100_000)      // 10만원 이하
class Manager  : Approver("부장", 500_000)      // 50만원 이하
class Director : Approver("이사", 2_000_000)    // 200만원 이하
class CEO      : Approver("대표", Int.MAX_VALUE) // 금액 제한 없음

// 체인: 팀장 → 부장 → 이사 → 대표
val teamLead = TeamLead()
teamLead.setNext(Manager()).setNext(Director()).setNext(CEO())

// 항상 체인의 시작(팀장)에게 요청
teamLead.approve(PurchaseRequest(50_000,    "사무용품"))    // [팀장] 승인
teamLead.approve(PurchaseRequest(300_000,   "장비 구매"))   // [부장] 승인
teamLead.approve(PurchaseRequest(1_500_000, "서버 증설"))   // [이사] 승인
teamLead.approve(PurchaseRequest(5_000_000, "사옥 임대"))   // [대표] 승인
```

---

## 5. 함수 체인으로 구현 (Kotlin 스타일)

함수 타입으로 미들웨어 체인을 구성합니다. HTTP 미들웨어(OkHttp Interceptor, Ktor)와 동일한 원리입니다.

```kotlin
// Middleware: (요청, 다음 핸들러) → 응답?
typealias Middleware = (request: String, next: (String) -> String?) -> String?

// 미들웨어 배열을 하나의 함수로 합성
fun chain(vararg middlewares: Middleware): (String) -> String? {
    return { request ->
        // buildChain: index번째 미들웨어부터 시작하는 체인 생성
        fun buildChain(index: Int): (String) -> String? = { req ->
            if (index >= middlewares.size) null  // 모든 미들웨어 통과 → null 반환
            else middlewares[index](req, buildChain(index + 1))  // 현재 → 다음
        }
        buildChain(0)(request)
    }
}

// 인증 미들웨어: "auth:" 접두사가 없으면 요청 차단
val authMiddleware: Middleware = { req, next ->
    if (req.startsWith("auth:")) next(req.removePrefix("auth:"))  // 통과
    else { println("인증 실패"); null }  // 차단
}

// 로깅 미들웨어: 요청/응답을 기록하고 다음 미들웨어로 전달
val loggingMiddleware: Middleware = { req, next ->
    println("요청: $req")
    next(req).also { println("응답: $it") }  // also: 응답값을 그대로 반환하면서 로그
}

// 로깅 → 인증 순서로 체인 구성
val handler = chain(loggingMiddleware, authMiddleware)

handler("auth:GET /users")  // 요청 로그 → 인증 통과 → 응답 로그
handler("GET /admin")       // 요청 로그 → 인증 실패 → null
```

---

## 6. 정리

- 책임 연쇄: 요청을 체인으로 전달, 처리 가능한 핸들러가 담당
- `setNext()` 반환값을 핸들러 자신으로 설정해 빌더 스타일 체이닝 가능
- **처리 후 중단** vs **처리 후 계속**: 사용 목적에 따라 선택
- 실전 활용: 로그 레벨, 결재 체인, HTTP 미들웨어, 폼 유효성 검사 체인
- Kotlin 함수 타입: 클래스 없이 미들웨어 체인을 유연하게 구성 가능
