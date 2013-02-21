# 오일러 프로젝트 문제 풀이 - 2번 문제

## 문제 2

피보나치 수열의 각 항은 바로 앞의 항 두 개를 더한 것이 됩니다. 1과 2로 시작하는 경우 이 수열은 아래와 같습니다.

1, 2, 3, 5, 8, 13, 21, 34, 55, 89, ...
짝수이면서 4백만 이하인 모든 항을 더하면 얼마가 됩니까?

## 풀이(스칼라)

### 일반적 fib함수를 사용한 풀이 

일단 가장 쉬운 것은 재귀 함수에 의한 풀이일 것이다
 
우선, 표준적인 이 문제의 피보나치 함수는 다음과 같이 작성할 수 있다.
```scala
def fib( n: Int ): Int = n match {
    case 0 => 1 
    case 1 => 2
    case _ => fib(n-1) + fib(n-2)
  }
```

이렇게 작성한 fib함수를 사용하면 계산 시간이 무진장 오래 걸릴 것이다. 이를 만회하기 위해 
꼬리재귀를 사용하도록 변경한 fib_with_tail함수는 다음과 같다.

```scala
def fib_with_tail( n: Int ): Int = {
  def fib_tail_rec( n: Int, b: Int, a: Int ): Int = n match {
    case 0 => a 
    case _ => fib_tail_rec( n-1, a + b, b )
  }    
  fib_tail_rec(n, 2, 1) // n==0일때 1, n==1일때 2
}
```

fib()의 값이 400만 이하인 동안 돌면서 결과값을 리스트로 만든다면 될 것이다. 
문제는 fib(n) <= 4000000인 n이 얼마인지를 알려면 fib(n)의 일반항에서 역으로 계산을 해야 한다는 점이다.
(물론 fib(n)의 일반항 구하기는 고등학교 수학 수준이므로 못 풀것은 없다. 아니면 그냥 구글에서 찾아봐도 된다.)

대신 0부터 시작하는 n에 대해 어떤 함수 f를 적용한 리스트를 만들되, f(n)의 값이 4000000이하일 동안만 리스트를 만들면 된다.

이를 위해 먼저 다음과 같은 함수를 만들자. 시작값, 적용함수, 판단함수를 받는 리스트 생성 함수이다.

```scala
def gen_list_with_pred(n:Int, f:Int=>Int, pred:Int=>Boolean):List[Int] = 
  if(pred(f(n))) {
    f(n)::gen_list_with_pred(n+1, f, pred)
  } else {
    Nil
  }
```

이를 사용하면 쉽게 다음과 같이 전체 값을 계산할 수 있다.

1. `fib(n)<4000000` 인 fib(n)의 값 목록 얻기: `gen_list_with_pred(0, fib_with_tail, _<4000000)`
2. 목록에서 짝수만 골라내기: `gen_list_with_pred(0, fib_with_tail, _<4000000).filter(_%2 == 0)`
3. 2에서 얻은 값의 합계 구하기: `gen_list_with_pred(0, fib_with_tail, _<4000000).filter(_%2 == 0).sum`

### 무한 시퀀스를 사용한 풀이

그런데 앞에서 만든 `gen_list_with_pred`가 하는 일을 살펴보면, `0`부터 `pred`를 만족하는 `n`에 대해서만  
`f(n)`의 값을 리스트로 만드는 것이다. 

이를 그대로 `fib`에 적용하면

1. 0부터 시작하는 무한 시퀀스(lazy list)를 얻는다.
2. 원소중에 짝수만을 걸러낸다.
3. 다시 2에서 얻은 무한 시퀀스의 원소 중에 f(i) < 4000000인 i의 f(i)를 모은다.
4. 3에서 모은 리스트를 모두 더한다.

일단 0부터 시작하는 무한 시퀀스를 얻을 수 있어야 한다. 이는 스칼라 스트림을 사용해 간단하게 만들 수 있다.

```scala
def from(i:Int):Stream[Int] = i #:: from(i+1)
```

이제 이 무한 시퀀스를 사용해 무한 피보나치 수열을 얻을 수 있다.
```scala
// 사실은 그냥 from(0).map(fib_with_tail)해도 된다.
val fibs = for( i <- from(0) ) yield fib_with_tail(i)
```

