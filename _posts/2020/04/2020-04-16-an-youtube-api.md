---
title: Android Youtube Data API v3 사용법 - 영상 리스트 검색하기
tags: [Android, Youtube]
style: fill
color: dark
description: 안드로이드에서 youtube data API 를 활용하여 검색하고 정보를 가져오는 방법
---

## Youtube Data API로 영상 리스트 검색하고 가져오기
유튜브 영상의 정보를 가져오는 방법이 2개가 존재한다. 하나는 oAuth를 통한 인증방법이고 다른 하나는 API키를 활용한 인증방법이다. 이 포스트에서는 API키를 통한 샘플 코드이다. 하지만 API키를 활용하면 앱으로 배포했을 때 노출이 되므로 위험하니 키 제한이 필요하고 클라이언트 단보다는 서버 단 로직에서 실행해야 키 자체가 노출이 안된다. 클라이언트 단에서 만들고 싶으면 oauth 사용을 권장합니다.


**1단계 안드로이드 샘플 프로젝트 만들기**
- 안드로이드 스튜디오를 실행시킨 후에 Empty 테마를 사용하여 새로운 프로젝트를 만듭니다.
- 그 이후 인터넷 권한 사용을 위해 androidManifest.xml 파일에 아래의 코드를 추가해주세요.

```javascript
<uses-permission android:name="android.permission.INTERNET"/>
```

**2단계 Youtube API키 만들기**
- youtube 의 자료를 검색하고 가져오려면 API 키가 반드시 필요하다.
- [API키 만들기](android-youtube)포스트를 통해 사용할 API키를 만들어보자. 이미 있다면 생략해도 된다.


**3단계 종속성 추가**

```javascript
apply plugin: 'com.android.application'

android {
    packagingOptions {
        exclude 'META-INF/DEPENDENCIES'
    }
}

dependencies {
    //생략
    implementation 'com.google.apis:google-api-services-youtube:v3-rev183-1.22.0'
    implementation 'com.google.http-client:google-http-client-android:+'
    implementation 'com.google.api-client:google-api-client-android:+'
    implementation 'com.google.api-client:google-api-client-gson:+'
}
```
- 앱 수준의 그래들 파일을 오픈하여 상단의 코드 추가
- pacakgeingOptions 저거를 추가해야 빌드가 잘 됩니다.
- dependencies 의 코드 추가
- grandl synced !! 

**4단계 간단한 레이아웃 만들기**

```javascript
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <Button
        android:id="@+id/btn_search"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginStart="144dp"
        android:layout_marginLeft="144dp"
        android:layout_marginTop="116dp"
        android:text="비디오 검색하기"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/btn_show"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginStart="144dp"
        android:layout_marginLeft="144dp"
        android:layout_marginTop="196dp"
        android:text="비디오 검색결과보기"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/tv_result"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello World!"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</android.support.constraint.ConstraintLayout>
```
- 유튜브를 검색하기 위한 버튼, 결과 확인 버튼, 결과를 보여줄 텍스트 뷰 생성

**5단계 코드 작성**

