플레이 폼 처리시 정의하는 객체로 `play.api.data.Form`이 있다. 이에 대해 더 자세히 살펴보자.

먼저 사용방법을 살펴보고 그 후 내부 구현을 간략하게 들여다 볼 것이다.
이 부분을 정리하고 나면 플레이 프레임워크에서 좀 더 폼을 잘 활용하게 될 것이라 기대한다.
(글쓴이도 소스를 분석하며 쓰는 글이라서 두서가 없을 것이다... 향후 시간이 나면 다듬을 것을 
약속한다.. 물론 생계가 걸린 일들이 발목을 잡을 수 있기 때문에, 장담은 못한다.)

# 모델 객체로 변환이 필요하지 않을 떄

특별히 모델 객체는 필요없고, 등록된 폼에서 값을 가져오거나, 역으로 폼에 값을 집어넣고 싶을 때가 있을 것이다.

가장 기본적인 사용방법은 튜플을 사용하는 것이다. 간단하게 2-튜플을 사용하는 예를 살펴보자.
예를 들자면 로그인 페이지에서 `id`와 암호를 받는 경우가 될 수 있다.

```scala
import play.api.data._
import play.api.data.Forms._

val loginForm = Form(
  tuple(
    "userid" -> text,
    "password" -> text
  )
)
```

이제 `Form`객체가 있으면 다양한 작업을 할 수 있다. `Form`이 제공하는 기본적인 메소드로는 
다음과 같은 것이 있다.

1.폼에 데이터를 넣을 때 사용하는 메소드들
  1. `bind(data:JsValue):Form[T]` : json값(JsValue)으로 폼을 바인딩할 때 사용한다.
  2. `bind(data:Map[String,String]):Form[T]` : 맵으로 폼을 바인딩할 때 사용한다.
  3. `bindFromRequest()(implicit request: play.api.mvc.Request[_])` : HTTP 요청을 가지고 폼 값을 채워 넣는다.
  4. `fill(value:T):Form[T]` : 기존 객체로 폼 값을 채워 넣는다. 예를 들어 모델 객체를 가지고 수정하는 폼의 경우 이 메소드를 활용할 수 있다.
  5. `fillAndValidate(value:T)` : `fill(value:T)`와 같이 폼 값을 기존 객체로 채워 넣되, 입력 값을 검증한다.
2. 데이터를 얻어오는 메소드들
  1. `apply()` : 키 값(또는 필드 이름)을 가지고 필드를 가져온다.
  2. `get()` : `Form[T]`에서 `T` 타입의 객체를 가져온다. 바인딩이 안된 경우 예외(`NoSuchElementException`)를 던진다.
  3. `value()` : 위의 `get()`과 비슷한 역할을 하지만, `Option[T]` 타입을 반환하기 때문에, 바인딩이 안된 경우 `None`을 돌려준다.
3. `Form(mapping: Mapping[T], data: Map[String, String], errors: Seq[FormError], value: Option[T])` 생성자: `Form`은 케이스 클래스이다.
내부에 `mapping`(폼 필드를 표현), `data`(폼 내의 필드 값의 이름과 값을 저장), `errros`(폼의 오류들), `value`(폼 등록 성공시 T 타입의 객체를 만들어서 저장)한다.

일단 이렇게 폼 객체를 만들면 여러가지 조작이 가능하다. 예를 들어 위의 `loginForm`에 맵을 바인딩해보자.

```scala
val testData = Map("userid" -> "manse", "password" -> "amho")
val boundedForm = loginForm.bind(testData)
val (userid, password) = boundedForm.get
val vv = boundedForm.value
```

웹 애플리케이션인 경우 `bindFromRequest()`를 사용해 쉽게 폼에 POST나 GET 요청으로 들어온 필드 값을 
넣을 수 있다. 이때 `bindFromRequest`의 인자 `request`는 `implicit request: play.api.mvc.Request[_]`로 
되어 있기 때문에, 직접 넘기지 않아도 프레임워크에서 받아와서 사용 가능하다.

## `Form` 생성 자세히 들여다 보기

폼의 생성자를 직접 사용하는 일은 별로 없다. 폼 이란 것이 데이터를 바인딩하기 위한 구조이기 때문에, 
보통은 생성자의 정보 중 `mapping` 정보만을 채워 넣으면 폼을 처리할 준비가 된 것이다. 
그리고 나서 필요에 따라 `data`를 바인딩하거나, 거꾸로 `value`를 채워 넣는 방식으로 처리가 진행될 것이다.

