---
title: Android Youtube Search API with data v3 유튜브 검색 API
tags: [Android]
style: fill
color: dark
description: Android Youtube Search API with data v3 유튜브 검색 API
---

## 설명
유튜브 검색 API를 사용하기 위해서는 구글 클라우드에서 설정이 필요하다.


### API 설정
- [https://console.cloud.google.com/apis/dashboard?hl=ko&pli=1](https://console.cloud.google.com/apis/dashboard?hl=ko&pli=1)
- 라이브러리 > Youtube 검색 > Data API V3 사용
- 사용자 인증정보 클릭 > 사용자 인증정보 만들기 - API 키 복사 > 키제한설정 - Youtube Data API V3 고정

![preview](https://github.com/cheonjoosung/cheonjoosung/blob/master/image/youtube/youtube1.png?raw=true)

### 개발환경
- Android Studio Chipmunk 2021.2.1
- Gradle 7.1.3
- Google AAC, retrofit2, rx, gldie, pierfrance, pierfrancescosoffritti.androidyoutubeplayer


### 코드

**viewModel**

```kotlin
fun getYoutubeVideo(keyword: String) = viewModelScope.launch {

        val result = searchRepository.getYoutubeVideo(keyword)

        when (result.code()) {
            200 -> {

                val idList = mutableListOf<String>()
                val convertedList = mutableListOf<Video>()

                result.body()?.let {

                    nextPageToken = it.nextPageToken

                    it.items?.let { list ->
                        for (item in list) {

                            val id = item.id
                            val snippet = item.snippet

                            convertedList.add(
                                Video(
                                    videoId = id.videoId,
                                    title = snippet.title,
                                    description = snippet.description,
                                    publishedAt = snippet.publishedAt,
                                    imgUrl = snippet.thumbnails.medium.url,
                                    channelTitle = snippet.channelTitle
                                )
                            )

                            idList.add(id.videoId)
                        }
                    }

                }

                coroutineScope {
                    (0 until idList.size).map { idx ->
                        async(Dispatchers.IO) {
                            val resultInfo = searchRepository.requestVideoInfo(idList[idx])

                            when (resultInfo.code()) {
                                200 -> {
                                    resultInfo.body()?.let {
                                        convertedList.find { video ->
                                            video.videoId == idList[idx]
                                        }?.let { findVideo ->
                                            it.items?.let { list ->
                                                for (item in list) {
                                                    findVideo.duration =
                                                        item.contentDetails.duration
                                                    findVideo.viewCount =
                                                        item.statistics.viewCount.toString()
                                                }
                                            }
                                        }
                                    }
                                }
                                else -> {
                                    Log.e(TAG, "onFailure ${resultInfo.message()}")
                                }
                            }
                        }
                    }.awaitAll()

                    firstSearch.postValue(convertedList)
                }

            }
            else -> {
                Log.e(TAG, "onFailure ${result.message()}")
            }
        }

    }
```

- 첫 요청에서 받은 비디오 검색에서 videoID를 따로 저장
- videoID로 Video API 요청하여 duration & viewCount 저장
- 코루틴 & async를 사용하여 각 요청을 비동기로 진행하고 모든 결과를 기다린 후에 리스트를 추가


**repository**

```kotlin
@WorkerThread
suspend fun getYoutubeVideo(searchText: String) =
    YoutubeApiRequestFactory.retrofit.getYouTubeVideos(
        apiKey = apiKey, query = searchText, videoOrder = "relevance", maxResults = 10
    )

@WorkerThread
suspend fun requestVideoInfo(videoId: String) =
    YoutubeApiRequestFactory.retrofit.getVideoInfo(
        apiKey = apiKey,
        videoId = videoId
    )

object YoutubeApiRequestFactory {

    private const val youtubeBaseUrl = "https://www.googleapis.com/youtube/v3/"

    val retrofit: YoutubeApiService = Retrofit.Builder()
        .baseUrl(youtubeBaseUrl)
        .addConverterFactory(GsonConverterFactory.create())
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .client(
            OkHttpClient.Builder().addInterceptor(
                HttpLoggingInterceptor().apply { this.level = HttpLoggingInterceptor.Level.BODY }
            ).build())
        .build()
        .create(YoutubeApiService::class.java)
}

interface YoutubeApiService {

    @GET("search")
    suspend fun getYouTubeVideos(
        @Query("key") apiKey: String,
        @Query("q") query: String,
        @Query("order") videoOrder: String,
        @Query("type") videoType: String = "video",
        @Query("maxResults") maxResults: Int,
        @Query("channelId") channelId: String = "",
        @Query("part") part: String = "snippet",
    ): Response<YoutubeVideo>

    @GET("search")
    suspend fun getYouTubeMoreVideos(
        @Query("key") apiKey: String,
        @Query("q") query: String,
        @Query("pageToken") nextPageToken: String,
        @Query("order") videoOrder: String,
        @Query("type") videoType: String = "video",
        @Query("maxResults") maxResults: Int,
        @Query("channelId") channelId: String = "",
        @Query("part") part: String = "snippet",
    ): Response<YoutubeVideo>

    @GET("videos")
    suspend fun getVideoInfo(
        @Query("key") apiKey: String,
        @Query("id") videoId: String,
        @Query("part") part: String = "contentDetails, statistics",
    ): Response<YoutubeVideoInfo>
}
```

- apiKey 는 구글 콘솔 API 에서 복사한 키를 사용
- part 에 본인이 원하는 정보가 있으면 data api v3 모델을 참조하여 요청하면 된다.
- 요청 결과를 맵핑하기 위한 VO 객체들이 필요


**network model**

```kotlin
import android.os.Parcelable
import com.google.gson.annotations.Expose
import com.google.gson.annotations.SerializedName
import kotlinx.parcelize.Parcelize

@Parcelize
data class YoutubeVideo(
    @SerializedName("kind")
    @Expose
    val kind: String,
    @SerializedName("etag")
    @Expose
    val etag: String,
    @SerializedName("nextPageToken")
    @Expose
    val nextPageToken: String,
    @SerializedName("regionCode")
    @Expose
    val regionCode: String,
    @SerializedName("pageInfo")
    @Expose
    val pageInfo: PageInfo,
    @SerializedName("items")
    @Expose
    val items: List<Items>?
) : Parcelable

@Parcelize
data class PageInfo(
    @SerializedName("totalResults")
    @Expose
    val totalResults: Int,
    @SerializedName("resultsPerPage")
    @Expose
    val resultsPerPage: Int
) : Parcelable

@Parcelize
data class Items(
    @SerializedName("id")
    @Expose
    val id: Id,
    @SerializedName("snippet")
    @Expose
    val snippet: Snippet
) : Parcelable

@Parcelize
data class Id(
    @SerializedName("kind")
    @Expose
    val kind: String,
    @SerializedName("videoId")
    @Expose
    val videoId: String
) : Parcelable

@Parcelize
data class Snippet(
    @SerializedName("publishedAt")
    @Expose
    val publishedAt: String,
    @SerializedName("channelId")
    @Expose
    val channelId: String,
    @SerializedName("title")
    @Expose
    val title: String,
    @SerializedName("description")
    @Expose
    val description: String,
    @SerializedName("thumbnails")
    @Expose
    val thumbnails: ThumbNail,
    @SerializedName("publishTime")
    @Expose
    val publishTime: String,
    @SerializedName("channelTitle")
    @Expose
    val channelTitle: String,
) : Parcelable

@Parcelize
data class ThumbNail(
    @SerializedName("medium")
    @Expose
    val medium: Medium
) : Parcelable

@Parcelize
data class Medium(
    @SerializedName("url")
    @Expose
    val url: String,
    @SerializedName("width")
    @Expose
    val width: Int,
    @SerializedName("height")
    @Expose
    val height: Int
) : Parcelable


@Parcelize
data class YoutubeVideoInfo(
    @SerializedName("kind")
    @Expose
    val kind: String,
    @SerializedName("etag")
    @Expose
    val etag: String,
    @SerializedName("items")
    @Expose
    val items: List<TrendItem>?
) : Parcelable

@Parcelize
data class TrendItem(
    @SerializedName("kind")
    @Expose
    val kind: String,
    @SerializedName("etag")
    @Expose
    val etag: String,
    @SerializedName("id")
    @Expose
    val id: String,

    @SerializedName("snippet")
    @Expose
    val snippet: Snippet,

    @SerializedName("tags")
    @Expose
    val tags: List<String>,

    @SerializedName("contentDetails")
    @Expose
    val contentDetails: ContentDetails,

    @SerializedName("statistics")
    @Expose
    val statistics: Statistics
) : Parcelable

@Parcelize
data class ContentDetails(
    @SerializedName("duration")
    @Expose
    val duration: String
) : Parcelable

@Parcelize
data class Statistics(
    @SerializedName("viewCount")
    @Expose
    val viewCount: String? = ""
) : Parcelable
```

- SerializedName 를 사용하지 않는 경우 debug모드에서는 문제가 없지만 release 모드에서 작동하지 않을 수 있음 → proguard 가 적용되어 클래스 맵핑이 안될 수 있음


### 참조
- 유튜브 검색 : [https://github.com/cheonjoosung/Youtube-Search-Android](https://github.com/cheonjoosung/Youtube-Search-Android)
- 유튜브 모션레이아웃  : [https://github.com/cheonjoosung/Youtube-MotionLayout](https://github.com/cheonjoosung/Youtube-MotionLayout)
- 유튜브 Data API : [https://developers.google.com/youtube/v3/docs?hl=ko](https://developers.google.com/youtube/v3/docs?hl=ko)