이 무한 피보나치 수열을 필터링하는 것과 그 결과의 합을 구하는 것은 간단하다.

```scala
val fibs_even = fibs filter (_%2 == 0)
val fibs_even_under_4000000 = fibs_even takeWhile(_<4000000)
val fibs_even_under_4000000_sum = fibs_even_under_4000000.sum
```

#### 주의: 왜 굳이 takeWhile을 사용하는걸까? filter를 사용하면 안되나?

위 문제를 잘 살펴보면 리스트 컴프리핸션에서 단순히 from(0)의 원소에 대해 fib_with_tail(i)를 적용한 무한
시퀀스를 얻어서 `takeWhile`로 400만 이하인 값만 취하는 것을 볼 수 있다. 

그렇다면 리스트 컴프리핸션에 조건을 넣을 수는 없는걸까?

```scala
val fibs_under_4000000 = for( i <- from(0) if fib_with_tail(i)<4000000 ) yield fib_with_tail(i)
val fibs_even_under_4000000 = fibs_under_4000000 filter (_%2 == 0)
val fibs_even_under_4000000_sum = fibs_even_under_4000000.sum
```

이렇게 코드를 변경하고, 실행해 보라. 무한루프를 돌 것이다. 그럼 takeWhile 대신 filter를 사용해
다음과 같이 한다면?

```scala
val fibs_even = fibs filter (_%2 == 0)
val fibs_even_under_4000000 = fibs_even filter(_<4000000)
val fibs_even_under_4000000_sum = fibs_even_under_4000000.sum
```
역시 무한루프를 돈다. 게다가 화면에 출력되는 값을 보면 피보나치 값도 음수가 되었다 양수가 되었다 함을
알 수 있다.

음수/양수가 왔다갔다 하는건 전형적인 정수 overflow 증상이다. 이는 데이터형을 변경하거나, 
400만 이하의 함수를 제대로 필터링할 수 있게 되면 문제가 안될 수 있다.

무한루프를 도는 또 다른 이유는 `filter`는 필터를 만족하는 `모든` 원소를 포함하는 스트림을 반환하기 때문이다.
따라서 무한 스트림에 대해 필터를 적용해 유한 스트림이 되더라도, 무한 스트림의 `모든` 원소에 대해 필터 함수를 
검사해야 한다. 따라서 무작적 `filter`를 사용하면 원하는 결과를 얻지 못하는 경우가 생긴다.

이를 방지하기 위해서는 다음에 주의해야 한다.

1. 무한시퀀스를 만들 때 순서가 일정하게(예: 단조증가/단조감소/BFS/DFS등) 하라. 따라서 무한시퀀스를 
생성하는 제너레이터 함수를 만들 때부터 주의깊게 설계해야 한다.
2. 무한 시퀀스에 필터를 적용한 경우에는 그 시퀀스를 사용해 다른 시퀀스를 만드는 것은 문제 없지만(이경우 
다시 무한 시퀀스가 생기므로, 3의 원칙을 사용해 필요한 작업을 하면 된다), sum등 모든 원소를 대상으로 
하나하나 연산을 수행해야만 하는 경우를 피하라.
3. 무한 시퀀스에서 유한 시퀀스를 만들려면, takeWhile과 1의 시퀀스를 사용하거나,
원하는 갯수만큼 원소를 얻어와 사용하거나, 시퀀스 대상으로 이터레이션하면서 
원하는 정보를 다 얻어올 때 까지만 작업을 수행해야 한다.

### 성능 문제
문제는 어떻게든 풀 수 있지만, 성능은 어떨까?

앞의 코드를 다 한데 모은 다음에 fib관련 함수에 print를 넣으면 다음과 같다.
이제 fib관련 함수 내부에서 덧셈이 일어날 때마다 화면에 출력될 것이다.