```javascript
public class MainActivity extends AppCompatActivity {
    private String API_KEY = "youtube_api_key";

    private TextView tv_result;
    private String result;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        tv_result = findViewById(R.id.tv_result);

        findViewById(R.id.btn_search).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                YoutubeAsyncTask youtubeAsyncTask = new YoutubeAsyncTask();
                youtubeAsyncTask.execute();
            }
        });

        findViewById(R.id.btn_show).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                tv_result.setText(result);

            }
        });
    }

    private class YoutubeAsyncTask extends AsyncTask<Void, Void, Void> {
        @Override
        protected void onPreExecute() {
            super.onPreExecute();
        }

        @Override
        protected Void doInBackground(Void... voids) {
            try {
                HttpTransport HTTP_TRANSPORT = new NetHttpTransport();
                final JsonFactory JSON_FACTORY = new JacksonFactory();
                final long NUMBER_OF_VIDEOS_RETURNED = 5;

                YouTube youtube = new YouTube.Builder(HTTP_TRANSPORT, JSON_FACTORY, new HttpRequestInitializer() {
                    public void initialize(HttpRequest request) throws IOException {
                    }
                }).setApplicationName("youtube-search-sample").build();

                YouTube.Search.List search = youtube.search().list("id,snippet");

                search.setKey(API_KEY);

                search.setQ("아이린");
                search.setChannelId("UCk9GmdlDTBfgGRb7vXeRMoQ"); //레드벨벳 공식 유투브 채널
                search.setOrder("relevance"); //date relevance

                search.setType("video");

                search.setFields("items(id/kind,id/videoId,snippet/title,snippet/thumbnails/default/url)");
                search.setMaxResults(NUMBER_OF_VIDEOS_RETURNED);
                SearchListResponse searchResponse = search.execute();

                List<SearchResult> searchResultList = searchResponse.getItems();

                if (searchResultList != null) {
                    prettyPrint(searchResultList.iterator(), "레드벨벳");
                }
            } catch (GoogleJsonResponseException e) {
                System.err.println("There was a service error: " + e.getDetails().getCode() + " : "
                        + e.getDetails().getMessage());
                System.err.println("There was a service error 2: " + e.getLocalizedMessage() + " , " + e.toString());
            } catch (IOException e) {
                System.err.println("There was an IO error: " + e.getCause() + " : " + e.getMessage());
            } catch (Throwable t) {
                t.printStackTrace();
            }

            return null;
        }

        @Override
        protected void onPostExecute(Void aVoid) {
            super.onPostExecute(aVoid);
        }

        public void prettyPrint(Iterator<SearchResult> iteratorSearchResults, String query) {
            if (!iteratorSearchResults.hasNext()) {
                System.out.println(" There aren't any results for your query.");
            }

            StringBuilder sb = new StringBuilder();

            while (iteratorSearchResults.hasNext()) {
                SearchResult singleVideo = iteratorSearchResults.next();
                ResourceId rId = singleVideo.getId();

                // Double checks the kind is video.
                if (rId.getKind().equals("youtube#video")) {
                    Thumbnail thumbnail = (Thumbnail) singleVideo.getSnippet().getThumbnails().get("default");
                    sb.append("ID : " + rId.getVideoId() + " , 제목 : " + singleVideo.getSnippet().getTitle() + " , 썸네일 주소 : " + thumbnail.getUrl());
                    sb.append("\n");
                }
            }

            result = sb.toString();
        }
    }
}

```
- 버튼을 클릭하면 비동기로 유투브 비디오 검색 실행
- 기존의 http을 활용하여 접근하는 방식이 아니라 구글에서 만든 https, json 등의 통신방식 활용
- Youtube.search.List는 유투브의 data api 중 검색 이용. Youtube.videos.List 비디오 정보 등 [Youtube Data API 형식](https://developers.google.com/youtube/v3/docs/search/list?hl=ko)의 사이트를 참조하면 된다.
- 각 API마다 필수사항이 있지만 검색에서는 part(id, snippet) 가 필수 매배변수로 어떤 정보에 대한 요청을 할 건지 명시 필요
- 선택 매개변수로 채널값, 검색 쿼리, 타입, 정렬순(최신 또는 관련순), setFields()를 통해 어떠한 데이터 정하기... 모든 데이터를 다 가져오려고 하면 하루 할당량 10,000 초과가 발생할 수 있으므로 필요한 정보만 거져오는게 좋다. [초과량 확인 방법](it-youtube-error)

**6단계 실행후 테스트**
- [YOutube Dat API 공식문서](https://developers.google.com/youtube/v3/quickstart/android)를 참고해서 만들었고 안드로이드 스튜디오 버전/구글 콘솔은 최신 버전에 맞춰서 포스트를 작성했습니다.


## 결론
- **필수 매개변수는 반드시 넣고 선택 매개변수의 조절을 통해 할당량을 초과하지 않도록 테스트 하는게 중요.. 많이 사용하면 초기화 될 때 까지 사용불가**
- **레드벨벳 아이린♥♥♥♥♥♥♥♥♥♥♥♥♥**