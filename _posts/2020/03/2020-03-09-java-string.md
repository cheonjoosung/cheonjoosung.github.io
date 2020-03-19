---
title: 자바 스트링 클래스 및 문자열 활용법
tags: [Java, 손코딩]
style: fill
color: dark
description: 자바 String Class 및 문자열 활용법IT(모바일, 웹, DB, 빅데이터 등)의 교육을 유료 또는 무료로 수강할 수 있는 사이트
---

## String Class
String은 문자열 문장을 의미하고 아래와 같은 것들이 문자열이다

```javascript
String a = "Hello"
String b = "Hello"
String c = new String("Hello"):
```

String은 리터럴 표현없이 사용할 수 있도록 자바에서 만들어준 클래스 자료형이다. 그렇기에 자바에서 레퍼런스 타입으로 생각할 수 있다. 하지만, String은 특이하게 Immutable 하다.?? 

```javascript
String a = "Hello"

print(a) - "hello"
change(a)

void change(String temp) {
    temp = "NO"
    print(temp) - "NO"
}

print(a) - "hello"
```

레퍼런스 타입은 주소를 전달하기에 함수를 호출한 곳에서 값이 변경되면 복귀해서도 값이 바뀐다. 하지만, String 클래스의 값을 할당할 때 private final char value[]; 으로 선언되어있고 final 키워드를 생성하기에 변경이 불가능하다.

```javascript
String a = "Hello"
a = "new String" + "new String"
```

변경이 불가능한데 a 변수에 새로운 문자열을 할당할 수 있다. 이유는 새로운 "New String"의 새로운 객체를 만들어서 a에다가 넣기 때문이다. 즉, 새로운 문자열이 만들어졌고 a는 그 값을 가리키도록 변경된 것이다.

하나의 코드를 더 살펴보자

```javascript
String a = "hello";
String b = "hello";
String c = new String("hello");
		
if(a == b) System.out.println("true a==b"); //출력됨
if(a == c) System.out.println("true a==c"); //출력되징 ㅏㄴㅎ음

결과는 
```

a, b, c의 모든 값은 .equals 통해 비교하면 같은 결과값이 나온다. 하지만, new 키워드를 통해 생성하면 저장공간에 새로운 "hello" 객체를 만들고 c변수가 가리키도록 만든다. 하지만, b에서 "hello"는 이전의 생성된 a 변수가 가리키는 "hello"이다. a와b는 같은 주소값을 가지므로 a==b 의 결과값은 True 가 나옵니다.

## String 클래스의 주요 메소드
[자바 스트링 클래스](https://docs.oracle.com/javase/7/docs/api/java/lang/String.html)공식 사이트에서 보면 정확한 정보를 확인할 수 있습니다.

- chatAt(i) : i 번째의 character를 가져오기
- contains(CharSequence) : 특정 문자열을 포함했는지에 따라 True Or False 리턴
- startsWith(String/endsWith(String) : 특정 문자열로 시작/끝나는지에 따라 True Or False 리턴
- indexOf(int ch) : 문자의 인덱스를 반환
- replace(old, new) / replaceAll(old, new) : old 캐릭터/문자열을 new 캐릭터/문자열로 변환
- toLowwerCase(), toUppercase() : 문자열을 소문자/대문자로 모두 변환
- length() : 문자열의 길이 반환
- trim() : 문자열의 공백 제거
- split(규칙) : " "가 규칙이면 공백에 따라 분리되어 String [] 이 반환된다
- valueOf(primite Type) : int, float 등의 프리미티브 타입을 스트링으로 변환해준다
- substring() : 문자열 중 일부를 잘라서 가져올 수 있다

## 문자열 활용 문제
```javascript
String a = "abcde"
char[] charArray = a.toCharArray();
		
for(int i=0 ; i<charArray.length/2 ; i++) {
    char temp = charArray[i];
    charArray[i] = charArray[charArray.length-1-i];
	charArray[charArray.length-1-i] = temp;
}
```
String은 Immutable Class이므로 값을 변환하는 것 자체가 불가능하다. 따라서 charArray() 메소드를 활용하여 변환하고 문자열을 뒤짚으면 된다. 그리고 뒤짚어진 배열의 값들을 다시 스트링으로 바꾸면 되지만 손이 많이 간다. 

다른 방법으로 StringBuilder/StringBuufer의 reverse() 메소드를 사용하면 편하다.


```javascript
print(11 + 11 + "11") -> 2211
print("11" + 11 + 11) -> 111111
```
숫자+문자의 경우 문자열로 반환된다

## 결론
- **String 은 Immutable Class** 이다.
- 공식 문서를 한번 읽어보는 것이 좋다.