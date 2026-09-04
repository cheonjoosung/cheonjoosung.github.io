---
title: (Kotlin/코틀린) 행동 패턴 — 커맨드 패턴 (Command)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 커맨드 패턴을 구현하는 방법과 실행 취소(Undo), 작업 큐, 매크로 명령 처리 패턴을 정리합니다.
---

---

## 1. 커맨드 패턴이란?

**요청(행동)을 객체로 캡슐화**해 매개변수화, 큐잉, 로깅, 실행 취소를 가능하게 하는 패턴입니다.
"무엇을 할지"를 객체로 만들어 나중에 실행하거나, 취소하거나, 재실행할 수 있습니다.

```
Invoker → Command.execute() → Receiver
               ↕
           Command.undo()
```

- **Invoker**: 커맨드를 호출하는 주체 (버튼, 메뉴, 스케줄러)
- **Command**: 실행/취소 방법을 담은 객체
- **Receiver**: 실제 작업을 수행하는 객체 (에디터, 파일 시스템)

---

## 2. 기본 구현

커맨드 인터페이스에 `execute()`와 `undo()`를 정의하고, 각 동작을 별도 클래스로 구현합니다.

```kotlin
// 커맨드 인터페이스 — 실행과 취소 모두 정의
interface Command {
    fun execute()
    fun undo()
}

// 수신자(Receiver): 실제 편집 로직을 담당
class TextEditor {
    private val content = StringBuilder()

    // 현재 커서 위치에 텍스트 삽입
    fun insertText(text: String) {
        content.append(text)
        println("삽입: \"$text\" → 현재: \"$content\"")
    }

    // 끝에서 length만큼 삭제 (undo 시 사용)
    fun deleteText(length: Int) {
        val removed = content.takeLast(length)
        repeat(length) { content.deleteCharAt(content.lastIndex) }
        println("삭제: \"$removed\" → 현재: \"$content\"")
    }

    fun getText() = content.toString()
}

// 구체 커맨드: 삽입 동작
class InsertCommand(
    private val editor: TextEditor,
    private val text: String
) : Command {
    override fun execute() = editor.insertText(text)
    // undo: 삽입한 글자 수만큼 뒤에서 삭제
    override fun undo() = editor.deleteText(text.length)
}

// 보관자(Caretaker): Undo 스택 관리
class CommandHistory {
    // ArrayDeque를 스택처럼 사용 (addLast/removeLast)
    private val history = ArrayDeque<Command>()

    fun execute(command: Command) {
        command.execute()
        history.addLast(command)  // 실행된 커맨드를 스택에 push
    }

    fun undo() {
        if (history.isEmpty()) println("실행 취소할 명령 없음")
        else history.removeLast().undo()  // 가장 최근 커맨드를 pop 후 undo
    }
}

// 사용
val editor = TextEditor()
val history = CommandHistory()

history.execute(InsertCommand(editor, "Hello"))    // "Hello" 삽입
history.execute(InsertCommand(editor, " World"))   // " World" 삽입 → "Hello World"
history.undo()   // " World" 삭제 → "Hello"
history.undo()   // "Hello" 삭제 → ""
```

> **포인트**: Undo 스택에는 커맨드 객체 자체가 저장됩니다. 각 커맨드가 undo 방법도 알고 있으므로, 보관자(CommandHistory)는 단순히 스택만 관리하면 됩니다.

---

## 3. 람다 기반 커맨드 (Kotlin 스타일)

Kotlin에서는 커맨드 인터페이스 대신 **실행/취소 람다 쌍**을 사용해 클래스 없이 구현할 수 있습니다.

```kotlin
class CommandHistory {
    // execute와 undo를 람다 쌍으로 묶은 데이터 클래스
    data class ReversibleAction(
        val execute: () -> Unit,
        val undo: () -> Unit
    )
    private val history = ArrayDeque<ReversibleAction>()

    fun execute(action: ReversibleAction) {
        action.execute()          // 즉시 실행
        history.addLast(action)   // 스택에 저장
    }

    // removeLastOrNull: 스택이 비어 있어도 예외 없이 null 반환
    fun undo() = history.removeLastOrNull()?.undo() ?: println("실행 취소 없음")
}

val editor = TextEditor()
val history = CommandHistory()

// 별도 클래스 없이 람다로 커맨드 표현
history.execute(CommandHistory.ReversibleAction(
    execute = { editor.insertText("Kotlin") },
    undo    = { editor.deleteText(6) }   // "Kotlin"은 6글자
))

history.undo()  // "Kotlin" 삭제
```

---

## 4. 실전 예제 — 드로잉 앱 커맨드

`sealed class`를 사용하면 커맨드 종류를 명확히 열거하고 `when`으로 타입 안전하게 처리할 수 있습니다.

