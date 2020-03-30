---
title: Java Collection framework & List, Set, Map Interface
tags: [Java]
style: fill
color: dark
description: 자바의 컬렉션 프레임워크와 주요 인터페이스인 List, Set, Map을 알아보자
---

## Collection Framework
Collection Framework는 데이터를 위한 자료 구조 및 알고리즘을 구조화하여 만들어 구현해 놓은 것이다.

인터페이스로 구현된 이유는 구현하고 있는 클래스에서만 수정을 하면 되기 때문이다. 상속으로 연결이 되어있다면 거치는 클래스에서 매번 수정을 해야하지 않을까 생각한다. 결국 재사용성과 연관되어 있지 않을까??? 

[스택오버 플로우 추상클래스가 아니고 인터페이스로 구현된 이유](https://softwareengineering.stackexchange.com/questions/262706/why-liste-is-an-interface-but-not-abstract-class)

**1. 주요 인터페이스 특징**
- List<E> : 순서가 있고 중복 O
- Set<E> : 순서가 없고 중복 X
- Map<Key, Value> : 키와 값의 상 순서 없음. 키 중복 x 값은 중복 O

**2. 주요 메소드**
- add() : 요소 추가
- clear() : 비우기
- contains() : 포함 여부 체크
- equals() : 비교
- isEmpty() : 비었는지 체크 
- size() : 크기 반환
- remove() : 특정 객체 제거

## Collection Framework의 주요 인터페이스
**1.List**;
- ArrayList, Vector : 순서가 있는 자료구조형으로 비슷. 벡터는 현재 기존 코드와의 호환성을 위해서 존재하므로 ArrayList를 추천
```javascript
List<Integer> list = new ArrayList<>()
list.add(1) list.add(2) list.add(3)

for(int i=0 ; i<list.size() ; i++)
    list.get(i)

Collections.sort(list)

list.remove(1);
```

- LinkedList : 저장방식이 다름. 내부의 포인터를 통해 연결.
```javascript
LinkedList<Integer> list = new LinkedList<>();
list.add(4) list.add(2) list.add(3)

list.remove(1);
print(list.get(0))
list.set(0, 4)
```

- Stack : First In Last Out의 자료구조형
```javascript
Stack<Integer> s = new Stack<>();

s.push(1);
s.push(4);

s.pop();
if(!s.isEmpty()) s.pop();
```

- Queue : First In First Out의 자료구조형
```javascript
Queue<Integer> s = new LinkedList();

s.add(1);
s.add(4);

while(!s.isEmpty()) {
    Integer val = s.poll();
    System.out.println(val);
}
```

**2.Set**
- HashSet : 순서에 상관없이 저장하고 중복된 값은 저장하지 않는다. 순서가 필요하다면 LinkedHashSet 사용
```javascript
HashSet<String> hs = new HashSet<>();

hs.add("1");
hs.add("2");
hs.add("3");

Iterator<String> it = hs.iterator();

while (it.hasNext()) {
    System.out.println(it.next());
}
```

- TreeSet : 데이터가 정렬된 상태로 저장되고 이진 검색 트리의 형태가 된다
```javascript
TreeSet<Integer> ts = new TreeSet<>();

ts.add(1);
ts.add(3);
ts.add(2);

Iterator<Integer> it = ts.iterator();

while (it.hasNext()) {
    System.out.println(it.next()); //정렬되어 1->2->3 출력
}
```

**3.Map**
- HashMap<> : 중복된 키로 저장 불가
- Hashtable<> : HashMap과 같은 동작을 함. HashMap 사용 권장
- TreeMap<> : 이진 검색 트리의 형태로 저장하기에 추가/제거 등의 동작이 빠름
```javascript
HashMap<String, Integer> hm = new HashMap<>();

hm.put("1", 1);
hm.put("2", 11);
hm.put("3", 111);
hm.put("4", 1111);

for(String key : hm.keySet())
    System.out.println(key + " " + hm.get(key));

hm.remove("1");

System.out.println("size : " + hm.size());

Iterator<String> keys = hm.keySet().iterator();
while(keys.hasNext()) {
    String key = keys.next();
    System.out.println(key + " " + hm.get(key));
}
```

## 결론
- **List, Set, HashMap를 사용해보기 + Iterator 활용법 익히기**