따라서 `Form` 객체를 생성하는 것은 우선은 `mapping`을 생성하는 것이 주가 될 수 밖에 없다. `mapping`이야 말로 
HTML 폼과 모델 객체 사이의 변환을 위한 스펙이라 할 수 있다.

스칼라에서 어떤 클래스 정의를 보던지, 가장 먼저 봐야 할 것 중 하나는 그 클래스의 컴패니언 객체일 것이다. 
`Form[T]` 클래스에도 역시 컴패니언 객체가 있다. 소스코드는 
`$PLAY_HOME/framework/src/play/src/main/scala/play/api/data/Form.scala`에 있다. 

```scala
/**
 * `Form` 값을 생성하기 위한 연산을 정의한다
 */
object Form {

  /**
   * 매핑을 가지고 새 폼을 만든다
   *
   * 예:
   * {{{
   * import play.api.data._
   * import format.Formats._
   *
   * val userForm = Form(
   *   tuple(
   *     "name" -> of[String],
   *     "age" -> of[Int],
   *     "email" -> of[String]
   *   )
   * )
   * }}}
   *
   * @param mapping form에서 사용할 매핑
   * @return 폼 정의
   */
  def apply[T](mapping: Mapping[T]): Form[T] = Form(mapping, Map.empty, Nil, None)

  /**
   * 매핑을 가지고 새 폼을 만들되, 루트 키를 지정한다.
   *
   * 예:
   * {{{
   * val userForm = Form(
   *   "user" -> tuple(
   *     "name" -> of[String],
   *     "age" -> of[Int],
   *     "email" -> of[String]
   *   )
   * )
   * }}}
   *
   * @param (루트 키, 매핑) 연관을 가진 매핑
   * @return 폼 정의
   */
  def apply[T](mapping: (String, Mapping[T])): Form[T] = Form(mapping._2.withPrefix(mapping._1), Map.empty, Nil, None)

}
```

컴패니언 객체에 `apply()` 메소드가 정의되어 있다(참고로, 클래스 정의 안에는 필드 억세스를 위한 `apply()`
메소드가 있다.) 두 가지 메소드가 있는데, 차이는 루트키를 정의하느냐 마느냐이다
(이 부분은 나중에 기회가 되면 설명할 것이다). 이를 제외하고 나면, 예상했던 대로 `mapping`만 정의하고, 
나머지는 `Map.empty`, `Nil`, `None`등을 넘겨서 `Form`객체를 초기화해 준다.

다시, loginForm을 만들었던 기억을 되살려보자.

```scala
import play.api.data._
import play.api.data.Forms._

val loginForm = Form(
  tuple(
    "userid" -> text,
    "password" -> text
  )
)
```

이제 `Form.apply()`가 어떻게 정의되어 있는지를 알고 있으므로, 쉽게 `tuple`이라는 것이 매핑을 생성해주는
메소드 또는 함수라는 사실을 유추할 수 있다. 이제 `tuple`을 살펴보자.

## `tuple()` 들여다 보기

`tuple`을 보려니, 반환하는 값이 `Mapping[T]`이다. 이를 먼저 살펴보자.

### `Mapping[T]`에 대해

이제 `Form[T]`의 생성자에 정의된 `mapping`의 타입 `Mapping[T]` 타입이름을 보면 
딱 "T 타입으로의 맵핑이구나" 하는 감이 온다. 따라서, 아마도 Mapping[T]는 필드 이름과 필드 값을 받아서 
T 타입의 값을 만들어내주는 객체가 되리라 상상할 수 있다. 그것으로 충분할까? 아니다. 조금 더 생각해보자.

`Form[T]`에 있는 `bind`류의 함수들은 필드이름,필드값을 받아서 `T` 값을 만들어낼 수 있다면 충분히 
구현 가능할 것이다. 그렇지만, `fill`류의 함수들은 거꾸로 `T`에서 필드 값을 만들어 줘야 한다. 따라서 
`Mapping[T]`는 단순히 한방향(필드이름, 값의 맵 => `T` 타입의 값)만 지원해서는 안되고, 그 역방향(`T` 타입의 값 => 필드이름, 값의 맵)도 
지원해 주어야 함을 알 수 있다.

