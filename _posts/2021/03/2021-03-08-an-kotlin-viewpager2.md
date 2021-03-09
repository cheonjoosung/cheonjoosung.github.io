---
title: (안드로이드/코틀린) 뷰페이저 + 카드뷰 만들기 - ViewPager2 + CardView
tags: [Android, Kotlin]
style: fill
color: dark
description: 안드로이드 코틀린에서 뷰페이저 + 카드뷰 만들기 - ViewPager2 + CardView
---

## 안드로이드 ViewPager2 + CardView 만들기
안드로이드에서 카드뷰와 뷰페이저2를 활용하여 예쁜 디자인을 구현해봅시다. 목표는
![preview](https://i.ytimg.com/vi/UsXv6VRqZKs/maxresdefault.jpg)


## 실습하기
**1. Gradle 종속성 추가**

```javascript
//app gradle
dependencies {
    //생략
    // - - Statistics for walk
    implementation 'androidx.cardview:cardview:1.0.0'
    implementation "androidx.viewpager2:viewpager2:1.0.0"
}

```

- Gradle APP 과 Gradle Project 에 코드를 추가

**2. 레이아웃 추가**

```javascript

//activity_main.xml
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <androidx.viewpager2.widget.ViewPager2
        android:id="@+id/view_pager2"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:clipChildren="false"
        android:clipToPadding="false"
        android:foregroundGravity="center"
        android:overScrollMode="never"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>

//item.xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"

    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:layout_gravity="center_vertical"
    android:gravity="center_vertical"
    android:orientation="vertical">

    <androidx.cardview.widget.CardView
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <TextView
            android:id="@+id/title"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Title" />

    </androidx.cardview.widget.CardView>

</LinearLayout>
```
- activit_main.xml 과 item.xml (카드뷰)


**3. theme.xml 추가 & manifest.xml**
```javascript
<resources xmlns:tools="http://schemas.android.com/tools">
    //생략

    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <item name="android:windowDrawsSystemBarBackgrounds" tools:ignore="NewApi">true</item>
        <item name="android:statusBarColor" tools:ignore="NewApi">@android:color/transparent</item>
        <item name="android:windowTranslucentStatus">true</item>
        <item name="windowActionBar">false</item>
        <item name="windowNoTitle">true</item>
    </style>

</resources>

<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
```
- android:theme="@style/AppTheme"> 추가
-


**3. Activity 코드 및 Adapter 코드**
```javascript
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val viewPager = findViewById<ViewPager2>(R.id.view_pager2)

        var models: MutableList<String> = mutableListOf()

        models.add("AAAAAA")
        models.add("BBBBBB")
        models.add("CCCCCC")
        models.add("DDDDDD")

        var adapter = Adapter(models, this)
        viewPager.adapter = adapter
        viewPager.setPadding(30, 0, 30, 0)

        viewPager.registerOnPageChangeCallback(object : ViewPager2.OnPageChangeCallback() {
            override fun onPageScrolled(position: Int, positionOffset: Float, positionOffsetPixels: Int) {
               super.onPageScrolled(position, positionOffset, positionOffsetPixels)
            }

            override fun onPageSelected(position: Int) {
                super.onPageSelected(position)
            }

            override fun onPageScrollStateChanged(state: Int) {
                super.onPageScrollStateChanged(state)
            }
        })


    }
}

class Adapter (
    var models: List<String>,
    var context: Context
) : RecyclerView.Adapter<Adapter.AdapterViewHolder>() {

    override fun getItemCount(): Int {
        return models?.size ?: 0
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): AdapterViewHolder {
        val v = LayoutInflater.from(parent.context).inflate(R.layout.item, parent, false)
        return AdapterViewHolder(v)
    }

    override fun onBindViewHolder(holder: AdapterViewHolder, position: Int) {
        val item = models[position]
        holder.title.text = item
    }

    inner class AdapterViewHolder(view: View) : RecyclerView.ViewHolder(view) {
        val title: TextView = view.findViewById(R.id.title)
    }
}
```
- ViewPager2가 되면서 어댑터를 RecylerView를 활용하여 뷰의 자유도를 높였습니다. 해당 샘플 코드에서는 텍스트뷰만 넣어지만 item.xml 에 다양한 뷰를 추가하여 꾸밀 수 있습니다.
- 샘플 영상은 아래 유투브를 참조하시고 해당 코드는 자마 기반으로 되어있기에 코틀린과 다소 다를 수 있습니다.
<iframe width="560" height="315" src="https://www.youtube.com/embed/UsXv6VRqZKs" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## 결론
- **복 붙**