```kotlin
data class Point(val x: Int, val y: Int)

// sealed class: 가능한 드로잉 커맨드를 컴파일 타임에 열거
sealed class DrawCommand : Command {
    // 선 그리기 커맨드 — from/to 좌표를 캡처
    class DrawLine(private val from: Point, private val to: Point) : DrawCommand() {
        override fun execute() = println("선 그리기: $from → $to")
        override fun undo() = println("선 지우기: $from → $to")
    }

    // 원 그리기 커맨드 — 중심과 반지름을 캡처
    class DrawCircle(private val center: Point, private val radius: Int) : DrawCommand() {
        override fun execute() = println("원 그리기: 중심=$center, 반지름=$radius")
        override fun undo() = println("원 지우기: 중심=$center, 반지름=$radius")
    }

    // 색 채우기 커맨드 — undo를 위해 이전 색(prevColor)도 저장
    class FillColor(
        private val target: Point,
        private val color: String,
        private val prevColor: String  // undo 시 이 색으로 복원
    ) : DrawCommand() {
        override fun execute() = println("색 채우기: $target → $color")
        override fun undo() = println("색 되돌리기: $target → $prevColor")
    }
}

class DrawingApp {
    private val history = ArrayDeque<DrawCommand>()

    fun draw(command: DrawCommand) {
        command.execute()       // 즉시 실행
        history.addLast(command)  // 히스토리에 기록
    }

    fun undo() = history.removeLastOrNull()?.undo() ?: println("되돌릴 작업 없음")
}

val app = DrawingApp()
app.draw(DrawCommand.DrawLine(Point(0, 0), Point(100, 100)))
app.draw(DrawCommand.DrawCircle(Point(50, 50), 30))
app.draw(DrawCommand.FillColor(Point(50, 50), "red", "white"))
app.undo()  // 색 채우기 되돌리기 (red → white)
app.undo()  // 원 지우기
```

---

## 5. 매크로 커맨드 (복합 커맨드)

여러 커맨드를 하나로 묶어 원자적으로 실행/취소합니다. Undo 시 역순으로 취소하는 것이 핵심입니다.

```kotlin
// 복합 커맨드: 여러 커맨드를 순서대로 실행, 역순으로 undo
class MacroCommand(private val commands: List<Command>) : Command {
    override fun execute() = commands.forEach { it.execute() }
    // 역순 undo: 마지막에 실행한 것부터 취소 (스택 원리)
    override fun undo() = commands.reversed().forEach { it.undo() }
}

// 예: 문서 서식 적용을 하나의 매크로로 묶기
val macro = MacroCommand(listOf(
    InsertCommand(editor, "제목"),
    InsertCommand(editor, "\n"),
    InsertCommand(editor, "본문 내용"),
))
history.execute(macro)    // 3개 커맨드 순서대로 실행
history.undo()            // 역순으로 3개 모두 한번에 undo
```

> **포인트**: undo 시 `commands.reversed()`로 역순 실행하는 것이 중요합니다. 예를 들어 A→B→C 순서로 실행했다면, undo는 C→B→A 순서로 해야 합니다.

---

## 6. 작업 큐 패턴

커맨드를 큐에 쌓아 두고 순서대로 처리합니다. 요청자와 실행 시점을 분리할 수 있습니다.

```kotlin
class TaskQueue {
    private val queue = ArrayDeque<Command>()

    // 큐의 뒤에 추가 (FIFO)
    fun enqueue(command: Command) = queue.addLast(command)

    // 큐의 앞에서 하나씩 꺼내 실행
    fun processAll() {
        while (queue.isNotEmpty()) {
            queue.removeFirst().execute()
        }
    }
}

val queue = TaskQueue()
// 작업을 쌓아두고
queue.enqueue(InsertCommand(editor, "Task 1"))
queue.enqueue(InsertCommand(editor, "Task 2"))
queue.enqueue(InsertCommand(editor, "Task 3"))
// 나중에 일괄 실행
queue.processAll()  // Task 1 → Task 2 → Task 3 순서로 실행
```

---

## 7. 정리

- 커맨드 패턴: "무엇을 할지"를 객체로 만들어 실행/취소/큐잉 가능
- Kotlin에서는 **람다 쌍(execute/undo)** 으로 클래스 없이 가볍게 구현 가능
- `sealed class`: 커맨드 종류가 고정된 경우 누락 방지 + 타입 안전
- **매크로 커맨드**: 여러 동작을 하나로 묶어 원자적 실행, 역순 undo
- **작업 큐**: 요청 시점과 실행 시점을 분리해 비동기 처리에 활용
- Redo 구현: undo 시 커맨드를 redo 스택으로 이동, redo 시 다시 execute + undo 스택으로 이동