여기서 조금 더 나아가 생각해보면, `Form[T]`에는 입력값의 적합성 검사와 관련한 어떤 정보도 들어가 있지 
않으므로, `Mapping[T]`에 이런 적합성 검사가 들어가야 함을 예상할 수 있다. 그리고, 매핑이 변환을 담당해준다면,
적함성 검사는 변환에서 처리해 주는 것이 맞으므로 이는 타당하다 할 수 있다.

자, 그럼 한번 `Mapping[T]` 코드를 살펴보자. 역시 `Form[T]`와 같은 파일(`Form.scala`)에 정의되어 있다.

```scala

/**
 * 매핑은 폼 필드를 처리하기 위한 양방향 바인더이다
 */
trait Mapping[T] {
  self =>

  /**
   * 필드 키
   */
  val key: String

  /**
   * 하위 매핑(하위키로 볼 수도 있다)
   */
  val mappings: Seq[Mapping[_]]

  /**
   * 이 필드의 형식(지정하지 않을 수도 있음)
   */
  val format: Option[(String, Seq[Any])] = None

  /**
   * 이 필드의 제약사항
   */
  val constraints: Seq[Constraint[T]]

  /**
   * 필드를 바인드한다. 즉, 제출된 데이터를 가지고 구체적인 값을 만든다.
   *
   * @param data 제출한 데이터
   * @return 바인딩 성공시 타입 `T`의 구체적 값. 바인딩 실패시에는 오류 집합을 반환함.
   */
  def bind(data: Map[String, String]): Either[Seq[FormError], T]

  /**
   * 필드를 언바인드 한다. 즉, 구체적 값을 가지고 필드 값을 채워넣는다.
   *
   * @param value 언바인드할 T 타입의 값
   * @return 필드 이름,값의 맵(성공한 경우)이거나 오류 정보(언바인드가 실패한 경우)
   */
  def unbind(value: T): (Map[String, String], Seq[FormError])

  /**
   * 이 매핑을 기반으로 하되, 키에만 앞에 접두사를 붙인 새로운 매핑을 만든다
   *
   * @param prefix 키 앞에 붙일 접두사
   * @return 키만 접두사가 붙어서 바뀐 새로운 매핑
   */
  def withPrefix(prefix: String): Mapping[T]

  /**
   * 이 매핑에 새로운 제약사항을 추가한 매핑을 만든다
   *
   * 예:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying(required) )
   * }}}
   *
   * @param constraints 추가할 제약사항들
   * @return 새로운 매핑
   */
  def verifying(constraints: Constraint[T]*): Mapping[T]

  /**
   * 이 매핑에 새로운 임의의 제약사항을 추가한 매핑을 만든다
   *
   * 예:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying {_.grouped(2).size == 5})
   * }}}
   *
   * @param constraint T 타입의 값을 받아서 검증 실패시 `false`를 반환하는 함수
   * @return the new mapping
   */
  def verifying(constraint: (T => Boolean)): Mapping[T] = verifying("error.unknown", constraint)

  /**
   * 이 매핑에 새로운 임의의 제약사항을 추가한 매핑을 만든다
   *
   * 예:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying("Bad phone number", {_.grouped(2).size == 5}))
   * }}}
   *
   * @param error 검증 실패시 표시할 오류 메시지
   * @param constraint T 타입의 값을 받아서 검증 실패시 `false`를 반환하는 함수
   * @return the new mapping
   */
  def verifying(error: => String, constraint: (T => Boolean)): Mapping[T] = {
    verifying(Constraint { t: T =>
      if (constraint(t)) Valid else Invalid(Seq(ValidationError(error)))
    })
  }

  /**
   * Mapping[T]를 Mapping[B]로 변환함
   *
   * @tparam B 새 매핑에서 사용할 구체적 값의 타입
   * @param f1 T 타입의 값을 B 타입의 값으로 변환할 때 사용하는 함수
   * @param f2 B 타입의 값을 T 타입의 값으로 변환할 때 사용하는 함수
   */
  def transform[B](f1: T => B, f2: B => T): Mapping[B] = WrappedMapping(this, f1, f2)

  // 내부 도우미 함수들

  protected def addPrefix(prefix: String) = {
    Option(prefix).filterNot(_.isEmpty).map(p => p + Option(key).filterNot(_.isEmpty).map("." + _).getOrElse(""))
  }

  protected def applyConstraints(t: T): Either[Seq[FormError], T] = {
    Right(t).right.flatMap { v =>
      Option(collectErrors(v)).filterNot(_.isEmpty).toLeft(v)
    }
  }

  protected def collectErrors(t: T): Seq[FormError] = {
    constraints.map(_(t)).collect {
      case Invalid(errors) => errors.toSeq
    }.flatten.map(ve => FormError(key, ve.message, ve.args))
  }

}
```

