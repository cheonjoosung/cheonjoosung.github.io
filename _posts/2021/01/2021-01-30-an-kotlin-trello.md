---
title: Android 트렐로 API 연동하는 방법 - Trello REST API
tags: [Android, Kotlin]
style: fill
color: dark
description: 안드로이드에서 Trello(트렐로) API 를 활용하여 보드/리스트/카드를 접근하고 활용하는 방법
---

## 트렐로 API로 리스트, 카드 만들기
트렐로는 프로젝트를 보드로 정리하는 협업 툴입니다. 프로젝트 뿐만 아니라 개인의 학습을 정리하기에도 유용하고 To-DO List로 많이 활용되고 있습니다. UI도 상당히 직관적이라 사용법을 익히지 않아도 누구든지 쉽게 활용이 가능합니다. 웹, 앱이 존재하지만 API를 활용하여 효과적으로 활용할 수도 있습니다.


**1단계 트렐로 회원가입 및 보드 생성하기**
- 트렐로의 보드, 리스트, 카드 등은 고유의 ID가 존재하고 이를 통해 API 호출을 합니다.
- [트렐로](https://trello.com/) 로그인 이후 보드를 생성하면 "https://trello.com/b/{id}/{board_title}" 로 생성이 됩니다.
- id 를 메모를 해주세요


**2단계 트렐로 API Key & Token 생성하기**
- 트렐로의 API 를 사용하기 위해서 Key & Token 이 필요하다
- [API키 만들기](https://trello.com/app-key) 에서 키/토큰을 만들고 메모를 해주세요


**3단계 안드로이드 코드 작성을 위한 세팅**
- 3가지 기능으로 보드 ID가져오기, 리스트 만들기, 카드 만들기 를 구현할 겁니다.
- 먼저 AndroidManiFest.xml 에 인터넷 권한을 추가해주세요

```javascript
<uses-permission android:name="android.permission.INTERNET"/>
```

- Gradle 에 아래의 코드를 추가해주세요

```javascript
apply plugin: 'com.android.application'

android {
    packagingOptions {
        exclude 'META-INF/DEPENDENCIES'
    }
}

dependencies {
    //생략
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.3.9'
    implementation 'com.mashape.unirest:unirest-java:1.3.27'
}
```
- 코루틴은 코틀린의 비동기 작업을 위한 라이브러리 입니다.
- unirest 는 HttpClicent 라이브러리 입니다.

**4단계 간단한 레이아웃 만들기**

```javascript
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/textView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="144dp"
        android:text="Trello Test"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/btn_board_id"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="29dp"
        android:text="Get Board ID"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/textView" />

    <Button
        android:id="@+id/btn_make_card"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:text="Make Card"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/btn_make_list" />

    <Button
        android:id="@+id/btn_make_list"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:text="Get List ID"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_bias="0.498"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/btn_board_id" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
- 트렐로 API 호출하기 위한 3개의 버튼

**5단계 코틀린 코드 작성**

```javascript
import android.os.Bundle
import android.util.Log
import android.widget.Button
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import com.mashape.unirest.http.HttpResponse
import com.mashape.unirest.http.JsonNode
import com.mashape.unirest.http.Unirest
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.launch

class MainActivity : AppCompatActivity() {
    var id = "1단계에서 메모한 ID"
    var boardID = ""
    var listID = ""

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val key = "2단계에서 메모한 KEY값"
        val token = "2단계에서 메모한 토큰값"

        val btnBoardID = findViewById<Button>(R.id.btn_board_id)
        val btnMakeList = findViewById<Button>(R.id.btn_make_list)
        val btnMakeCard = findViewById<Button>(R.id.btn_make_card)

        btnBoardID.setOnClickListener {
            GlobalScope.launch(Dispatchers.IO) {
                getBoardID(id, key, token)
            }
        }

        btnMakeList.setOnClickListener {
            if (boardID?.length > 0) {
                GlobalScope.launch(Dispatchers.IO) {
                    makeList("List Make Test", key, token)
                }
            }
        }

        btnMakeCard.setOnClickListener {
            if (boardID?.length > 0 && listID?.length > 0) {
                GlobalScope.launch(Dispatchers.IO) {
                    makeCard("Card Test", key, token)
                }
            }
        }

    }

    private fun getBoardID(id: String, key: String, token: String) {
        //https://trello.com/b/kIG7bBk4/kotlin-upload 트렐러 보드를 생성하면 b/뒤에 있는 부분이 ID값
        val response: HttpResponse<JsonNode> =
                Unirest.get("https://api.trello.com/1/boards/${id}")
                        .header("Accept", "application/json")
                        .queryString("key", key)
                        .queryString(
                                "token",
                                token
                        )
                        .asJson()
        Log.e("Trello", response.body.toString())
        boardID = response?.body?.`object`?.getString("id") ?: ""
    }

    private fun makeList(name: String, key: String, token: String) {
        val response = Unirest.post("https://api.trello.com/1/boards/${boardID}/lists")
                .queryString("key", key)
                .queryString("token", token)
                .queryString("name", name)
                .asJson() //asString() 기존

        listID = response?.body?.`object`?.getString("id") ?: ""
        Log.e("Trello", response.body.toString())
    }

    private fun makeCard(text: String, key: String, token: String) {
        val response = Unirest.post("https://api.trello.com/1/cards")
                .queryString("key", key)
                .queryString(
                        "token",
                        token
                )
                .queryString("idList", listID)
                .queryString("name", text)
                .asString()

        Log.e("Trello", response.body.toString())
    }
}

```

- [트렐로 REST API](https://developer.atlassian.com/cloud/trello/rest/api-group-boards/) 에 위의 코드 3가지 이외에도 다양한 코드가 있으므로 필요에 따라 바꾸면 됩니다.
- [코루틴 활용법](https://developer.android.com/kotlin/coroutines?hl=ko) 에서 정확한 문법을 확인할 수 있습니다. 위에서 구현한 코드는 자바에서 Thread().run 과 유사하면서 확실히 쉽게 비동기 코드를 구현할 수 있습니다.
- ? 는 해당 객체가 널일경우 다음 프로퍼티를 가져오지 않고 Null을 반환하는 코틀린 문법입니다. ?: 은 앞에서 널이 발생한 경우 뒤에 있는 값으로 해당 변수에 할당하는 키워드 입니다.

**6단계 실행**
- 보드 ID 가져오기 -> 리스트 만들기 -> 카드 만들기 순서대로 눌려야 정상적인 실행이 가능합니다.


## 결론
- **실제로 서비스를 구현한다면 키/토큰 값이 노출이 되므로 서버 코딩에서 작성해야 합니다. 안드로이드에서는 개인적인 활용을 위해서만 하는것이 좋습니다**
