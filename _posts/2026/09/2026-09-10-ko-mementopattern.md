---
title: (Kotlin/코틀린) 행동 패턴 — 메멘토 패턴 (Memento)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 메멘토 패턴을 구현하는 방법과 텍스트 편집기 Undo/Redo, 게임 세이브, data class 스냅샷 패턴을 정리합니다.
---

---

## 1. 메멘토 패턴이란?

**객체의 상태를 저장하고 나중에 복원**할 수 있게 하는 패턴입니다.
캡슐화를 유지하면서도 외부에 스냅샷(메멘토)을 보관할 수 있습니다.

```
Originator (원본 객체)
    createMemento() ──→ Memento (상태 복사본)
    restoreFrom(m)  ←── Memento (복원)

Caretaker (보관자)
    undo 스택 / redo 스택에 메멘토 보관
```

- **Originator**: 상태를 가진 원본 객체. 스냅샷 생성과 복원을 담당
- **Memento**: 상태의 복사본. 외부에서 내용을 수정하면 안 됨
- **Caretaker**: 메멘토를 저장하고 관리. 내용은 모르고 저장만 함

---

## 2. 기본 구현

텍스트 에디터의 Undo/Redo 기능을 구현합니다.

```kotlin
// 메멘토(스냅샷): 에디터 상태의 복사본
// data class: 불변 복사본 생성에 적합, copy()도 지원
data class EditorMemento(val content: String, val cursorPosition: Int)

// 원본(Originator): 실제 편집 기능을 담당
class TextEditor {
    var content: String = ""
    var cursorPosition: Int = 0

    // 현재 커서 위치에 텍스트 삽입
    fun type(text: String) {
        content = content.substring(0, cursorPosition) + text + content.substring(cursorPosition)
        cursorPosition += text.length  // 커서를 삽입한 텍스트 끝으로 이동
    }

    // 커서 앞의 텍스트를 count만큼 삭제 (백스페이스와 동일)
    fun delete(count: Int) {
        if (cursorPosition < count) return
        content = content.substring(0, cursorPosition - count) + content.substring(cursorPosition)
        cursorPosition -= count  // 커서를 삭제한 만큼 앞으로 이동
    }

    // 현재 상태의 스냅샷(메멘토) 생성
    fun save() = EditorMemento(content, cursorPosition)

    // 저장된 스냅샷으로 상태 복원
    fun restore(memento: EditorMemento) {
        content = memento.content
        cursorPosition = memento.cursorPosition
    }

    fun show() = println("\"$content\" (커서: $cursorPosition)")
}

// 보관자(Caretaker): Undo/Redo 스택으로 메멘토를 관리
class EditorHistory {
    private val undoStack = ArrayDeque<EditorMemento>()
    private val redoStack = ArrayDeque<EditorMemento>()

    // 작업 후 현재 상태를 저장 (redo 스택은 초기화 — 새 작업 시 redo 불가)
    fun save(editor: TextEditor) {
        undoStack.addLast(editor.save())
        redoStack.clear()  // 새 작업이 발생하면 redo 히스토리 삭제
    }

    // Undo: 가장 최근 상태를 redo 스택으로 이동하고, 그 전 상태로 복원
    fun undo(editor: TextEditor) {
        if (undoStack.size <= 1) { println("되돌릴 기록 없음"); return }
        redoStack.addLast(undoStack.removeLast())  // 현재 상태를 redo 스택으로
        editor.restore(undoStack.last())           // 이전 상태로 복원
    }

    // Redo: redo 스택에서 꺼내 다시 적용
    fun redo(editor: TextEditor) {
        if (redoStack.isEmpty()) { println("다시 실행할 기록 없음"); return }
        val memento = redoStack.removeLast()       // redo 스택에서 꺼냄
        undoStack.addLast(memento)                 // undo 스택에도 추가
        editor.restore(memento)                    // 해당 상태로 복원
    }
}

// 사용 예시
val editor = TextEditor()
val history = EditorHistory()

history.save(editor)                              // 초기 상태 저장 (빈 문서)
editor.type("Hello"); history.save(editor)
editor.show()  // "Hello" (커서: 5)

editor.type(" World"); history.save(editor)
editor.show()  // "Hello World" (커서: 11)

editor.delete(5); history.save(editor)
editor.show()  // "Hello " (커서: 6)

history.undo(editor); editor.show()  // "Hello World" 복원 (커서: 11)
history.undo(editor); editor.show()  // "Hello" 복원 (커서: 5)
history.redo(editor); editor.show()  // "Hello World" 재실행 (커서: 11)
```

> **Undo 스택 크기 주의**: `undoStack.size <= 1`로 초기 상태(빈 문서)는 undo하지 못하게 막습니다. 초기 상태 이전으로는 돌아갈 수 없습니다.

---

## 3. data class 스냅샷 패턴

Kotlin `data class`의 `copy()`를 활용하면 메멘토를 매우 간결하게 구현합니다.
게임 세이브 슬롯처럼 여러 지점을 이름으로 저장하는 예입니다.

