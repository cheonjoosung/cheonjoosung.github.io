---
layout: post
title:  "Codeground 화학자의 문장"
date:   2017-08-05
categories: algorithm
highlight: false
image: /images/it/algorithm/algo.jpg
tag: algorithm
---

 `CodeGround 의 화학자의 문장 풀이에 관한 포스팅입니다.` 풀면서 의아했던 점이 왜 Java 는 4초 이내일까 라는 부분이다. 충분히 2초내에 주어진 테스트 케이스를 끝낼 수 있을 것 같기 때문이다. 화학자의 문장은 한글자 체크 두글자 체크해서 모든 예외사항을 체크하지 않으면 100점에 도달하기 힘든 문제였던 것 같다. 처음에 이런 방식으로 풀었는데 99점이 나왔는데 1점에 대한 감이 안잡혀서 다른 방법으로 풀었다.

<br><b>1) `주어진 문제를 풀이`</b><br>
<p>=) 1) 화학 기호를 보면 중복 되는 부분이 많다. B 와 Br , C 와 Cn 등 첫글자가 겹치는 원소기호들이 존재한다.</p>
<p>=) 2) 테스트 인풋값은 소문자로 주어진다. 그렇기에 화학 원소기호를 소문자로 변형해서 변수에 저장하는 것이 좋다.</p>

<br><b>2) `이제 정리된 문제를 코딩으로 옮기기`</b><br>
<p>=) 1) 저는 ArrayList 를 사용해서 주어진 화학기호를 소문자로 변형시켜서 저장하였습니다. ArrayList 의 Contains 를 사용하고자...</p>
<p>=) 2) 주어진 글자가 가능한 여부를 True or False 인가를 체크하기 위해 line.length+1 큰 변수를 생성합니다. 초기값은 True</p>
<p>=) 3) i >= 2, check[i] 에 의미는 이전까지의 글자가 가능여부를 판단하는 것이다.
<p>=) 3-1) 한글자 체크는 Check 배열의 i-1 까지 True 라면 i-2 까지의 모든 글자는 화학기호라는 의미다. i-1 의 word 가 화학기호로 존재한다면 한글자의 화학기호가 존재한다는 의미이다.</p>
<p>=) 3-2) 두글자 체크는 Check 배열의 i-2 까지 True 라면 i-3 까지의 모든 글자는 화학기호라는 의미다.이고 i-2 to i-1 의 word 가 화학기호로 존재한다면 두글자의 화학기호가 존재한다는 의미이다.</p>
<p>=) 3-3) TTTFFT 의 경우는 나올 수 없다. F 가 2번 이상나온다면 그 이후부터는 Check i-1 과 Check i-2 가 F 이기 때문이다. 즉, F가 2번 나오면 그 이후부터는 쭈욱 F다</p>
<p>=) 3-4) TTTFTT 의 경우는 나올 수 있다. i-1 에서 한글자 체크는 안됬지만 i번째에서 2글자 체크가 되었기 때문이다. </p>
<p>=) 3-5) F 가 연속으로 2번 나오면 Return 시키면 시간을 단축할 수 있지만 마지막에서 TT...TTF 의 경우가 나올 수 있다. 그러므로 Answer 의 체크를 마지막 값이 False or True 를 비교하면 된다.</p>

{% highlight ruby %}

static int Answer;

static ArrayList<String> list = new ArrayList<>();

static String [] temp = {
  "H", "He", "Li", "Be", "B", "C", "N", "O", "F", "Ne", "Na", "Mg", "Al",
  "Si", "P", "S", "Cl", "Ar", "K", "Ca", "Sc", "Ti", "V", "Cr", "Mn", "Fe",
  "Co", "Ni", "Cu", "Zn", "Ga", "Ge", "As", "Se", "Br", "Kr", "Rb", "Sr",
  "Y","Zr", "Nb", "Mo", "Tc", "Ru", "Rh", "Pd", "Ag", "Cd", "In", "Sn", "Sb",
  "Te", "I", "Xe", "Cs", "Ba", "Hf", "Ta", "W", "Re", "Os", "Ir", "Pt", "Au",
  "Hg", "Tl", "Pb", "Bi", "Po", "At", "Rn", "Fr", "Ra", "Rf", "Db", "Sg",
  "Bh", "Hs", "Mt", "Ds", "Rg", "Cn", "Fl", "Lv", "La", "Ce", "Pr", "Nd",
  "Pm", "Sm", "Eu", "Gd", "Tb", "Dy", "Ho", "Er", "Tm", "Yb", "Lu", "Ac",
  "Th", "Pa", "U", "Np", "Pu", "Am", "Cm", "Bk", "Cf", "Es", "Fm", "Md",
  "No", "Lr"
};

public static void main(String args[]) throws Exception	{
  Scanner sc = new Scanner(System.in);

  listInit();

  int T = sc.nextInt();

  for(int test_case = 0; test_case < T; test_case++) {
    Answer = 0;

    String line = sc.next();

    line.toLowerCase();

    boolean [] check = new boolean[line.length()+1];

    check[0] = true;
	  check[1] = wordCheck(line.charAt(0)+"");

    for(int i=2 ; i <= line.length() ; i++) {
      check[i] =
        (check[i-1] && wordCheck(line.charAt(i-1)+""))
          || (check[i-2] && wordCheck(line.substring(i-2, i)));
        }

        System.out.println("Case #"+(test_case+1));

  if(check[line.length()]) {
  System.out.println("YES");
  } else {
    System.out.println("NO");
  }
}

public static void listInit() {
  for(String val : temp) {
    val = val.toLowerCase();
    list.add(val);
  }
}

public static boolean wordCheck(String val) {
  if(list.contains(val)) return true;
  else return false;
}

{% endhighlight %}

`코드그라운드 문제 풀이를 공유하는 것이 문제가 되면 바로 삭제하도록 하겠습니다.`