```scala
def fib( n: Int ): Int = n match {
    case 0 => 1 
    case 1 => 2
    case _ => { println("fib(" + (n-1) + ") + fib(" + (n-2) + ")"); fib(n-1) + fib(n-2) }
  }

def fib_with_tail( n: Int ): Int = {
  def fib_tail_rec( n: Int, b: Int, a: Int ): Int = {
   println("fib_tail_rec(" + n + "," + b +"," + a + ")");
   n match {
     case 0 => a 
     case _ => fib_tail_rec( n-1, a + b, b )
   }    
  }
  fib_tail_rec(n, 2, 1) // n==0일때 1, n==1일때 2
}

def from(i:Int):Stream[Int] = i #:: from(i+1)

// 사실은 그냥 from(0).map(fib_with_tail)해도 된다.
val fibs = for( i <- from(0) ) yield fib_with_tail(i)

val fibs_even = fibs filter (_%2 == 0)

val fibs_even_under_4000000 = fibs_even takeWhile(_<4000000)

val fibs_even_under_4000000_sum = fibs_even_under_4000000.sum
```

실행 결과는 다음과 같다.

```
... 앞부분 생략 ...
fib_tail_rec(2,5702887,3524578)
fib_tail_rec(1,9227465,5702887)
fib_tail_rec(0,14930352,9227465)
fib_tail_rec(34,2,1)
fib_tail_rec(33,3,2)
fib_tail_rec(32,5,3)
fib_tail_rec(31,8,5)
fib_tail_rec(30,13,8)
fib_tail_rec(29,21,13)
fib_tail_rec(28,34,21)
fib_tail_rec(27,55,34)
fib_tail_rec(26,89,55)
fib_tail_rec(25,144,89)
fib_tail_rec(24,233,144)
fib_tail_rec(23,377,233)
fib_tail_rec(22,610,377)
fib_tail_rec(21,987,610)
fib_tail_rec(20,1597,987)
fib_tail_rec(19,2584,1597)
fib_tail_rec(18,4181,2584)
fib_tail_rec(17,6765,4181)
fib_tail_rec(16,10946,6765)
fib_tail_rec(15,17711,10946)
fib_tail_rec(14,28657,17711)
fib_tail_rec(13,46368,28657)
fib_tail_rec(12,75025,46368)
fib_tail_rec(11,121393,75025)
fib_tail_rec(10,196418,121393)
fib_tail_rec(9,317811,196418)
fib_tail_rec(8,514229,317811)
fib_tail_rec(7,832040,514229)
fib_tail_rec(6,1346269,832040)
fib_tail_rec(5,2178309,1346269)
fib_tail_rec(4,3524578,2178309)
fib_tail_rec(3,5702887,3524578)
fib_tail_rec(2,9227465,5702887)
fib_tail_rec(1,14930352,9227465)
fib_tail_rec(0,24157817,14930352)
fibs_even_under_4000000_sum: Int = _6_3___
```

잘 보면 0부터 1씩 커지면서 fib(i)를 계산하는데, 그 앞에서 fib(i)를 계산한 결과를 fib(i+1)에서 활용하지 
않고 있음을 알 수 있다.

어떻게 하면 이를 활용할 수 있을까? 두가지 방법이 있을 수 있다.
한가지 방법은 캐시기법(메모이제이션,memoization)이라는 것으로 함수를 호출시 인자값과 결과를 캐시해 나중에 같은 
값으로 호출되는 경우 재활용하는 방식이고, 다른 한 방법은 스트림을 활용하되 스트림을 만드는 식을 잘 
고안해서 호출 횟수를 최소화하는 것이다.

### 메모이제이션(캐시기법) 활용하기

어떤 함수 `f(args):ReturnType`이 있을 때, 메모이제이션을 하려면 다음과 같이 하면 된다.
더 잘 하려면 스칼라 함수 클래스를 상속해 맵을 추가할 수도 있지만, 이는 연습문제로 남겨둔다.

1. args 타입 => ReturnType인 맵을 하나 정의한다.

```scala
val mamo_f_map = scala.collection.mutable.HashMap.empty[Typeof(args),ReturnType]
```

2. memo_f라는 함수를 다음과 같이 만든다.

```
def memo_f(args):ReturnType = {
  memo_f_map.get(args) match {
    case None => {
      val v = f(args) 
      memo_f_map += (args, v)
      v
    }
    case Some(v) => v
  }
}
```

3. 원래의 f 안에서 f를 재귀호출하는 부분이 있다면 이를 모두 memo_f로 바꾼다.

