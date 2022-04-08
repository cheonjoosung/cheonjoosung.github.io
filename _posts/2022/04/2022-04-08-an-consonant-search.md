---
title: Android 한글 초성 검색 어댑터
tags: [Android]
style: fill
color: dark
description: Android 한글 초성 검색 어댑터
---

## 한글 초성 검색
영어의 경우 한글처럼 받침이 있지 않기에 contain()만으로 주어진 목록에서 필요한 데이터를 추출할 수 있다.

한글 단어의 경우 contains()를 통해서 검색 기능을 어느 정도 활용할 수 있지만 초성(ㅅㄱ, ㄱㄴ, ㄱㄴㄷ)을 사용하다면 추가적인 기능이 필요하다.



**Unicode 사용**
```javascript

private val koreanUnicodeStart = 44032 // 가
private val koreanUnicodeEnd = 55203   // 힣

private val koreanUnicodeBased = 588   // 각 자음 마다 가지는 글자 수

// 자음
private val koreanConsonant = arrayOf(
    'ㄱ', 'ㄲ', 'ㄴ', 'ㄷ', 'ㄸ', 'ㄹ', 'ㅁ', 'ㅂ', 'ㅃ',
    'ㅅ', 'ㅆ', 'ㅇ', 'ㅈ', 'ㅉ', 'ㅊ', 'ㅋ', 'ㅌ', 'ㅍ', 'ㅎ'
)
```
가~힣까지의 범위에 있으면 한글이라 생각하면 된다. koreanUnicodeBased는 각 자음 마다 가지는 글자 수를 통해 가,갸,고,겨.......등은 ‘ㄱ’ 글자의 맵핑 시키기 위해 사용한다.


**검색 키워드의 초성 or 한글 체크**
```javascript
/**
* 해당 문자가 초성인지 체크
*/
private fun isConsonant(ch: Char): Boolean {
  return koreanConsonant.contains(ch)
}

/**
* 해당 문자가 한글인지 체크
*/
private fun isKorean(ch: Char): Boolean {
  val charCode = ch.code
  return charCode in koreanUnicodeStart..koreanUnicodeEnd
}

/**
* 자음을 얻는다
*/
private fun getConsonant(ch: Char): Char {
  val hasBegin = (ch.code - koreanUnicodeStart)
  val idx = hasBegin / koreanUnicodeBased
  return koreanConsonant[idx]
}
```
비교대상과 검색어의 각 글자를 비교하기 위해서는 초성(ㄱ, ㄴ, ㄷ.....) 또는 한글(가 ~ 힣)인지를 알아야 한다.

(**문자열의 코드값** - **유니코드 시작값**) /  **각 자음마다 가지는 글자 수의 몫은 자음 배열에서 각 위치를 의미한다.**

예를 들어 **“난”이라고 가정을하면**  가  ~ 나  ~~ **난** ~~ 다 ...... 처럼 유니코드 순서가 존재한다. 각 자음에 시작코드값을 빼면 ‘가’의 위치는 0 이고 ‘나’의 위치는 1 * koreanUnicodeBased 이고 ‘다의’ 위치는 2 * koreanUnicodeBased 가 된다.

**‘난’의 값은 1*koreanUnicodeBased + 알파 이므로 koreanUnicodeBased로 나누면 index 값이 1이 나오고 이는 자음 배열에서 ‘ㄴ’이 되므로 난의 자음을 찾을 수 있게 된다**.


**비교대상을 검색어와 맵핑**
```javascript
/**
* 초성 또는 한글 검색
* @param based 비교대상
* @param search 검색단어
*/
fun matchKoreanAndConsonant(based: String, search: String): Boolean {
  var temp = 0
  val diffLength = based.length - search.length
  val searchLength = search.length

  if (diffLength < 0) {
      return false
  } else {
      for (i in 0..diffLength) {
          temp = 0

          while (temp < searchLength) {
              if (isConsonant(search[temp]) && isKorean(based[i + temp])) {
                  // 현재 char이 초성이고 based가 한글일 때
                  if (getConsonant(based[i + temp]) == search[temp]) {
                      // 각각의 초성끼리 같은지 비교
                      temp++
                  } else {
                      break
                  }
              } else {
                  // char가 초성이 아니라면
                  if (based[i + temp] == search[temp]) {
                      temp++
                  } else {
                      break
                  }
              }
          }

          // 모두 일치한 결과
          if (temp == searchLength) return true
      }

      // 일치하는 것을 찾지 못했을 때
      return false
  }

}
```
“가나다” 에서 “ㄱㄴ”, “ㄴㄷ”, “ㄷ” 등 다양하게 찾는 경우가 발생한다. searchKeyword의 모든 자음 또는 문자가 based와 모두 맞는지를 비교하여 일치하는 경우에는 true를 리턴하도록 구현한다.


**샘플 코드**
```javascript
override fun getFilter(): Filter {
        return object : Filter() {
            override fun performFiltering(constraint: CharSequence): FilterResults {
                val charString = constraint.toString()

                if (charString.isEmpty()) {
                    filteredList.clear()
                    filteredList.addAll(dataSet)
                } else {
                    val filteringList: ArrayList<String> = ArrayList()

                    val koreanMatcher = KoreanMatcher()

                    for (name in dataSet) {
                        if (koreanMatcher.matchKoreanAndConsonant(name, charString)) filteringList.add(name)
                        else { // 영어인 경우
                            if (name.lowercase(Locale.getDefault()).contains(charString.lowercase(Locale.getDefault()))) {
                                filteringList.add(name)
                            }
                        }
                    }

                    filteredList.clear()
                    filteredList.addAll(filteringList)
                }

                val filterResults = FilterResults()
                filterResults.values = filteredList
                return filterResults
            }

            override fun publishResults(constraint: CharSequence, results: FilterResults) {
                filteredList = results.values as ArrayList<String>
                notifyDataSetChanged()
            }
        }
    }
```
검색 키워드를 koreanMatcher를 통해 먼저 비교하고 실패할 경우 contains()을 통해 한번 더 비교하도록 구현했다.  코드를 한번 더 리팩토링 한다면 한글 vs 영어를 선 체크후 따로 호출하는 것이 더 좋아보인다.

adapter 나 textWater의 내용은 비슷하기에 Filterable 관련 코드만 추가했고 자세한 내용은 아래의 위치에서 프로젝트를 확인하면 된다.
[깃헙코드 참고](https://github.com/cheonjoosung/Android-Consonant-SearchAdapter)
