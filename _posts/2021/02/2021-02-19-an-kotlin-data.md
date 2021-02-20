---
title: (안드로이드/코틀린) Activity 와 Fragment 사이에서 데이터 전달 및 통신방법
tags: [Android, Kotlin]
style: fill
color: dark
description: 안드로이드 코틀린에서 액티비티와 프래그먼트 사이에서 데이터 전달 및 통신방법
---

## Activity to Fragment 데이터 전달 및 통신방법
많은 모바일 서비스들이 화면들이 많아지면서 액티비티 위에 Bottom Nav를 그리고 프래그먼트를 사용하는 경우가 많아졌다. 이렇게 되면서 데이터나 버튼 액션을 가끔씩 전달해줘야 하는 경우가 생긴다.


## 실습하기
**1. Fragment 코드 만들기**
- 모바일 뿐만 아니라 다른 파트의 많은 개발자들이 Utils 클래스를 만들고 그 안에 자주 쓰이는 코드들을 넣고 다니며 프로젝트 개발 속도를 늘린다.

```javascript
class DashboardFragment : Fragment() {

    override fun onCreateView(inflater: LayoutInflater,container: ViewGroup?,savedInstanceState: Bundle?
    ): View? {
        val root = inflater.inflate(R.layout.fragment_dashboard, container, false)
        return root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
       super.onViewCreated(view, savedInstanceState)

       listener?.onFragmentListener("HI")
   }


    private var listener: OnFragmentListener? = null

    interface OnFragmentListener {
        fun onFragmentListener(msg: String)
    }

    override fun onAttach(context: Context) {
        super.onAttach(context)

        if (context is OnFragmentListener) {
            listener = context
        } else {
            throw RuntimeException("$context must implement OnFragmentInteractionListener")
        }
    }
}
```

- 인터페이스를 OnFragmentListener 를 만든 후에 리스너를 하나 선언해줍니다.
- onAttach 메소드를 오버라이딩 한 후 액티비티에서 프래그먼트를 호출 할때 해당 컴포넌트를 연결해줍니다.

**2. Activity 코드 작성하기**

```javascript
class MainActivity : AppCompatActivity(), DashboardFragment.OnnFragmentListener {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }

    override fun onFragmentListener(msg: String) {
        Toast.makeText(this, "msg from fragment", Toast.LENGTH_SHORT).show()
    }
}
```
- DashboardFragment의 프래그먼트 리스너를 implement 해주시고
- onFragmentListerner 메소드를 오버라이딩 하여 구현하면 끝

## 결론
- **리스너&인터페이스 -> onAttach -> 액티비티에서 구현 -> 끝**