4. 여기서 f와 memo_f가 상호 재귀호출(mutual recursion)이 된다. 따라서 이 두 함수(그리고 맵)은 한 객체나 클래스 안에 정의되어야 한다.


이에 준해 메모이제이션된 피보나치를 정의(그리고 프린트문 추가)하면 다음과 같다.
```scala
object Foo {
 val memo_fib_map =  scala.collection.mutable.HashMap.empty[Int,Int]
 
 def memo_fib(x:Int):Int = {
   memo_fib_map.get(x) match {
     case None => {
       val v = fib(x) 
       println("add : " + x + "->" + v)
       memo_fib_map += (x -> v)
       v
     }
     case Some(v) => { println("found : " + x + "," + v);v }
   }
 }
 
 def fib( n: Int ): Int = n match {
     case 0 => 1 
     case 1 => 2
     case _ => { println("fib(" + (n-1) + ") + fib(" + (n-2) + ")"); memo_fib(n-1) + memo_fib(n-2) }
   }
}
```

이제 이를 사용해 앞의 fibs_even_under_4000000_sum를 풀어보면 함수 호출 횟수가 확 줄어들 수 있다.

```scala
def from(i:Int):Stream[Int] = i #:: from(i+1)

// 사실은 그냥 from(0).map(Foo.memo_fib)해도 된다.
val fibs = for( i <- from(0) ) yield Foo.memo_fib(i)

val fibs_even = fibs filter (_%2 == 0)

val fibs_even_under_4000000 = fibs_even takeWhile(_<4000000)

val fibs_even_under_4000000_sum = fibs_even_under_4000000.sum
```

실행 결과는 다음과 같다.

```
scala> def from(i:Int):Stream[Int] = i #:: from(i+1)
from: (i: Int)Stream[Int]

scala> // 스트림을 만들 때 맨 첫 원소는 평가함을 알 수 있다.
val fibs = for( i <- from(0) ) yield Foo.memo_fib(i)
add : 0->1
fibs: scala.collection.immutable.Stream[Int] = Stream(1, ?)

scala> // 필터를 하면 필터를 만족하는 맨 처음 원소까지는 값을 평가함을 알 수 있다.
val fibs_even = fibs filter (_%2 == 0)    
add : 1->2
fibs_even: scala.collection.immutable.Stream[Int] = Stream(2, ?)

scala> val fibs_even_under_4000000 = fibs_even takeWhile(_<4000000)
fibs_even_under_4000000: scala.collection.immutable.Stream[Int] = Stream(2, ?)

scala> val fibs_even_under_4000000_sum = fibs_even_under_4000000.sum
fib(1) + fib(0)
found : 1,2
found : 0,1
add : 2->3
fib(2) + fib(1)
found : 2,3
found : 1,2
add : 3->5
fib(3) + fib(2)
found : 3,5
found : 2,3
add : 4->8
fib(4) + fib(3)
found : 4,8
found : 3,5
add : 5->13
fib(5) + fib(4)
found : 5,13
found : 4,8
add : 6->21
fib(6) + fib(5)
found : 6,21
found : 5,13
add : 7->34
fib(7) + fib(6)
found : 7,34
found : 6,21
add : 8->55
fib(8) + fib(7)
found : 8,55
found : 7,34
add : 9->89
fib(9) + fib(8)
found : 9,89
found : 8,55
add : 10->144
fib(10) + fib(9)
found : 10,144
found : 9,89
add : 11->233
fib(11) + fib(10)
found : 11,233
found : 10,144
add : 12->377
fib(12) + fib(11)
found : 12,377
found : 11,233
add : 13->610
fib(13) + fib(12)
found : 13,610
found : 12,377
add : 14->987
fib(14) + fib(13)
found : 14,987
found : 13,610
add : 15->1597
fib(15) + fib(14)
found : 15,1597
found : 14,987
add : 16->2584
fib(16) + fib(15)
found : 16,2584
found : 15,1597
add : 17->4181
fib(17) + fib(16)
found : 17,4181
found : 16,2584
add : 18->6765
fib(18) + fib(17)
found : 18,6765
found : 17,4181
add : 19->10946
fib(19) + fib(18)
found : 19,10946
found : 18,6765
add : 20->17711
fib(20) + fib(19)
found : 20,17711
found : 19,10946
add : 21->28657
fib(21) + fib(20)
found : 21,28657
found : 20,17711
add : 22->46368
fib(22) + fib(21)
found : 22,46368
found : 21,28657
add : 23->75025
fib(23) + fib(22)
found : 23,75025
found : 22,46368
add : 24->121393
fib(24) + fib(23)
found : 24,121393
found : 23,75025
add : 25->196418
fib(25) + fib(24)
found : 25,196418
found : 24,121393
add : 26->317811
fib(26) + fib(25)
found : 26,317811
found : 25,196418
add : 27->514229
fib(27) + fib(26)
found : 27,514229
found : 26,317811
add : 28->832040
fib(28) + fib(27)
found : 28,832040
found : 27,514229
add : 29->1346269
fib(29) + fib(28)
found : 29,1346269
found : 28,832040
add : 30->2178309
fib(30) + fib(29)
found : 30,2178309
found : 29,1346269
add : 31->3524578
fib(31) + fib(30)
found : 31,3524578
found : 30,2178309
add : 32->5702887
fib(32) + fib(31)
found : 32,5702887
found : 31,3524578
add : 33->9227465
fib(33) + fib(32)
found : 33,9227465
found : 32,5702887
add : 34->14930352
fibs_even_under_4000000_sum: Int = 4______

scala>
```

