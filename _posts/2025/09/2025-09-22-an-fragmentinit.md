---
title: 안드로이드 Fragment not initialized 해결 방법 commit, runOnCommit, commitNow 
tags: [Android]
style: fill
color: dark
description: 안드로이드 Fragment Fragment not initialized , Fragment is not attached to a context , binding is null 해결 방법 관한 포스팅
---

## ✨ 개요

```bash
java.lang.IllegalStateException: Fragment SomeFragment not initialized

# or

lateinit property binding has not been initialized
```

이러한 에러는 대부분 Fragment의 뷰가 완전히 생성되기 전에 무언가를 호출할 때 발생합니다.
이번 포스팅에서는 이 에러가 발생하는 원인, commit 계열의 처리 시점, runOnCommit 사용법까지 정리해보겠습니다.

---

## 1. 문제 발생 상황 예시

```kotlin
val fragment = SomeFragment()
supportFragmentManager.beginTransaction()
    .replace(R.id.container, fragment)
    .commit()

fragment.doSomething() // ❌ Exception
```
- 정상적으로 실행될 것 같지만 Exception 이 발생할 확률이 높습니다
- commit()은 비동기적으로 실행되므로, doSomething() 호출 시점에 아직 Fragment는 초기화되지 않았을 수 있습니다.

---

## 2. 해결 방법

#### 방법 1: `commitNow()` 사용

```kotlin
supportFragmentManager.beginTransaction()
    .replace(R.id.container, fragment)
    .commitNow() // 즉시 commit + attach
    
fragment.doSomething() // 안전하게 접근 가능
``` 
- ✅ commitNow()는 즉시 실행되기 때문에 다음 줄에서 Fragment를 안전하게 사용할 수 있음


#### 방법 2: `FragmentManager.runonCommit {}` 사용

```kotlin
val fragment = SomeFragment()
supportFragmentManager.beginTransaction()
    .replace(R.id.container, fragment)
    .runOnCommit {
        fragment.doSomething() // Fragment가 attach된 이후 안전하게 실행
    }
    .commit()
``` 
- ✔ Fragment attach 이후 실행 보장
- ✔ commit() 사용 시도 유지 가능
- ✔ View 초기화 후 안전하게 로직 실행 가능



#### 방법 3: Fragment 내부에서 lifecycle-aware 처리

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    // 뷰 초기화 완료 후 안전한 작업 수행
}

viewLifecycleOwner.lifecycleScope.launchWhenStarted {
    // View가 초기화된 이후 동작
}
``` 
- ViewBinding이 onCreateView() 이전에 호출되면 binding is null 예외 발생 가능

---

## 3. Fragment 초기화 관련 에러 종류 정리

| 에러 메시지                                  | 원인                                            |
| --------------------------------------- | --------------------------------------------- |
| `Fragment not initialized`              | 트랜잭션이 완료되기 전에 접근                              |
| `binding is null`                       | `onCreateView()` 또는 `onViewCreated()` 이전 접근   |
| `Fragment is not attached to a context` | `getContext()`, `requireActivity()` 등이 조기 호출됨 |

---

## 4. 결론

| 요약 핵심 포인트                                            |
| ---------------------------------------------------- |
| `commit()`은 비동기 → 트랜잭션 직후 접근하면 Fragment 초기화 안됐을 수 있음 |
| `commitNow()` 또는 `runOnCommit {}` 사용으로 즉시 처리 가능      |
| Fragment 내부에서는 `onViewCreated()` 이후를 활용              |
| `binding is null` 오류도 대부분 뷰 초기화 타이밍 문제               |