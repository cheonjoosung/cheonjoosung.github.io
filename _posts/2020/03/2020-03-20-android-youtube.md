---
title: 안드로이드 Youtube API 연동하기 + youtube 샘플 앱
tags: [Android]
style: fill
color: dark
description: 안드로이드에서 youtube API 연동하여 동영상 시청하는 샘플 코드 
---

## Youtube API 연동하기

**1단계 안드로이드 샘플 프로젝트 만들기**
- 안드로이드 스튜디오를 실행시킨 후에 Empty 테마를 사용하여 새로운 프로젝트를 만듭니다.
- 그 이후 인터넷 권한 사용을 위해 androidManifest.xml 파일에 아래의 코드를 추가해주세요.

```javascript
<uses-permission android:name="android.permission.INTERNET"/>
```

**2단계 Youtube API 연동을 파일 다운로드**
- [youtube API 다운로드 사이트](https://developers.google.com/youtube/android/player/downloads?hl=ko)
- 안드로이드 스튜디오에서는 gradle를 이용하여 import를 하지만 youtube API가 아직도 이클립스 기준으로 설정되어 있기 때문에 불편함. 자체적으로 제공하는 플레이어도 불편하고 구린편

**3단계 종속성 추가**
- 다운로드 한 zip파일의 압축을 풀면 샘플 코드가 있습니다. 그 중에서 libs 폴더에 `YouTubeAndroidPlayerApi.jar`파일을 복사하여 안드로이드 스튜디오 프로젝트의 app/libs/ 에 붙여넣기
- 안드로이드 스튜디오의 상단의 File - Project Structure - Dependencies - app - + 버튼 - `YouTubeAndroidPlayerApi.jar`의 종속성 추가
- 약간의 시간이 지나면 gradle build가 되면서 synced 성공 메시지가 뜨면 성공

**4단계 구글 인증정보 만들기**
- [구글 개발자 api 사이트](https://console.developers.google.com/apis)
- 새로운 프로젝트 생성하면 약간의 시간이 소요됨 =
- 사용자 인증 정보 - 상단에 사용자 인증 정보 만들기 클릭 생성되는 키값 복사후 메모장에 저장
- 안드로이드 스튜디오에 와서 gradle projects - app - signing reposts 클릭하면 sha1 키값 복사 [방법 참고](https://snowdeer.github.io/android/2017/08/21/android-studio-debug-sha1/)
- api 사이트에가서 제한사항 추가 - Android 앱 항목을 선택하고 패키지명과 sha1 기입

**5단계 화면 및 코드 작성**
- activity_main.xml 레이아웃에 `YouTubePlayerView` 의 뷰와 버튼 하나 만들기

```javascript
<com.google.android.youtube.player.YouTubePlayerView
        android:id="@+id/youtubeView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="80dp"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/youtubeBtn"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginStart="256dp"
        android:layout_marginLeft="256dp"
        android:layout_marginTop="456dp"
        android:text="Button"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```

- MainAcitiy 파일에 아래의 코드 추가하는데 **유투브 extends YoutubebaseActivity 반드시 필요**

```javascript
public class MainActivity extends YouTubeBaseActivity {
    YouTubePlayerView youTubePlayerView;
    Button btn;
    YouTubePlayer.OnInitializedListener listener;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        btn = findViewById(R.id.youtubeBtn);
        youTubePlayerView = findViewById(R.id.youtubeView);
        listener = new YouTubePlayer.OnInitializedListener() {
            @Override
            public void onInitializationSuccess(YouTubePlayer.Provider provider, YouTubePlayer youTubePlayer, boolean b) {
                youTubePlayer.loadVideo("NmkYHmiNArc"); //
                //https://www.youtube.com/watch?v=NmkYHmiNArc 유투브에서 v="" 이부분이 키에 해당
            }

            @Override
            public void onInitializationFailure(YouTubePlayer.Provider provider, YouTubeInitializationResult youTubeInitializationResult) {

            }
        };

        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                youTubePlayerView.initialize("개발자 콘솔에서 만든 api키", listener);

            }
        });
    }
}
```
- 버튼을 클릭하면 api키를 바탕으로 요청하고 listener 동작
- loadVideo(유투브 영상 키값)을 통해서 불러오면 끝



**6단계 실행후 테스트**
- [동빈나님의 유투브](https://www.youtube.com/watch?v=vewH-f3fAes)를 참고해서 만들었고 안드로이드 스튜디오 버전/구글 콘솔은 이전버전이기에 최신 버전에 맞춰서 포스트를 작성했습니다. ()


## 결론
- **유투브 연결하는게 익혔으니 직접 검색해서 데이터 가져오는 것도 만들어야 할듯**