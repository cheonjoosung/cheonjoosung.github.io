---
title: Android Fragment 데이터 전달 safeargs 활용 (코틀린)
tags: [Android, Kotlin]
style: fill
color: dark
description: Android Fragment 간 데이터 전달하기 args 
---

탭 구조에서 프래그먼트 간 이동이 발생할 때 탭도 이동하고 데이터도 전달해야 하는 경우가 있습니다.
safeargs 를 통해 데이터를 전달하고 받는 방법입니다.

### 1. Gradle 설정
```kotlin
plugins {
    /** 생략 **/
    id 'androidx.navigation.safeargs.kotlin' version '2.7.7' apply false //추가하기
}
```
- Project 수준의 gradle 파일에 위의 코드를 추가해주세요. <br><br>

```kotlin
plugins { 
    id 'androidx.navigation.safeargs'  // 추가하기
}
```
- App 수준의 gradle 파일 상단에 위의 코드를 추가해주세요.

### 2. navigation.xml safeArgs & action 설정
```kotlin
    // 데이터 전달할 곳
    <fragment
    android:id="@+id/navigation_list"
    android:name="com.js.view_job.ui.list.ListFragment"
    android:label="@string/title_list">
    
        <action
        android:id="@+id/goToViewFragment"
        app:destination="@id/navigation_view"
        app:launchSingleTop="true">
        
            <argument
            android:name="argsUrl"
            app:argType="string" />
        </action>
    </fragment>
        
    // 데이터 받을 곳    
    <fragment
        android:id="@+id/navigation_view"
        android:name="com.js.view_job.ui.view.ViewFragment"
        android:label="@string/title_view" >
        
        <argument
        android:name="argsUrl"
        android:defaultValue=""
        app:argType="string" />
    </fragment>
```
- action
  + fragment 간 이동을 위한 액션 
  + id 이 액션을 접근하기 위한 값이고 destination 이동할 fragment id 값
- argument
  + action 이동 시 전달한 인자 값 설정
  + 데이터를 받는 fragment 에도 argument 를 선언해줘야 함 

### 3. Fragment 에서 데이터를 전달하고 받는 코드 
```kotlin
// 데이터 전달할 곳
val companyUrl = jobSite.companyUrl ?: ""
val action = ListFragmentDirections.goToViewFragment(companyUrl)
findNavController().navigate(action)


// 데이터 받을 곳
private val args: ViewFragmentArgs by navArgs()
val companyUrl = args.argsUrl
```
- navigation 에 액션을 설정하면 {Fragment이름}Directions.{action_ID_값} 자동으로 빌드되어 사용 가능해짐 
- navigation 에 설정한 argument id 를 키로 값을 꺼내옴

## 결론
- 공통 viewModel 을 만들고 이를 상속받아 전달하는 방법도 가능할 것 같음
