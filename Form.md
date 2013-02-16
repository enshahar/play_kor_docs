플레이 폼 처리시 정의하는 객체로 `play.api.data.Form`이 있다. 이에 대해 더 자세히 살펴보자.

먼저 사용방법을 살펴보고 그 후 내부 구현을 간략하게 들여다 볼 것이다.
이 부분을 정리하고 나면 플레이 프레임워크에서 좀 더 폼을 잘 활용하게 될 것이라 기대한다.

# 사용방법

## 모델 객체로 변환이 필요하지 않을 떄

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

### `Form` 생성 자세히 들여다 보기

폼의 생성자를 직접 사용하는 일은 별로 없다. 폼 이란 것이 데이터를 바인딩하기 위한 구조이기 때문에, 
보통은 생성자의 정보 중 `mapping` 정보만을 채워 넣으면 폼을 처리할 준비가 된 것이다. 
그리고 나서 필요에 따라 `data`를 바인딩하거나, 거꾸로 `value`를 채워 넣는 방식으로 처리가 진행될 것이다.

따라서 `Form` 객체를 생성하는 것은 우선은 `mapping`을 생성하는 것이 주가 될 수 밖에 없다. `mapping`이야 말로 
HTML 폼과 모델 객체 사이의 변환을 위한 스펙이라 할 수 있다.

스칼라에서 어떤 클래스 정의를 보던지, 가장 먼저 봐야 할 것 중 하나는 그 클래스의 컴패니언 객체일 것이다. 
`Form[T]` 클래스에도 역시 컴패니언 객체가 있다. 소스코드는 
`$PLAY_HOME\framework\src\play\src\main\scala\play\api\data\Form.scala`에 있다. 

```scala
/**
 * Provides a set of operations for creating `Form` values.
 */
object Form {

  /**
   * Creates a new form from a mapping.
   *
   * For example:
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
   * @param mapping the form mapping
   * @return a form definition
   */
  def apply[T](mapping: Mapping[T]): Form[T] = Form(mapping, Map.empty, Nil, None)

  /**
   * Creates a new form from a mapping, with a root key.
   *
   * For example:
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
   * @param mapping the root key, form mapping association
   * @return a form definition
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

이제 `Form.apply()`가 어떻게 정의되어 있는지를 알고 있으므로, 쉽게 tuple이라는 것이 mapping을 생성해주는
메소드 또는 함수라는 사실을 유추할 수 있다.

이제 `Form[T]`의 생성자에 정의된 `mapping`의 타입 `Mapping[T]` 타입이름을 보면 
딱 "T 타입으로의 맵핑이구나" 하는 감이 온다. 따라서, 아마도 Mapping[T]는 필드 이름과 필드 값을 받아서 
T 타입의 값을 만들어내주는 객체가 되리라 상상할 수 있다. 그것으로 충분할까? 아니다. 조금 더 생각해보자.

`Form[T]`에 있는 `bind`류의 함수들은 필드이름,필드값을 받아서 T 값을 만들어낼 수 있다면 충분히 
구현 가능할 것이다. 그렇지만, `fill`류의 함수들은 거꾸로 T에서 필드 값을 만들어 줘야 한다. 따라서 
`Mapping[T]`는 단순히 한방향(필드이름, 값의 맵 => T 타입의 값)만 지원해서는 안되고, 그 역방향(T 타입의 값 => 필드이름, 값의 맵)도 
지원해 주어야 함을 알 수 있다.

여기서 조금 더 나아가 생각해보면, Form[T]에는 입력값의 적합성 검사와 관련한 어떤 정보도 들어가 있지 않으므로, 
Mapping[T]에 이런 적합성 검사가 들어가야 함을 예상할 수 있다.

자, 그럼 한번 Mapping[T] 코드를 살펴보자. 역시 Form[T]와 같은 파일(Form.scala)에 정의되어 있다.

```scala

/**
 * A mapping is a two-way binder to handle a form field.
 */
trait Mapping[T] {
  self =>

  /**
   * The field key.
   */
  val key: String

  /**
   * Sub-mappings (these can be seen as sub-keys).
   */
  val mappings: Seq[Mapping[_]]

  /**
   * The Format expected for this field, if it exists.
   */
  val format: Option[(String, Seq[Any])] = None

  /**
   * The constraints associated with this field.
   */
  val constraints: Seq[Constraint[T]]

  /**
   * Binds this field, i.e. construct a concrete value from submitted data.
   *
   * @param data the submitted data
   * @return either a concrete value of type `T` or a set of errors, if the binding failed
   */
  def bind(data: Map[String, String]): Either[Seq[FormError], T]

  /**
   * Unbinds this field, i.e. transforms a concrete value to plain data.
   *
   * @param value the value to unbind
   * @return either the plain data or a set of errors, if the unbinding failed
   */
  def unbind(value: T): (Map[String, String], Seq[FormError])

  /**
   * Constructs a new Mapping based on this one, adding a prefix to the key.
   *
   * @param prefix the prefix to add to the key
   * @return the same mapping, with only the key changed
   */
  def withPrefix(prefix: String): Mapping[T]

  /**
   * Constructs a new Mapping based on this one, by adding new constraints.
   *
   * For example:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying(required) )
   * }}}
   *
   * @param constraints the constraints to add
   * @return the new mapping
   */
  def verifying(constraints: Constraint[T]*): Mapping[T]

  /**
   * Constructs a new Mapping based on this one, by adding a new ad-hoc constraint.
   *
   * For example:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying {_.grouped(2).size == 5})
   * }}}
   *
   * @param constraint a function describing the constraint that returns `false` on failure
   * @return the new mapping
   */
  def verifying(constraint: (T => Boolean)): Mapping[T] = verifying("error.unknown", constraint)

  /**
   * Constructs a new Mapping based on this one, by adding a new ad-hoc constraint.
   *
   * For example:
   * {{{
   *   import play.api.data._
   *   import validation.Constraints._
   *
   *   Form("phonenumber" -> text.verifying("Bad phone number", {_.grouped(2).size == 5}))
   * }}}
   *
   * @param error The error message used if the constraint fails
   * @param constraint a function describing the constraint that returns `false` on failure
   * @return the new mapping
   */
  def verifying(error: => String, constraint: (T => Boolean)): Mapping[T] = {
    verifying(Constraint { t: T =>
      if (constraint(t)) Valid else Invalid(Seq(ValidationError(error)))
    })
  }

  /**
   * Transform this Mapping[T] to a Mapping[B].
   *
   * @tparam B The type of the new mapping.
   * @param f1 Transform value of T to a value of B
   * @param f2 Transform value of B to a value of T
   */
  def transform[B](f1: T => B, f2: B => T): Mapping[B] = WrappedMapping(this, f1, f2)

  // Internal utilities

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