앞에서 예상했던 기능이 어디 정의되어 있나 보자. 매핑과 `T` 타입의 값 사이의 변환은 `bind`와 `unbind`가 담당한다. 
그리고, 제약사항 추가는 `verifying()` 메소드가 담당한다. 

### `tupple()`에 대해

이제 `tupple()`를 보자. `$PLAY_HOME/framework/src/play/src/main/scala/play/api/data/Forms.scala`에 
정의되어 있는 `Forms`객체의 메소드이다.

```scala
  def tuple[A1, A2](a1: (String, Mapping[A1]), a2: (String, Mapping[A2])): Mapping[(A1, A2)] = 
  mapping(a1, a2)((a1: A1, a2: A2) => (a1, a2))((t: (A1, A2)) => Some(t))
```

타입을 잘 보면, `tuple`의 결과는 `Mapping[(A1,A2)]`이다. 따라서, `(A1,A2)`라는 튜플 타입을 폼 바인딩시 
생기는 구체적 값으로 사용한다는 것을 알 수 있다. 그리고, 이 메소드의 몸체를 잘 보면, `mapping()`을 
부른다. 이 함수는 `Forms.scala`에 정의되어 있다.

```scala
  def mapping[R, A1, A2](a1: (String, Mapping[A1]), a2: (String, Mapping[A2]))(apply: Function2[A1, A2, R])(unapply: Function1[R, Option[(A1, A2)]]): Mapping[R] = {
    ObjectMapping2(apply, unapply, a1, a2)
  }
```

이 함수는 `A1` 타입의 값, `A2` 타입의 값, `apply`, `unapply`를 받는 `ObjectMapping2` 객체를 만든다. 
결국 `ObjectMapping2`가 무언가 필요한 일은 다 하겠지만 이 부분은 나중에 따로 떼어서 살펴 보자.

이제 `tuple` 메소드안에 `mapping`을 호출하는 부분을 살펴보면, `A1`, `A2` 타입은 두 인자(모두 튜플들이다)의 
두번째 인자인 `Mapping[A1]`과 `Mapping[A2]`에서 오며, `apply`는 항등 함수이고, `unapply`는 값을 `Some`으로 
둘러싸기만 함임을 알 수 있다.

이제, 구체적인 예를 들여보자. 앞에서 `tuple("userid" -> text,"password" -> text)`라는 메소드호출을 
생각해 보면, `text`라는 것이 아마도 `Mapping[String]` 타입일 것이란 추측을 할 수 있다. 그에 따라, 
`Form(tuple("userid" -> text,"password" -> text))`으로 만들어진 폼은 `(String,String)` 튜플을 구체적 
대상 값으로 사용하고, "userid"와 "password"에 각각 `String` 타입의 값을 바인딩하게 될 것이다.

한번 `text`를 `Forms.scala`에서 찾아보면,

```scala
val text: Mapping[String] = of[String]
```

이다. 여기서 `of[]`를 찾아보면 다음과 같다.

```scala
  /**
   * 타입 `T`의 매핑을 만든다.
   *
   * 예:
   * {{{
   * Form("email" -> of[String])
   * }}}
   *
   * @tparam T 매핑 타입
   * @return 단순 필드 매핑
   */
  def of[T](implicit binder: Formatter[T]): FieldMapping[T] = FieldMapping[T]()(binder)
```

다시 `FieldMapping[T]`를 살펴보면 다음과 같다.

