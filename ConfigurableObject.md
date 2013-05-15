프로그램을 짜다 보면 각종 설정을 조합해 어떤 객체를 만들고, 이를 사용해 작업을 수행해야 하는 경우가 있다.
이때, 꼭 설정해야만 하는 부분과 그렇지 않은 부분이 있거나, 조합에 따라 허용되거나 허용되지 않는 경우가 있을 수도 있다. 예를 들어 원격에서 데이터를 받아 플레이하는 비디오 플레이어를 만든다고 하자. 이때 HTTP로 연결해 파일을 가지고와서 AVC 코덱으로 플레이할 수도 있고, RTSP로 연결해 스트리밍으로 플레이하되 H.264 코덱을 사용할 수도 있을 것이다. 어느경우든 데이터 전송 방식과 코덱은 반드시 설정해야 하지만, 코덱 뒤의 영상 필터는 설정하지 않아도 된다.

라이브러리를 제공하는 경우에도 이런 경우가 많이 있다. 필수 설정 요소는 물론 메뉴얼에 적어두거나 스칼라독 등 소스에 있는 문서 부분에 정보를 추가해 두지만, 시간이 지나면 잊혀지기 마련이다. 

런타임에 체크해야만 하는 경우도 있겠지만, 정적으로 구조를 결정해야 하는 상황이라면 컴파일시 필수적인 설정이 되어 있지 않은 객체는 컴파일 자체가 안된다면 더 좋을 것이다. 

스칼라의 implicit와 일반적 타입(generic type)을 사용해 이런 처리가 가능하다. 또한, @impilcitNotFound 애노테이션을 사용해 오류 메시지에 설정 상태를 표시할 수도 있다. 물론 타입시스템을 사용하기 때문에 아주 자세한(개별 설정 값 수준의) 정보는 보여주기 힘들지만, 설정이 되었는지 여부 정도는 표시할 수 있다.