### 스트림 활용하기

앞에서는 `from(0)`을 사용해 정수 시퀀스를 만들고, 이 정수 시퀀스에 `fib()`함수를 적용해 무한 피보나치 
시퀀스를 만들었다. 하지만, 직접 피보나치 시퀀스를 만들지 못할 이유가 어디 있을까?

우선 무식한 피보나치 수열 함수를 보자.
```scala
def fib( n: Int ): Int = n match {
    case 0 => 1 
    case 1 => 2
    case _ => fib(n-1) + fib(n-2)
  }
```

이를 스트림을 만드는 함수 fibs로 바꾼다고 생각해 보자. 우선 반환타입이 `Stream[Int]`가 되어야 한다. 
그러면 반환값 타입에 맞춰 각 `case`의 뒤에 적당한 스트림을 붙여주면 된다.

```scala
def fibs( n: Int ): Stream[Int] = n match {
    case 0 => 1 #:: fibs(1)
    case 1 => 2 #:: fibs(2)
    case _ => (fibs(n-1) + fibs(n-2)) #:: fibs(n+1)
  }
```

일단 이렇게 바꾸고 잘 되나 보자.

```scala
scala> def fibs( n: Int ): Stream[Int] = n match {
     |     case 0 => 1 #:: fibs(1)
     |     case 1 => 2 #:: fibs(2)
     |     case _ => (fibs(n-1) + fibs(n-2)) #:: fibs(n+1)
     |   }
<console>:10: error: type mismatch;
 found   : Stream[Int]
 required: String
           case _ => (fibs(n-1) + fibs(n-2)) #:: fibs(n+1)
                                      ^
```

앗! fibs(n-1)이나 fibs(n-2)는 스트림이기 때문에, 직접 사용할 수가 없다. 이를 각 스트림의 첫 원소를 가져오게
바꾸자.

```scala
def fibs( n: Int ): Stream[Int] = n match {
    case 0 => 1 #:: fibs(1)
    case 1 => 2 #:: fibs(2)
    case _ => (fibs(n-1).head + fibs(n-2).head) #:: fibs(n+1)
  }
```

이제 컴파일이 잘 된다. 이런저런 장난을 해보면 잘 됨을 볼 수 있다.

```scala
scala> lazy val fibss = fibs(0)
fibss: Stream[Int] = <lazy>

scala> fibss(32)
res4: Int = 5702887

scala> fibss
res5: Stream[Int] = Stream(1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610
, 987, 1597, 2584, 4181, 6765, 10946, 17711, 28657, 46368, 75025, 121393, 196418
, 317811, 514229, 832040, 1346269, 2178309, 3524578, 5702887, ?)

scala> fibss.takeWhile(_<4000000).filter(_%2==0).sum
res6: Int = 4______
```