```scala
/**
 * 단일 필드 매핑
 *
 * @param key 필드의 키
 * @param constraints 이 필드와 관련된 제약사항들
 */
case class FieldMapping[T](val key: String = "", val constraints: Seq[Constraint[T]] = Nil)(implicit val binder: Formatter[T]) extends Mapping[T] {

  /**
   * 이 필드의 형식(지정하지 않을 수도 있음)
   */
  override val format: Option[(String, Seq[Any])] = binder.format

  /**
   * 이 매핑에 새로운 제약사항을 추가한 매핑을 만든다
   *
   * 예:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying(required) )
   * }}}
   *
   * @param constraints 추가할 제약사항
   * @return 새 매핑
   */
  def verifying(addConstraints: Constraint[T]*): Mapping[T] = {
    this.copy(constraints = constraints ++ addConstraints.toSeq)
  }

  /**
   * 이 필드를 처리하는 데 사용되는 바인더를 바꾼다.
   *
   * @param binder 변경할 새 바인더
   * @return 바인더만 바꾼 새 매핑
   */
  def as(binder: Formatter[T]): Mapping[T] = {
    this.copy()(binder)
  }

  /**
   * 필드를 파인드한다. 즉 제출된 데이터를 가지고 구체적 값을 만든다.
   *
   * @param data 제출된 데이터
   * @return 바인딩 성공시 타입 `T`의 구체적 값. 바인딩 실패시에는 오류 집합을 반환함.
   */
  def bind(data: Map[String, String]): Either[Seq[FormError], T] = {
    binder.bind(key, data).right.flatMap { applyConstraints(_) }
  }

  /**
   * 필드를 언바인드 한다. 즉, 구체적 값을 가지고 필드 값을 채워넣는다.
   *
   * @param value 언바인드할 값
   * @return 필드 이름,값의 맵(성공한 경우)이거나 오류 정보(언바인드가 실패한 경우)
   */
  def unbind(value: T): (Map[String, String], Seq[FormError]) = {
    binder.unbind(key, value) -> collectErrors(value)
  }

  /**
   * 이 매핑을 기반으로 하되, 키에만 앞에 접두사를 붙인 새로운 매핑을 만든다
   *
   * @param prefix 키 앞에 붙일 접두사
   * @return 키만 접두사가 붙어서 바뀐 새로운 매핑
   */
  def withPrefix(prefix: String): Mapping[T] = {
    addPrefix(prefix).map(newKey => this.copy(key = newKey)).getOrElse(this)
  }

  /** 하위 매핑(하위키로 볼 수도 있다) */
  val mappings: Seq[Mapping[_]] = Seq(this)

}
```

이제 `of`에서 `class FieldMapping[T]`로 전달한 인자인 `binder`를 찾아보면 될 것이다. 
`binder`는 `Formatter[T]` 타입으로 정의되어 있다. 그런데, 
`text = of[String](implicit binder: Formatter[String]): FieldMapping[String] = FieldMapping[String]()(binder)`이다.
따라서, `Formatter[String]`으로 정의된 값을 찾으면 된다.  `Form.scala`내부에는 정의된 값이 없고, 
`Form.scala`내부에서 임포트한 모듈 중 어딘가에 있을 것이다. `import format._`이 있으니, 
`play.api.data.format` 패키지를 뒤져봐야 한다.

다행히(?) `$PLAY_HOME/framework/src/play/src/main/scala/play/api/data/format/Format.scala`에 
있는 `Formats` 객체에 정의되어 있다. 소스를 보기 전에 여러분이라면 `bind`와 `unbind`를 어떻게 만들지 
생각해 보라. 위 `FieldMapping`에서 `bind`와 `unbind`를 사용하는 부분을 참조하면 쉽게 예상 가능할 것이다.

```scala
  /**
   * `String` 타입의 기본 Formatter
   */
  implicit def stringFormat: Formatter[String] = new Formatter[String] {
    def bind(key: String, data: Map[String, String]) = data.get(key).toRight(Seq(FormError(key, "error.required", Nil)))
    def unbind(key: String, value: String) = Map(key -> value)
  }

```

필드 매핑의 경우 `bind`는 `data` 맵에서 `key`에 해당하는 값을 꺼내서 반환(이때 
`Either[Seq[FormError], String]` 타입의 값을 반환해야 함)해야 한다. 위 코드의 `bind` 메소드를 
살펴보면:

1. `data.get(key)` : `data` 맵에서 `key`에 대응하는 값을 가져온다. `Option[String]`을 반환한다.
2. `data.get(key).toRight(Seq(FormError(key, "error.required", Nil)))` : 앞에서 반환한 `Option`의 `toRight`메소드를 호출한다. 
toRight()는 만약 `Option`이 `Some`이면 `Some(x)`의 `x`를 `Right(x)`로 만들고, `None`이면 `Seq(FormError(key, "error.required", Nil))`를 `Left`로 만든다.

