---
title: Android License View (라이센스 뷰 만들기)
tags: [Android]
style: fill
color: dark
description: Android License View (라이센스 뷰 만들기)
---

## 설명
오픈소스를 사용하면 라이센스 정책에 따라 고지를 해야 한다. 유튜브나 카카오톡의 기타 앱들을 보면 오픈소스 라이센스 사용 목록을 고지한다.

여러 방법이 있지만 라이센스 뷰를 쉽게 만드는 것은 깃헙과 웹뷰를 활용하는 방법이다.


**깃헙에 라이센스 파일 업로드**
깃헙에서 업로드한 파일 클릭
![preview](https://github.com/cheonjoosung/cheonjoosung/blob/master/image/license3.png?raw=true)

raw 버튼 클릭 후 상단의 url 복사
![preview](https://github.com/cheonjoosung/cheonjoosung/blob/master/image/license2.png?raw=true)



**라이센스 웹뷰 화면 만들기**
```javascript
class LicenseWebViewActivity : AppCompatActivity() {

    private lateinit var binding: ActivityLicenseWebViewBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityLicenseWebViewBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val url = "your_license_file"
        binding.webView.loadUrl(url)
    }
}
```
url에 깃헙에 업로드한 라이센스 파일 경로를 입력하면 끝

```javascript
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".LicenseWebViewActivity">

    <WebView
        android:id="@+id/web_view"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

결과
![preview](https://github.com/cheonjoosung/cheonjoosung/blob/master/image/license1.jpeg?raw=true)