성능 체크를 위해 print를 넣어보면, 결과가 참혹하다 내 PC(intel core-i5 2400 3.2GHz)에서 10:55분에 시작해서,
4분 정도가 걸렸다.

```scala
def fibs( n: Int ): Stream[Int] = n match {
    case 0 => 1 #:: fibs(1)
    case 1 => 2 #:: fibs(2)
    case _ => { println("fibs:" + (n-1) + "+" + (n-2)); (fibs(n-1).head + fibs(n-2).head) #:: fibs(n+1) }
  }
```

이를 개선할 방법은? 일단 소극적으로는 fibs(n-1).head = fibs(n-2).tail.head라는 점을 이용해 
스크림을 공유해 보는 것이다. 해보면 약간 속다가 빨라지긴 하지만, 그렇게 실용적이지는 않아 보인다.

```scala
def fibs2( n: Int ): Stream[Int] = n match {
    case 0 => 1 #:: fibs2(1)
    case 1 => 2 #:: fibs2(2)
    case _ => { 
       lazy val fibs_n_2 = fibs(n-2)
       println("fibs:" + (n-1) + "+" + (n-2)); (fibs_n_2.tail.head + fibs_n_2.head) #:: fibs(n+1) 
    }
  }
```

다시 좀 더고민해 보면? 어차피 fibs2(0)이란 무한수열이 있다면, 그 뒤의 fibs(0)(n-1)등은 그냥 얻을 수 있다.
이를 잘 생각해 보면 다음과 같이 쓸 수 있다.

```scala
lazy val fibs3: Stream[Int] = {
    def f(x:Int) : Stream[Int] =
      if(x==0) 1 #:: f(1)
      else if(x==1) 2 #:: f(2)
      else (fff(x-1)+fff(x-2))#::f(x+1)
    f(0)
  }

lazy val fibs3_log: Stream[Int] = {
    def f(x:Int) : Stream[Int] =
      if(x==0) 1 #:: f(1)
      else if(x==1) 2 #:: f(2)
      else ({ println("fibs:" + (x-1) + "+" + (x-2));; fff(x-1)+fff(x-2)})#::f(x+1)
    f(0)
  }  
```

로그를 넣은 버전으로 돌려보면, 결과는 놀랍다. 잘 설계한 스트림은 메모이제이션 효과를 절로 가져오게 된다.

이제 스트림 설계에 있어 일반론을 이야기할 수 있을 것이다.

1. 원래 i번째 원소를 계산하는 함수 f가 있다면, 이를 Stream[Retur을 반환하는 함수로 변경한다. 일단은 함수를 그대로 복사해서 
값을 반환할 때마다 
2. 





--------------------------
아래는 원 사이트(http://euler.synap.co.kr)의 카피라이트이다. 이에 따라 이 페이지의 라이센스도
역시 Creative Commons License: Attribution-NonCommercial-ShareAlike 2.0 UK: England & Wales를
따른다. 원 사이트의 원 사이트 –;; 는 http://projecteuler.net/이다.

Copyright Information

이곳에 실린 문제들은 사이트에 영문으로 수록된 것을 한글로 번역한 것입니다.
원본 문제의 저작권은 Copyright Information 페이지에 설명된 것처럼
Creative Commons License: Attribution-NonCommercial-ShareAlike 2.0 UK: England & Wales 를 따르고 있습니다.

한글 번역본은 원본의 파생 저작물(derivative work)로 볼 수 있으므로,
위 Creative Commons License의 내용에 따라 동일한 (Share Alike) 라이선스를 따릅니다.
라이선스의 내용을 간단히 요약하면 아래와 같습니다.
복사, 배포, 전시 및 파생 저작물을 만들기 위한 행위가 허용됩니다.

그에 따르는 조건은,
Attribution – 원 저작자에 대한 정보를 제공해야 합니다.
Non-Commercial – 상업적인 목적으로 이용할 수 없습니다.
Share Alike – 만약 이 저작물의 내용을 변경, 변형하거나 이를 기반으로 파생 저작물을 만든 경우 본 라이선스와 동일한 라이선스를 적용해야 합니다.

자세한 내용은 해당 라이선스의 요약문 및 라이선스 전문을 참고하시기 바랍니다.
