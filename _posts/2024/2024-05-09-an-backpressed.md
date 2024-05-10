---
title: Android onBackPressedDispatcher 방법. onBackPressed deprecated (코틀린)
tags: [Android, Kotlin]
style: fill
color: dark
description: Android onBackPressedDispatcher 방법 onBackPressed deprecated (코틀린) 
---

onBackPressed deprecated 되면서 onBackPressedDispatcher 를 구현해서 백키 로직을 구현해야 한다.

###  백키
```kotlin
override fun onCreateView(
  inflater: LayoutInflater,
  container: ViewGroup?,
  savedInstanceState: Bundle?,
): View {
    requireActivity().onBackPressedDispatcher.addCallback(viewLifecycleOwner) {
      // bottom nav 사용하는 경우
      findNavController().popBackStack()
    
     // 또는 기타 다른 코드
  }
}

// onAttach 로 사용 시
override fun onAttach(context: Context) {
  super.onAttach(context)
  val callback = object : OnBackPressedCallback(true) { // default to enabled) 
    override fun handleOnBackPressed() {
      // your code when back key
    }
  }
  requireActivity().onBackPressedDispatcher.addCallback(this, callback) //// LifecycleOwner
}
```
- onCreatedView 에서 onBackPressedDispatcher callback 을 처리해주면 됨
- bottom nav 를 쓰는 경우 팝업 스택을 콜해서 처리해도 되고 다른 로직을 원하면 작성 가능


## 결론
- onBackPressed 그립다
