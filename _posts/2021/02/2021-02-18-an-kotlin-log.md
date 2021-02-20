---
title: (안드로이드/코틀린) 로그 릴리즈/디버그 모드 나누기
tags: [Android, Kotlin]
style: fill
color: dark
description: 안드로이드 버전을 출시할 때 로그를 하나씩 지우는게 아니라 디버그 모드에서만 작동되도록 만들기
---

## 안드로이드 로그 릴리즈/디버그 모드 나눠서 찍기
안드로이드 개발을 하다보면 로그를 찍어야 값 확인도 가능하고 디버그하는데도 편리하다. 디버그 모드에서 Log.d()를 찍어놓고 릴리즈 버전 때 하나씩 지우려면 상당히 많은 노력이 수반된다. 디버그 모드를 체크하는 로직을 통해 로그를 찍는 것이 코드 관리 및 성능에도 좋다


## 실습하기
**1단계 Utils() 라는 별도의 클래스 만든다**
- 모바일 뿐만 아니라 다른 파트의 많은 개발자들이 Utils 클래스를 만들고 그 안에 자주 쓰이는 코드들을 넣고 다니며 프로젝트 개발 속도를 늘린다.

```javascript
import android.content.Context
import android.content.pm.ApplicationInfo
import android.util.Log


class Utils(context: Context) {
    private var TAG = "TEST"
    private val isDEBUG =
            (context.packageManager.getApplicationInfo(context.packageName, 0).flags
                    and ApplicationInfo.FLAG_DEBUGGABLE) != 0

    fun logD(message: String) {
        if (isDEBUG) {
            Log.d(TAG, "D MODE $message")
        }
    }

    fun logE(message: String) {
        if (isDEBUG) {
            Log.e(TAG, "E MODE $message")
        }
    }

    fun logI(message: String) {
        if (isDEBUG) {
            Log.i(TAG, "I MODE $message")
        }
    }

    fun logW(message: String) {
        if (isDEBUG) {
            Log.w(TAG, "W MODE $message")
        }
    }

    fun logV(message: String) {
        if (isDEBUG) {
            Log.v(TAG, "V MODE $message")
        }
    }
}
```
- isDEBUG 는 현 프로젝트의 정보가 디버그 모드인지 판단하는 로직으로 TRUE OR FALSE 값을 가진다
- 잘 안쓰는 로그 모드는 취향에 따라 제거해도 된다

**2. 액티비티나 프래그먼트에서 사용**
- 멤버 변수로 선언하고 사용하려면 lazy 키워드 활용

```javascript
class MainActivity : AppCompatActivity() {
    private val utils: Utils by lazy {
        Utils(this)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val navView: BottomNavigationView = findViewById(R.id.nav_view)

        val navController = findNavController(R.id.nav_host_fragment)
        // Passing each menu ID as a set of Ids because each
        // menu should be considered as top level destinations.
        val appBarConfiguration = AppBarConfiguration(setOf(
                R.id.navigation_home, R.id.navigation_dashboard, R.id.navigation_notifications))
        setupActionBarWithNavController(navController, appBarConfiguration)
        navView.setupWithNavController(navController)


        var str = "Kotlin"
        utils.logD("HI $str")
        utils.logI("HI $str")
        utils.logV("HI $str")
        utils.logE("HI $str")
        utils.logW("HI $str")
    }
}
```

## 결론
- Log.d("TAG", "MSG") 이 방식을 작성하는데 생각보다 손이 너무 많이 가므로 Utils클래스를 통해 효율적으로 코딩하는 것을 추천
- **BuildConfig를 활용하여 릴리즈/디버그 모드를 나누는 방법도 존재함**