```kotlin
// 게임 상태 — data class이므로 copy()로 불변 스냅샷 즉시 생성
data class GameState(
    val level: Int,
    val health: Int,
    val score: Int,
    val position: Pair<Int, Int>  // (x, y) 좌표
)

class Game {
    var state = GameState(level = 1, health = 100, score = 0, position = 0 to 0)
        private set  // 외부에서 직접 변경 불가

    private val saveSlots = mutableMapOf<String, GameState>()

    // 세이브: 현재 상태를 슬롯 이름으로 저장
    // copy()는 값 복사 → 이후 state가 바뀌어도 저장된 스냅샷은 불변
    fun save(slotName: String) {
        saveSlots[slotName] = state.copy()
        println("저장됨: $slotName")
    }

    // 로드: 슬롯에서 상태 복원
    fun load(slotName: String) {
        state = saveSlots[slotName] ?: run { println("슬롯 없음: $slotName"); return }
        println("불러옴: $slotName → $state")
    }

    // 게임 진행: 원하는 필드만 변경 (copy의 named parameters 활용)
    fun progress(levelUp: Int = 0, damage: Int = 0, points: Int = 0, dx: Int = 0, dy: Int = 0) {
        state = state.copy(
            level    = state.level + levelUp,
            health   = (state.health - damage).coerceAtLeast(0),  // 0 이하로 내려가지 않음
            score    = state.score + points,
            position = state.position.first + dx to state.position.second + dy
        )
    }
}

val game = Game()

game.progress(points = 500, dx = 10, dy = 5)
game.save("체크포인트1")   // 현재 상태 저장

game.progress(levelUp = 1, damage = 30, points = 1000, dx = 20)
game.save("체크포인트2")   // 또 다른 지점 저장

game.progress(damage = 80)  // 큰 피해 발생!
println("현재: ${game.state}")  // health 감소 확인

game.load("체크포인트1")    // 피해 없는 지점으로 복원
println("복원: ${game.state}")  // 체크포인트1 상태 출력
```

---

## 4. 증분 스냅샷 (변경분만 저장)

상태가 크고 변경이 적다면, 전체를 복사하는 대신 **변경분(diff)만 저장**해 메모리를 절약합니다.

```kotlin
// 변경 이력을 sealed class로 표현
sealed class Change {
    // 어느 위치(position)에 무슨 텍스트(text)를 삽입했는지 기록
    data class TextInserted(val position: Int, val text: String) : Change()
    // 어느 위치에서 몇 글자(count)를 삭제했는지 기록
    data class TextDeleted(val position: Int, val count: Int) : Change()
}

class EfficientEditor {
    private var content = ""
    // 전체 스냅샷 대신 변경 이력만 저장
    private val changes = ArrayDeque<Change>()

    fun insert(pos: Int, text: String) {
        content = content.substring(0, pos) + text + content.substring(pos)
        changes.addLast(Change.TextInserted(pos, text))  // 삽입 이력 기록
    }

    fun delete(pos: Int, count: Int) {
        changes.addLast(Change.TextDeleted(pos, count))  // 삭제 이력 기록 (삭제 전에!)
        content = content.substring(0, pos) + content.substring(pos + count)
    }

    // 마지막 변경을 되돌림 (변경 이력의 역연산)
    fun undo() {
        val change = changes.removeLastOrNull() ?: return
        when (change) {
            // 삽입을 undo → 삽입했던 만큼 삭제
            is Change.TextInserted ->
                content = content.substring(0, change.position) +
                    content.substring(change.position + change.text.length)
            // 삭제를 undo → 삭제했던 위치에 다시 삽입
            // (실제로는 삭제 전 내용도 Change에 저장해야 복원 가능)
            is Change.TextDeleted  ->
                content = content.substring(0, change.position) +
                    "?".repeat(change.count) + content.substring(change.position)
        }
    }

    fun getContent() = content
}
```

> **증분 방식의 장점**: 긴 문서의 작은 수정이 많을 때 전체 스냅샷 방식보다 메모리를 크게 절약합니다. Git의 커밋도 전체 복사가 아니라 변경분(diff)을 저장하는 동일한 원리입니다.

---

## 5. 정리

| 방식 | 특징 | 적합한 상황 |
|------|------|------------|
| 전체 스냅샷 | 구현 단순, 메모리 사용량 큼 | 상태가 작거나 저장 횟수가 적을 때 |
| data class copy() | Kotlin 관용, 한 줄 구현 | 불변 상태 + 코틀린 프로젝트 |
| 증분(diff) 방식 | 메모리 효율적 | 상태가 크고 변경이 적을 때 |

- **Originator**: 상태 생성·복원, **Memento**: 상태 복사본(읽기 전용), **Caretaker**: 스택 관리
- Undo/Redo 구현: undo 스택 + redo 스택, undo 시 redo 스택으로 이동
- Kotlin `data class copy()`: 불변 스냅샷을 한 줄로 생성하는 가장 실용적인 방법
- 실전 활용: 텍스트 에디터 Undo/Redo, 게임 세이브 포인트, 폼 임시 저장
