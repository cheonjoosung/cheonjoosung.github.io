---
title: 안드로이드 설치된 앱 목록 아이콘 가져오는 방법
tags: [Android]
style: fill
color: dark
description: Android 앱에서 현재 기기에 설치된 다른 앱의 패키지 목록을 확인하는 방법을 소개합니다. Android how to get a list of installed andoird appllications
---

## ✨ 개요

Android 앱을 개발하다 보면 사용자 기기에 설치된 **다른 앱의 패키지 목록을 확인**해야 할 때가 있습니다.  
예를 들어, 공유 가능한 앱 목록을 필터링하거나 특정 앱이 설치되어 있는지 확인해야 할 경우입니다.

이 글에서는 Android에서 설치된 앱 목록을 가져오는 방법과 주의사항을 다룹니다.

---

## 1. ✅ 설치된 앱 목록 가져오기

- PackagerManager 를 통한 앱 목록 조회
- 리스트 뷰에 앱 정보(패키지명 및 아이콘) 표시 

---

## 2. ✅ 설치된 앱 목록 가져오기 코드

### 2.1 AndromdManifest.xml 설정
Android 11 이상 <queires> 명시해야 함

```xml
<manifest>
    <queries>
        <intent>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent>
    </queries>
</manifest>

<!-- Android 11 이상에서 모든 앱 정보를 보려면 필요 -->
<uses-permission android:name="android.permission.QUERY_ALL_PACKAGES"
                 tools:ignore="QueryAllPackagesPermission" />
```
- ❌ Google Play에서는 불필요한 <queries> 사용을 제한합니다. 꼭 필요한 앱만 명시해야 합니다.
- 테스트를 위해서 2개의 방법 중 편한 방법 사용
- 해당 권한은 플레이스토어 오픈 시 구글 승인이 필요함 

### 2.2 레이아웃 코드

- activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   xmlns:tools="http://schemas.android.com/tools"
                                                   android:id="@+id/main"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="match_parent"
                                                   tools:context=".MainActivity">

    <Button
            android:id="@+id/getAppListButton"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:text="앱 목록 조회"
            android:textSize="20sp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

    <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/appListRecyclerView"
            android:layout_width="0dp"
            android:layout_height="0dp"
            android:layout_margin="16dp"
            app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@id/getAppListButton" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

- item_app.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:padding="8dp">

    <ImageView
        android:id="@+id/imageViewIcon"
        android:layout_width="48dp"
        android:layout_height="48dp"
        android:layout_marginEnd="8dp"/>

    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:orientation="vertical">

        <TextView
            android:id="@+id/textViewName"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textStyle="bold"
            android:textSize="16sp" />

        <TextView
            android:id="@+id/textViewPackage"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textSize="12sp"
            android:textColor="#888888" />
    </LinearLayout>
</LinearLayout>
```


### 2.2 Recycler View 코드

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView
import com.example.sample.databinding.ItemAppBinding

class AppAdapter(private val appList: List<AppInfo>) :
    RecyclerView.Adapter<AppAdapter.AppViewHolder>() {

    inner class AppViewHolder(val binding: ItemAppBinding) : RecyclerView.ViewHolder(binding.root) {
        fun bind(app: AppInfo) {
            binding.imageViewIcon.setImageDrawable(app.icon)
            binding.textViewName.text = app.appName
            binding.textViewPackage.text = app.packageName
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): AppViewHolder {
        val view = ItemAppBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return AppViewHolder(view)
    }

    override fun onBindViewHolder(holder: AppViewHolder, position: Int) {
        val app = appList[position]
        holder.bind(app)
    }

    override fun getItemCount(): Int = appList.size
}
```

### 2.3 안드로이드 설치된 앱 목록 조회 코드

```kotlin
import android.content.pm.PackageManager
import android.graphics.drawable.Drawable
import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import com.example.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = MainActivity::class.java.simpleName

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        with(binding) {
            getAppListButton.setOnClickListener {
                val appList = getInstalledAppPackages()

                val adapter = AppAdapter(appList)
                appListRecyclerView.adapter = adapter
            }
        }
    }

    /**
     * 설치된 앱 목록 전체 가져오기
     */
    private fun getInstalledAppPackages(): List<AppInfo> {
        val pm = packageManager
        val apps = pm.getInstalledApplications(PackageManager.GET_META_DATA)

        return apps
            .filter { pm.getLaunchIntentForPackage(it.packageName) != null } // 런처에서 실행 가능한 앱만, 제거시 시스템앱 포함한 목록 조회 가능
            .map {
                AppInfo(
                    appName = pm.getApplicationLabel(it).toString(),
                    packageName = it.packageName,
                    icon = pm.getApplicationIcon(it)
                )
            }
    }

    /**
     * 특정 앱이 설치된 여부
     */
    fun isAppInstalled(packageName: String): Boolean {
        return try {
            packageManager.getPackageInfo(packageName, 0)
            true
        } catch (e: PackageManager.NameNotFoundException) {
            false
        }
    }

}

data class AppInfo(
    val appName: String,
    val packageName: String,
    val icon: Drawable
)
```
- App 정보를 담을 data class AppInfo 필요
- isAppInstalled() 를 통해 특정 배 설치 여부 체크 가능

---

## 3.🧠 **결론**

- Android 11 이상에서는 패키지 접근 제한이 생겼기 때문에 <queries> 설정은 필수입니다.
- 민감한 정보 접근처럼 보여질 수 있으니, 실제 배포 시에는 불필요한 앱 탐색은 피하는 것이 좋습니다.