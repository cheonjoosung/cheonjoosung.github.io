---
title: 자바 String vs StringBuilder vs StringBuffer 비교
tags: [Java]
style: fill
color: dark
description: IT(모바일, 웹, DB, 빅데이터 등)의 교육을 유료 또는 무료로 수강할 수 있는 사이트
---

## String vs StringBuilder vs StringBuffer

| String         | StringBuilder | StringBuffer  |
|:-------------:|:-------------:|:-------------:|
| 불변            |가변            | 가변            |
|Thread Not Safe |Thread Not Safe| Thread-Safe   |

이전 포스트인 [String 클래스](java-string)에서 언급했듯이 스트링 클래스는 **Immutable Class**로 불변입니다. String temp = "a" + "b" + "c" + "d" 이와 같은 String을 연결하게 되는 경우 +연산마다 new String()과 concat()을 통해 메모리 낭비가 발생합니다.

하지만, StirngBuilder or StringBuffer의 경우 변할수 있기 때문에 다양한 활용이 가능합니다. 
```javascript
StringBuilder sb = new StringBuilder("str")
sb.append(" abc")
sb.append(" def")

print(sb.toString())

StringBuffer sb2 = new StringBuffer("str")
sb2.append(" abc")
sb2.append(" def")

print(sb2.toString())
```

StringBuilder & StringBuffer 는 AbstractStringBuilder.class를 상속받아서 다양한 기능을 사용하고 있습니다. 이 클래스안에 char[] value의 변수가 존재하고 이를 상속받아서 사용한다. new 연산자나 .append()가 호출이 되면 내부적으로 변수에 가용공간 확보를 하고 값을 복사하는 과정이 이루어진다.

[오라클 AbstractStringBuilder.class 정보](https://docs.oracle.com/javase/7/docs/api/java/lang/StringBuilder.html)

멀티 쓰레드 환경에서 사용해야 한다면 동기화를 지원하는 StringBuffer를 사용해야 한다. String 과 StringBuilder는 쓰레드에서 안전하지 않기 때문이다.

문자열과 관련한 코딩 테스트 또는 프로그램이 존재한다면 String을 쓰는 것보다 StringBuilder or StringBuffer를 쓰는 것이 좋다. 10~20번 사이의 과정만 반복되는 것이라면 실행시간의 큰 차이는 존재하지 않지만 그 단위가 억에서 십억이 되는 순간 처리량이 중요해진다.

**String은 + 연산자나 할당을 많이 하게 될 수록 코드의 성능이 떨어진다.** 반면에 StringBuilfer/StringBuffer는 String 보다는 연산이 많은 상황에서 더 좋은 성능을 발휘할 수 있다.

## StringBuilder StringBuffer 사용법
StringBuilder/StringBuffer의 경우 동일한 클래스를 상속받아서 **많은 부분이 유사**하다.

```javascript
StringBuilder sb = new StringBuilder("abcde")

//문자열 뒤짚기
sb.reverse()
print(sb.toString())

for(int i=0 ; i<sb.length()/2 ; i++) { //가변이기에 직접 접근하여 값을 변경할 수 있음
    char tmp = sb.charAt(i);
	sb.setCharAt(i, sb.charAt(sb.length()-1-i));
	sb.setCharAt(sb.length()-1-i, tmp);
}

//문자열 더하기
sb.append("1번").append("2번").append("3번")....

//String 클래스로 변환
String temp = sb.toString()
```
이외에 String 클래스에서 사용하는 charAt(), substring(), indexOf() 등 많은 기능을 동일하게 사용할 수 있다.


## 결론
- **String 참 편하지만 평소에도 ;StringBuilder 와 StringBuffer를 쓰는 것이 좋다**