스칼라에서 `Either`를 사용하는 경우 관례적으로 `Left`는 오류가 있는 경우, `Right`는 정상적인 경우를 표현한다.
`Option`이 아닌 `Either`를 사용하는 이유는 오류 정보등 복잡한 정보가 필요한 경우 이를 넘겨야 하기 때문이다.

`unbind`는 더 쉽다. `value`가 그냥 `String` 값이므로, 맵에 key->value 연관쌍을 넣어서 반환하면 된다.

이렇게 기본정의된 `stringFormatter`를 기반으로 만들어진 `FieldMapping[String]`이 바로 `tuple("userid" -> text,"password" -> text)`의 
두 인수이다. 이제 이렇게 두 인수를 받은 `tuple` 메소드는 다음과 같은 바인드, 언바인드 함수를 만들어낸다(이름은 그냥 내가 붙인것임)

```scala
val bindStringString = (a1: String, a2: String) => (a1, a2)
val unbindTuple = (t: (String, String)) => Some(t)
```

그리고 나서 이를 `mapping` 메소드에 넘긴다. 따라서 최종 호출되는 `mapping`은 다음과 같이 될 것이다. 
(혹시 몰라서 이야기하는데, 스칼라에서 보통 `->`는 `,`과 같은 역할을 한다. 따라서 `"password" -> text`는 
`("password",text)`와 같다.)

```scala
mapping(("userid", FieldMapping[String]), ("password", FieldMapping[String])) bindStringString unbindTuple
= ObjectMapping2(bindStringString, unbindTuple, ("userid", FieldMapping[String]), ("password", FieldMapping[String]))
```

### `ObjectMapping`에 대해

이제 개별 필드를 변환하는 것은 `FieldMapping[T]`에서 간단하게 살펴 보았다. 튜플을 만들려면 각 필드를 엮어서 
튜플로 만들어주는 작업을 해야 한다. 여러분이라면 `ObjectMapping2`라는 새로운 매핑을 어떻게 구현하겠는가?

아마도 다음과 같을 것이다.

1. `bind`는 두 필드 매핑의 `bind`를 사용해 최종 값을 만들고, 이를 전달받은 바인드 함수를 사용해 튜플로 엮는다.
2. `unbind`는 언바인드 함수를 사용해 먼저 객체를 필드들로 분리한 다음, 두 필드 매핑의 `unbind`를 사용해 각 값으로 변환한 맵을 합병하고, 오류가 발생하면 각 오류를 시퀀스로 엮어서 반환한다.

나머지 부분은 핵심적인 부분이 아니므로 넘어가자.

이제 실제 구현 코드는 어떻게 되어있나 보자. `ObjectMapping1`은 `Form.scala`에 정의되어 있으나 `ObjectMapping2`는 없다.
실제 이 구현은 `$PLAY_HOME/framework/src/play/src/main/scala/play/core/hidden/ObjectMappings.scala`에 있다. 
스칼라의 경우 패키지 구조와 디렉터리 경로가 꼭 일치할 필요는 없으므로, 이를 정보 은닉(?)에 활용할 수 있다.


```scala
package play.api.data

import format._
import validation._

case class ObjectMapping2[R, A1, A2](apply: Function2[A1, A2, R], unapply: Function1[R, Option[(A1, A2)]], f1: (String, Mapping[A1]), f2: (String, Mapping[A2]), val key: String = "", val constraints: Seq[Constraint[R]] = Nil) extends Mapping[R] with ObjectMapping {

  val field1 = f1._2.withPrefix(f1._1).withPrefix(key)

  val field2 = f2._2.withPrefix(f2._1).withPrefix(key)

  def bind(data: Map[String, String]): Either[Seq[FormError], R] = {
    merge(field1.bind(data), field2.bind(data)) match {
      case Left(errors) => Left(errors)
      case Right(values) => {
        applyConstraints(apply(

          values(0).asInstanceOf[A1],
          values(1).asInstanceOf[A2]))
      }
    }
  }

  def unbind(value: R): (Map[String, String], Seq[FormError]) = {
    unapply(value).map { fields =>
      val (v1, v2) = fields
      val a1 = field1.unbind(v1)
      val a2 = field2.unbind(v2)

      (a1._1 ++ a2._1) ->
        (a1._2 ++ a2._2)
    }.getOrElse(Map.empty -> Seq(FormError(key, "unbind.failed")))
  }

  def withPrefix(prefix: String): ObjectMapping2[R, A1, A2] = addPrefix(prefix).map(newKey => this.copy(key = newKey)).getOrElse(this)

  def verifying(addConstraints: Constraint[R]*): ObjectMapping2[R, A1, A2] = {
    this.copy(constraints = constraints ++ addConstraints.toSeq)
  }

  val mappings = Seq(this) ++ field1.mappings ++ field2.mappings

}
```