이 문서에 있는 방법은 [트위터 피네이글](https://github.com/twitter/finagle)에 쓰인 방식에서 핵심부만을 추려서 정리한 것임을 밝혀둔다.

## 들어가면서 필요한 선수 지식들

구체적으로 방법을 알려주기 전에 간단하게 전체를 이해할 때 도움이 될 수 있는 선수 지식을 정리하겠다.

### 일반적 케이스 클래스

일반적 케이스 클래스는 타입인자를 가지고 있는 케이스 클래스이다. 

    case class Foo[T](val v1:T, val v2:Int)

위와 같이 정의하면 스칼라 케이스 클래스가 자동으로 제공하는 도우미 객체를 사용해 `Foo("10",1)`과 같이 바로 객체 생성도 가능하고, `Foo(10,10).copy()`같이 복사생성도 가능하며, `Foo(10,10).copy(v2=100)`과 같이 기존 객체에서 특정 필드만 바꾼 새 객체의 생성도 가능하다.

타입 인자 T가 클래스 내에서 전혀 쓰이지 않을 수도 있다. 다음과 같이 `Bar` 클래스가 있다면,

    case class Bar[T](val v:String)

`Bar("10")`은 `Bar[Nothing]`타입의 객체를 만든다. 만약 `Nothing`이 아닌 다른 타입을 지정하고 싶다면 `Bar[Int]("test")`와 같이 써야 한다.

### 묵시적(implicit) 인자

함수 정의시 묵시 인자를 넣을 수 있다. 이때 생략시 어떤 인자가 생략된 것인지를 명확히 알 수 있어야 하기 때문에, 인자 목록의 뒤에서 부터 차례로 묵시적인 것으로 지정할 수 있다. 

함수 호출시 어떤 묵시적 인자가 생략된 경우에는 이 인자와 타입이 일치하는 값을 컴파일러가 찾아서 넣어줄 수 있어야 한다. 이때 인자를 찾는 방식은 다음과 같다.

1. 해당 호출한 지점에서 억세스 가능한 묵시적이라고 정의된 이름들 또는 전달된 묵시적 매개변수들
2. 위 1에 해당하는 값이 없다면, 묵시적 인자의 타입(클래스)의 도우미 객체에 있는 멤버 중 타입이 일치하는 묵시적이라 명시된 값 

설명이 어려워 이해가 어렵다면 아래 예를 보자.

    abstract class Foo[A] {
      def action(x: A): A
    }

    object Bar {
      
      implicit object StringFoo extends Foo[String] {
        def action(x: String): String = x + x
      }
      
      implicit object IntFoo extends Foo[Int] {
        def action(x: Int): Int = x * x
      }
      
      def mapAction[A](xs: List[A])(implicit m: Foo[A]): List[A] =
        xs.map(m.action(_))

      def main(args: Array[String]) {
        println(mapAction(List("foo","bar")))
        println(mapAction(List(1,3,5,7)))
      }
    }
    
실행해보면 다음과 같은 결과를 얻는다.

    scala> Bar.main(Array())
    List(foofoo, barbar)
    List(1, 9, 25, 49)

## 구현 방법 해설

이제 구현 방법을 설명한다.

### 설정 정보

설정 정보는 `ObjectConfig`라는 타입의 객체에 담는다. 먼저 도우미 객체는 다음과 같이 선언되어 있다.


    object ObjectConfig {

      /** 설정이 된 부분을 나타내기 위한 트레잇
        */
      sealed abstract trait Yes

    }

`Yes`는 설정이 된 부분을 표시하기 위해 사용하는 트레잇으로, 타입 체크시에만 된다.

설정 정보는 다음과 같다.

    /** 설정값과 설정이 되었는지를 타입으로 저장하는 클래스 
      * 타입인자 Config1과 Config2는 각각 config1과 config2가 설정이 되었다면 Yes가 되어야 한다. 
      * 아무 타입 인자를 안주고 생성하면 ObjectConfig[Nothing,Nothing]이 만들어진다는 것을 기억하라.
      */ 
    final case class ObjectConfig[Config1, Config2](
                                                     val config1: Option[String] = None,
                                                     val config2: Option[Int] = None,
                                                     val other: Option[Boolean] = None
                                                     ) {

      import ObjectConfig._

      /** 객체를 맵으로 바꿔주는 메소드 */
      def toMap = Map(
        "config1" -> config1,
        "config2" -> config2,
        "other" -> other
      )

      /** 문자열 표현을 반환 */
      override def toString = {
        "ObjectConfig(%s)".format(
          toMap flatMap {
            case (k, Some(v)) =>
              Some("%s=%s".format(k, v))
            case _ =>
              None
          } mkString (", "))
      }
    }

`ObjectConfig`의 타입 인자 `Config1`과 `Config2`는 실제로는 설정이 되었는지 여부만을 저장하는 플래그 역할을 한다. 
설정 값은 config1, config2, other에 저장된다.

`toMap`은 설정 정보를 맵으로 바꿔주는 메소드로 구현시 어떤헤 이런 류의 함수를 만들 수 있나 보여주기 위해 넣었을 뿐이다.

### 가능한 설정의 조합을 표현하는 트레잇

아래 소스코드를 보면 별 어려움 없이 이해가 되리라 본다.


   /** 가능한 설정 조합을 정의하기 위한 트레잇. implicitNotFound를 사용해 오류 메시지를 이쁘게 표시해준다.
      */
    @implicitNotFound("Object is not fully configured: config1: ${Config1}, config2: ${Config2}")
    private trait PredefinedPossibleConfigurations[Config1, Config2]

    /** 가능한 설정의 조합을 implicit object로 여기 정의해 두면, 타입 체커가 implicit 객체를 찾을 것이다. */
    private object PredefinedPossibleConfigurations {

      /** 두 가지 설정이 다 있는 경우 */
      implicit object FullyConfigured extends PredefinedPossibleConfigurations[ObjectConfig.Yes, ObjectConfig.Yes]

      /** 첫번째 부분은 설정하지 않고, 두번째만 한 경우 */
      implicit object OnlySecondConfigured extends PredefinedPossibleConfigurations[Nothing, ObjectConfig.Yes]

    }


여기서는 두 설정(config1,config2)은 둘 다 있거나(FullyConfigured), 두번째만 있어야 하고(OnlySecondConfigured), 나머지(ObjectConfig의 other)에 대해서는 상관 하지 않는다. 

### 설정에 따라 구성되어야 하는 객체

여기서는 타입체크 부분에만 관심이 있기 때문에, 설정에 따라 실제 객체가 어떻게 구현되는지에 대해서는 자세히 다루지 않을 것이다.

이 객체의 코드는 다음과 같이 시작한다.

    /** 설정에 따라 내부가 바뀔 수 있는 객체
      * 설정은 ObjectConfig안에 있으며, 이 설정값에 따라 build() 함수 내부에서 
      * 실제 써먹을 객체를 만들어 낸다.
      *
      */
    class ConfigurableObject[Config1, Config2](config: ObjectConfig[Config1, Config2]) {

      type This = ConfigurableObject[Config1, Config2]

      override def toString() = "ConfigurableObject(%s)".format(config.toString)

      private def this() = this(new ObjectConfig)

타입 인자 `Config1`과 `Config2`는 각각 `ObjectConfig`의 `config1`과 `config2`가 설정되었음을 표시하기 위한 것이다. 설정여부를 검사해야할 것이 많다면 당연히 이 타입인자의 갯수를 늘리면 된다. `type This`는 타입 표기를 줄이기 위해 정의한 것이며, `toString()`과 `this()`는 각각 이름대로 문자열 표현을 구하는 메소드와 인자가 없는 추가 생성자이다. 한가지 짚고 넘어갈 것은 이 추가생성자 안에서 `new ObjectConfig`를 호출하고, 이는 `ObjectClass[Nothing,Nothing]` 타입의 객체를 만들어낸다는 점이다. 따라서 이 추가생성자에 의해 생기는 객체는 `ConfigurableObject[Nothing,Nothing]`타입이 된다. 이는 두 설정값 모두 아직은 지정되지 않았음을 의미한다.

#### 설정 변경 함수

먼저 설정을 받아 새 ConfigurableObject를 만드는 함수를 정의한다.

      /** 기존의 ConfigurableObject의 설정을 복사한 새 객체를 만든다. 
        */
      protected def copy[Config1_1, Config2_1](
                                                config: ObjectConfig[Config1_1, Config2_1]
                                                ): ConfigurableObject[Config1_1, Config2_1] = {
        new ConfigurableObject(config)
      }

별로 어려운 것은 없다. 단지 인자로 받은 config에 따라 새로 만들어지는 `ConfigurableObject`와 그 안의 `config` 타입이 정해져야 하기 때문에, `copy`가 일반적 메소드로 정의되어 있을 뿐이다.

이제, 이 `copy`를 사용해 설정 값을 지정하는 함수에서 활용할 함수 `withConfig`를 만든다. 어떤 설정 값을 지정하는 함수를 만들 때 그 함수 내에서는 `withConfig`를 호출해서 새로운 설정값이 적용된 `ConfigurableObject`를 만들 것이다.

    /** 이 객체의 설정에 f를 적용해 변경된 설정을 따르는 새 객체를 만든다. 
      * 
      * @param f 설정을 변경해주는 함수
      */
    protected def withConfig[Config1_1, Config2_1](
                                                f: ObjectConfig[Config1, Config2] =>
                                                  ObjectConfig[Config1_1, Config2_1]
                                                ): ConfigurableObject[Config1_1, Config2_1] = copy(f(config))

역시 별다른 것은 없다. `f`는 `ObjectConfig` 객체를 받아서 설정값 중 일부를 변경하면서, 자신이 지정한 설정에 맞게 `Confing1_1`이나 `Config1_2`에 `Yes`가 들어간 타입(예: ObjectConfig[Yes,Nothing])의 객체를 반환하면 된다.  

#### 설정값 지정 함수

이제 필요한 요소들이 다 갖춰졌으므로 설정값을 지정하는 함수를 정의할 수 있다.

    def setConfig1(v: String): ConfigurableObject[ObjectConfig.Yes, Config2] = {
      withConfig(_.copy(config1 = Some(v)))
    }

몇 번째 설정을 변경하느냐에 따라 달라지겠지만, 위 `setConfig1` 함수는 첫번째 설정을 변경하기 때문에, 반환하는 객체가 `ConfigurableObject[ObjectConfig.Yes, Config2]` 타입이 되어야만 한다. `withConfig()`에 넘어가는 무명함수는 인자로 들어온 `ObjectConfig` 객체에 대해 `copy`를 호출하되, `config1`만 `Some(v)`로 변경한다. 앞에서 케이스 클래스에 대해 설명할 때 `Foo(10,10).copy(v2=100)`가 `Foo(10,10)`으로 만들어진 객체에서 `v2`만 `100`으로 지정하는 것이라 말했었다.

두번째 설정을 지정하는 메소드도 앞의 `setConfig1`와 마찬가지 논리이다.

관심의 대상이 아닌 설정값은 자유롭게 설정하면 되고, 당연히 `ConfigurableObject`의 타입 인자값도 변화가 없다. 따라서 다음과 같이 정의할 수 있다.

    /** 세번째 설정값을 적용한다. 타입에는 변화가 없다. */
    def setOther(v: Boolean): This = {
      withConfig(_.copy(other = Some(v)))
    }


#### 설정된 데로 객체 만들어내기

지금까지의 기억을 되살려보자.

1. `ConfigurableObject()`로 만든 객체는 타입이 `ConfigurableObject[Nothing,Nothing]`이다.
2. `ConfigurableObject[V1,V2]` 타입의 객체에 대해 `setConfig1()`을 호출해서 나온 객체의 타입은 `ConfigurableObject[Yes,V2]`가 된다.
3. `ConfigurableObject[V1,V2]` 타입의 객체에 대해 `setConfig2()`을 호출해서 나온 객체의 타입은  `ConfigurableObject[V1,Yes]`가 된다.
4. `ConfigurableObject[V1,V2]` 타입의 객체에 대해 관심의 대상이 아닌 설정값을 지정하는 `setOther()`을 호출하면  나오는 결과 객체는 `ConfigurableObject[V1,V2]` 타입이 그대로 유지된다.

이제 `build`라는 메소드를 만들자. 이 메소드는 어떤 특징을 지녀야 할까?

1. 메소드의 리시버 객체(this)의 타입이 `ConfigurableObject[Yes,Yes]`이거나 `ConfigurableObject[Nothing,Yes]`면 정상적으로 설정에 맞는 결과 객체가 나와야 한다.
2. 메소드의 리시버 객체 타입이 위 1의 경우가 아니라면 컴파일시 오류가 나야 한다. 

위 2번을 만족시키려면, 리시버 객체의 타입이 `ConfigurableObject[Yes,Yes]`이거나 `ConfigurableObject[Nothing,Yes]`인지를 알 수 있어야 한다. _"어떤 타입과 일치하는 뭔가를 찾는다?"_ 아까 앞에서 묵시적 인자에 대해 설명했던 내용을 기억해 보자.

"함수 호출시 어떤 묵시적 인자가 생략된 경우에는 _이 인자와 타입이 일치하는 값을_ 컴파일러가 찾아서 넣어줄 수 있어야 한다. 이때 인자를 찾는 방식은 다음과 같다."

`인자와 타입이 일치하는 값`을 컴파일러가 찾아준다는 이야기가 있다. 따라서 묵시적 인자를 사용하면 되리라는 아이디어를 얻을 수 있다.


    def build()(
      implicit THE_OBJECT_IS_NOT_FULLY_SPECIFIED:
        PredefinedPossibleConfigurations[Config1, Config2]
      ): String = {
        toString(); 
    }

묵시적 인자의 이름은 일부러 설명적인 긴 이름을 만들었다. 묵시적 인자의 타입은 `PredefinedPossibleConfigurations[Config1, Config2]`인데, 여기서 `Config1`과 `Config2`는 바로 리시버 `this`의 타입 `ConfigurableObject[Config1,Config2]`에 있는 두 타입 변수이다. 따라서 어떤 `setConfig?`가 호출되어 왔는지에 따라 여기 묵시적 인자의 `PredefinedPossibleConfigurations` 타입이 바뀌게 되고, 그 타입을 묵시적 인자 찾기 규칙에 따라 `PredefinedPossibleConfigurations` 도우미 객체에서 찾게 된다.

## 전체 소스 코드 

앞에서 설명한 모든 내용을 한 파일에 넣으면 다음과 같다. `ConfigurableObject.scala`로 저장하고, `scalac ConfigurableObject.scala`라는 명령을 내려 컴파일해, `scala ConfigurableObjectTest`라고 실행하면 된다.

만약 맨 아래의 `unconfiguredString` 주변의 코멘트를 제거하고 컴파일을 시도해 보면 implicitNotFound가 발생해 컴파일이 실패로 끝난다.

    import scala.annotation.implicitNotFound
    import scala.collection.mutable

    object ObjectConfig {

      /** 설정이 된 부분을 나타내기 위한 트레잇
        */
      sealed abstract trait Yes

    }

    /** 가능한 설정 조합을 정의하기 위한 트레잇. implicitNotFound를 사용해 오류 메시지를 이쁘게 표시해준다.
      */
    @implicitNotFound("Object is not fully configured: config1: ${Config1}, config2: ${Config2}")
    private trait PredefinedPossibleConfigurations[Config1, Config2]

    /** 가능한 설정의 조합을 implicit object로 여기 정의해 두면, 타입 체커가 implicit 객체를 찾을 것이다. */
    private object PredefinedPossibleConfigurations {

      /** 두 가지 설정이 다 있는 경우 */
      implicit object FullyConfigured extends PredefinedPossibleConfigurations[ObjectConfig.Yes, ObjectConfig.Yes]

      /** 첫번째 부분은 설정하지 않고, 두번째만 한 경우 */
      implicit object OnlySecondConfigured extends PredefinedPossibleConfigurations[Nothing, ObjectConfig.Yes]

    }

    /** 실제 설정을 해야 하는 대상이 될 객체
      */
    object ConfigurableObject {

      /** 객체 생성을 돕기 위한 펙토리 메소드로, 설정이 아무것도 되어있지 않은 ConfigurableObject[Nothing,Nothing]을 만든다. */
      def apply() = new ConfigurableObject()

    }

    /** 설정값과 설정이 되었는지를 타입으로 저장하는 클래스 
      * 타입인자 Config1과 Config2는 각각 config1과 config2가 설정이 되었다면 Yes가 되어야 한다. 
      * 아무 타입 인자를 안주고 생성하면 ObjectConfig[Nothing,Nothing]이 만들어진다는 것을 기억하라.
      */ 
    final case class ObjectConfig[Config1, Config2](
                                                     val config1: Option[String] = None,
                                                     val config2: Option[Int] = None,
                                                     val other: Option[Boolean] = None
                                                     ) {

      import ObjectConfig._

      /** 객체를 맵으로 바꿔주는 메소드 */
      def toMap = Map(
        "config1" -> config1,
        "config2" -> config2,
        "other" -> other
      )

      /** 문자열 표현을 반환 */
      override def toString = {
        "ObjectConfig(%s)".format(
          toMap flatMap {
            case (k, Some(v)) =>
              Some("%s=%s".format(k, v))
            case _ =>
              None
          } mkString (", "))
      }
    }

    /** 설정에 따라 내부가 바뀔 수 있는 객체
      * 설정은 ObjectConfig안에 있으며, 이 설정값에 따라 build() 함수 내부에서 
      * 실제 써먹을 객체를 만들어 낸다.
      *
      */
    class ConfigurableObject[Config1, Config2](config: ObjectConfig[Config1, Config2]) {

      type This = ConfigurableObject[Config1, Config2]

      override def toString() = "ConfigurableObject(%s)".format(config.toString)

      private def this() = this(new ObjectConfig)

      /** 기존의 ConfigurableObject의 설정을 복사한 새 객체를 만든다. 
        */
      protected def copy[Config1_1, Config2_1](
                                                config: ObjectConfig[Config1_1, Config2_1]
                                                ): ConfigurableObject[Config1_1, Config2_1] = {
        new ConfigurableObject(config)
      }

      /** 이 객체의 설정에 f를 적용해 변경된 설정을 따르는 새 객체를 만든다. 
        * 
        * @param f 설정을 변경해주는 함수
        */
      protected def withConfig[Config1_1, Config2_1](
                                                  f: ObjectConfig[Config1, Config2] =>
                                                    ObjectConfig[Config1_1, Config2_1]
                                                  ): ConfigurableObject[Config1_1, Config2_1] = copy(f(config))

      /** 첫번째 설정값을 적용한다
        *
        * @param v 첫번째 설정값
        *
        * @return 이 객체의 설정에서 첫 설정값을 v로 바꾼 새 객체. 
        *         다만 첫번째 값을 설정했기 때문에 이를 표시하기 위해 
        *         타입을 ConfigurableObject[ObjectConfig.Yes, Config2]로 바꿔준다.
        */
      def setConfig1(v: String): ConfigurableObject[ObjectConfig.Yes, Config2] = {
        withConfig(_.copy(config1 = Some(v)))
      }

      /** 두번째 설정값을 적용한다.
        * 
        * @see setConfig1(v: String)
        */
      def setConfig2(v: Int): ConfigurableObject[Config1, ObjectConfig.Yes] = {
        withConfig(_.copy(config2 = Some(v)))
      }

      /** 세번째 설정값을 적용한다. 타입에는 변화가 없다. */
      def setOther(v: Boolean): This = {
        withConfig(_.copy(other = Some(v)))
      }

      /** 사실은 모든 마법은 위 setConfig1, setConfig2에서 타입을 바꿔줬단 것과 
        * 여기 있는 묵시적 인자가 다 해내는 것이다. 묵시적 인자가 있으면 사용자가 명시하지 않은 경우
        * 해당 트레잇의 묵시적 객체 중 타입이 일치하는 것을 찾게 된다.
        *
        * 여기서 PredefinedPossibleConfigurations[Config1, Config2]가 묵시 인자의 타입이기 때문에,
        * this의 Config1과 Config2 타입과 일치하는 PredefinedPossibleConfigurations 객체가 필요하다.
        * 
        * 그런데, 위의 PredefinedPossibleConfigurations 객체 내부에 있는 묵시적 객체는  
        *  [ObjectConfig.Yes, ObjectConfig.Yes]인 경우와 [Nothing, ObjectConfig.Yes]인 경우밖에 없으므로,
        * this가 ConfigurableObject[Nothing,ObjectConfig.Yes]이나 ConfigurableObject[ObjectConfig.Yes,ObjectConfig.Yes]가 
        * 아니라면 implicitNotFound가 발생할 것이다.
        */
      def build()(
        implicit THE_OBJECT_IS_NOT_FULLY_SPECIFIED:
          PredefinedPossibleConfigurations[Config1, Config2]
        ): String = {
          // Config1과 Config2가 다 있을때만 정적으로 문자열을 반환한다.
          // 여기서는 편의상 toString()만을 해 두었지만, 실제로는 
          // 설정시 각 플래그에 해당하는 전략을 선택해 빌더를 만들어 객체를 빌드하거나 
          // 아예 설정에 그런 전략을 넣게 하거나, 팩토리에 설정을 넘겨서 설정 정보에 따라 
          // 객체를 생성하게 만드는 등의 방법이 있을 것이다. 
          toString(); 
      }
    }

    object ConfigurableObjectTest {
      def main(args: Array[String]) {
        // 정상적으로 모든 설정이 된 경우
        val configuredString = ConfigurableObject().
                    setConfig1("Test").
                    setConfig2(10).
                    build()

        // 두번째 설정만 넣는 경우
        val configuredString2 = ConfigurableObject().
                    setConfig2(2).
                    build()
        /*
        // 설정이 부족함을 타입 체크로 잡아서 컴파일이 안되는 경우 
        val unconfiguredString = ConfigurableObject().
                    setConfig1("Test2").
                    build()*/

        println("Configed string with config1+config2 = " + configuredString)
        println("Configed string with config2 only= " + configuredString2)
      }
    }
