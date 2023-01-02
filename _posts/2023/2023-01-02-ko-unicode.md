---
title: Kotlin 유니코드 한글 변환하기 | kotlin unicode to korean
tags: [Android, Kotlin]
style: fill
color: dark
description: Kotlin 유니코드 한글 변환하기 | kotlin unicode to korean
---

## 유니코드 한글로 변환하기
가끔식 네트워크를 통한 데이터를 받아보면 유니코드로 오는 경우가 있다.

- "formattedDate":2023\ub144 1\uc6d4 2\uc77c 이런식으로..
- "formattedDate":2023년 1월 2일 로 변경이 필요하다.

바꾸는 방법은 간단하다.

```kotlin
  private fun convertStringUnicodeToKorean(data: String): String {

  val sb = StringBuilder() // 단일 쓰레드이므로 StringBuilder 선언
  var i = 0

  /**
   * \uXXXX 로 된 아스키코드 변경
   * i+2 to i+6 을 16진수의 int 계산 후 char 타입으로 변환
   */
  while (i < data.length) {

    if (data[i] == '\\' && data[i + 1] == 'u') {

      val word = data.substring(i + 2, i + 6).toInt(16).toChar()
      sb.append(word)
      i += 5
    } else {
      sb.append(data[i])
    }

    i++
  }

  return sb.toString()
}
```
- 16진수 값의 형태를 취하기에 i 번째가 '\' i+1 번째가 'u' 일때 i+2 ~ i+6 까지의 문자열을 만든다
- 이를 16진수 정수로 만든 후 다시 char 로 변환하면 된다.


## 결론
- 
