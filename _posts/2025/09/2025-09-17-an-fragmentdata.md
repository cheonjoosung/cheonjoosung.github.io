---
title: Android Fragment Result API 사용법 정리 - Fragment 간 데이터 전달의 가장 깔끔한 방법
tags: [Android]
style: fill
color: dark
description: Android Fragment Result API 사용법 정리 - Fragment 간 데이터 전달의 가장 깔끔한 방법
---
---

## ✨ 개요

안드로이드에서 **Fragment 간 데이터 전달**은 예전부터 여러 방식이 존재했지만,  
최근에는 `FragmentResult API`가 Jetpack에서 공식으로 제공되면서 더 깔끔하고 안전한 방식이 등장했습니다.

이 포스팅에서는 `FragmentResult API`의 개념부터 사용법, 주의사항까지 정리합니다.

---

## 1. FragmentResult API란?

- Jetpack의 `androidx.fragment` 라이브러리에서 제공
- Fragment 간 데이터를 주고받을 수 있는 공식 API
- Fragment → Fragment 또는 Fragment → DialogFragment 등 다양한 통신 가능
- 중간에 Activity를 거치지 않고, FragmentManager를 통해 통신 처리

> 📦 AndroidX `fragment-ktx`가 포함된 프로젝트에서 사용 가능  
> ✅ ViewModel 없이도, Interface 없이도 깔끔하게 단방향 전달 가능!

---

## 2. 사용 예제

### 1. AFragment -> BFragment 로 데이터 전달
- BFragment 수신자
  + ```kotlin
    class BFragment : Fragment() {

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            setFragmentResultListener("userKey") { requestKey, bundle ->
                val username = bundle.getString("username")
                val age = bundle.getInt("age")
                Log.d("BFragment", "받은 데이터: $username $age")
            }
        }
    }
    ``` 
- AFragment 송신자
    + ```kotlin
      val bundle = Bundle().apply {
         putString("username", "Joo")
         putInt("age", 14)
      }
      setFragmentResult("userKey", bundle)

      // 이후 BFragment로 이동
      parentFragmentManager.beginTransaction()
        .replace(R.id.container, BFragment())
        .addToBackStack(null)
        .commit()
      ```

---

## 3. FragmentResult API 특징

| 특징           | 설명                         |
| ------------ | -------------------------- |
| ✅ 단방향 데이터 전달 | A → B, Dialog → Fragment 등 |
| ✅ 의존성 없음     | 인터페이스나 ViewModel 없이 동작     |
| ✅ 생명주기 안전    | `onCreate()` 안에서 수신자 등록 가능 |
| ✅ key 기반 분기  | 여러 Fragment에서 사용 가능        |

---

## 4. 사용 시 주의할 점

- setFragmentResultListener()는 onCreate() 또는 onViewCreated() 에서 호출해야 안전
- requestKey와 setFragmentResult()의 key가 정확히 일치해야 함
- 한 번 수신되면 자동으로 해제됨 → 반복적으로 수신할 경우 재등록 필요

--- 

## 5. 언제 쓰면 좋을까?

| 상황                              | FragmentResult API 적합 여부 |
| ------------------------------- | ------------------------ |
| A → B로 간단한 값 전달                 | ✅ 매우 적합                  |
| DialogFragment → Fragment로 값 전달 | ✅ 권장                     |
| Fragment ↔ Fragment 양방향 통신      | ❌ ViewModel이 더 적합        |
| 다수 Fragment에서 수신해야 함            | ✅ key로 구분 가능             |

---

## 6.  결론

- FragmentResult API는 Fragment 간 데이터를 전달할 때 가장 간결하고 안전한 방법입니다.
  + Interface 콜백 없이도 깔끔하게 전달 가능
  + Activity를 거치지 않아서 코드가 간단해짐
  + Lifecycle-safe하며 Jetpack 공식 방식
- 📌 단방향 통신이거나 Dialog → Fragment 데이터 전달 상황이라면
- 👉 FragmentResult API가 최고의 선택입니다!