`bind`를 보면 내 예상과 별 다르지 않다. 다만 오류처리가 필요하기 때문에, 약간의 부가 작업이 들어간 것 뿐이다. 
각 필드에 대해 `bind`를 한 다음 이를 `merge`한다. 
`merge`가 반환한 값을 봐서 오류(`Left`)인 경우에는 이를 그대로 반환하고, 정상(`Right`)인 경우에는 
ObjectMapping2의 생성시 제공받은 객체 바인딩 메소드(`apply`)를 호출해서 새로 객체를 만든 다음,
applyConstraints()를 호출해 값을 검증한다.

`merge`는 `ObjectMapping` 객체에 정의되어 있다. 

```scala
/**
 * 모든 ObjectMapping을 위한 공통 도우미 메소드들 
 */
trait ObjectMapping {
{
  /**
   * 두 바인딩의 결과를 결합함
   *
   * @see bind()
   */
  def merge2(a: Either[Seq[FormError], Seq[Any]], b: Either[Seq[FormError], Seq[Any]]): Either[Seq[FormError], Seq[Any]] = (a, b) match {
    case (Left(errorsA), Left(errorsB)) => Left(errorsA ++ errorsB)
    case (Left(errorsA), Right(_)) => Left(errorsA)
    case (Right(_), Left(errorsB)) => Left(errorsB)
    case (Right(a), Right(b)) => Right(a ++ b)
  }

  /**
   * 여러 바인딩의 결과를 합함
   *
   * @see bind()
   */
  def merge(results: Either[Seq[FormError], Any]*): Either[Seq[FormError], Seq[Any]] = {
    val all: Seq[Either[Seq[FormError], Seq[Any]]] = results.map(_.right.map(Seq(_)))
    all.fold(Right(Nil)) { (s, i) => merge2(s, i) }
  }
}
```

`merge`가 하는 일은 다음과 같다.

1. `results.map(_.right.map(Seq(_)))` : `results` 리스트 내의 `Either` 중에서 `Right[Any]`인 것들만 다시 `Right[Seq[Any]]` 형태로 만들어준다. 
 이렇게 하는 이유는 `results` 리스트가 `Left`인 경우에는 `Seq`이지만, `Right`인 경우에는 `Seq`가 아니라서 타입이 어긋나기 때문에, 
 나중에 fold시 인자간의 타입을 서로 맞춰주기 위해서이다.
2. `all.fold(Right(Nil)) { (s, i) => merge2(s, i) }` : 1.에서 만들어진 `Either`의 리스트에 대해 실제 머지를 하는 부분이다.
 함수언어에서는 이렇게 `fold`와 `map`을 사용해 리스트로 부터 최종 값을 모으는 경우가 많다. 보통 `map`을 
 사용해 각 원소에 대해 변환(또는 계산)을 실시한 다음 `fold`를 사용해 전체 결과를 한 값으로 모은다.
 
`merge2`는 두 인자중 어느 한쪽에  오류가 발생한 경우에는 오류를 모아가고, 둘 다 오류가 아닌 경우에는 정상 값을 모아간다.
아까 `merge`에서 `Seq`로 양쪽의 타입을 맞춰놨기 때문에, 여기서는 시퀀스의 `++` 연산자(append)를 사용해 값을 합쳐나가면 된다.

결국, `merge`의 최종 결과는 중간에 오류가 있는 경우 `Left(모든 오류의 리스트)`, 그렇지 않은 경우 
`Right(모든 변환된 값의 리스트)`가 된다.


이제 `ObjectMapping2.unbind`를 보면, 먼저 `unapply(value)`를 한 다음 각 필드에 대해 필드의 `unbind`를 호출한다.
그리고 나서, 각각을 `unbind`한 값에서 오류 부분과 맵 부분을 각각 `++`로 묶어준다. 

이제 (String, String) 튜플을 처리하는 경우를 보면서, 플레이 소스코드를 함께 살펴 보았다. 
전체적인 값 처리 부분이 이해가 되었으리라 본다.



 
 
 
