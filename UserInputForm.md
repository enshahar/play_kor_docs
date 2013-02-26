이제 목록을 표시하는 작업은 되었지만, 해야할 일이 많다. 예제 데이터를 보여주는 것 말고, 사용자(관리자)가 데이터를 입력할 수 있어야 한다. 

# 데이터 입력 폼

사용자 입력은 아다시피 html 폼(form)을 사용한다. 폼을 다룰 경우 다음과 같은 단계를 거쳐야 한다.

* 사용자 입력을 위한 빈 폼을 표시한다.
* 사용자가 데이터를 입력하고 submit을 누르면 POST(또는 GET)된 데이터의 적합성을 검사하고, 오류가 있으면 다시 표시해야 한다. 
* 적합성 검사를 통과하면 데이터를 객체에 담아 데이터베이스(또는 ORM)를 사용해 저장해야 한다.

빈 폼은 직접(HTML로) 만들 수 있다. 하지만, submit시 적합성 검사가 실패한 경우 다시 입력화면으로 전환되어야 하는데, 이때 필드에는 사용자가 이미 입력한 값과 왜 재 입력이 필요한지 오류 메시지가 표시되는 편이 좋다. 그 후 다시 객체에 담아 DB/ORM으로 전달하면 된다.

이를 직접 손으로 구현한다고 보자. 다음과 같은 일을 해야 할 것이다.

* 폼 표시시: 입력 태그를 HTML로 표시하고, 거기 적절한 이름(name 애트리뷰트)과 값(value 애트리뷰트)를 지정해야 한다. 새로 입력하는 경우는 값이 비어있지만, 사용자 입력에 잘못된 부분이 있어서 오류를 표시하고 돌아온 경우이거나, 기존 데이터를 수정하는 경우라면 각각 기존 입력 값(HTTP POST 데이터)이나 데이터베이스에서 가져온 모델 객체의 필드 값을 표시해야 한다.

* 폼 제출시: 제출 버튼을 누르면 데이터 무결성을 체크하고, 폼 액션을 수행해야 한다. 이때 폼 액션을 받는 URL에 매핑된 컨트롤러 함수에서는 거꾸로 POST된 데이터에서 데이터베이스 모델 객체에 값을 저장해 데이터베이스에 넣거나 관련 비즈니스 로직을 수행해야 한다.

이런 맵핑을 하려면 결국 폼의 필드들과 그에 대응하는 모델 객체의 필드/애트리뷰트 간에 연관을 시켜줘야 한다. 일반적으로 어떤 객체가 있을 때 그로부터 이런 맵핑을 만들 수 있는 방법은 다음과 같은 것들이 있다.

1. 리플랙션 등 객체/클래스 정보를 얻어올 수 있는 기능을 사용해 필드 이름 등을 가져온다.
2. 사용자가 맵핑을 지정하고, 그를 활용한다.
3. 프로그래머가 직접 코드를 작성한다.

가장 간편한 것은 1의 방법을 사용하는 것이다. 기본적으로 리플렉션을 사용해 디폴트 연동이 가능하게 하고, 프로그래머가 약간의 변경을 가할 수 있게 해주면 좋을 것이다. 하지만, 아직 스칼라에 리플렉션이 도입된 지 얼마 안되서 이런 부분은 플레이에 반영되어 있지 않은 모양이다. 어쩔 수 없이 2번 방법을 사용해야 한다.

## 폼이 하는 일

매핑시 필요한 변환을 잘 생각해 보자.

1. 객체 -> 폼 변환시 : 객체의 각 필드로부터 폼의 입력 데이터를 채워넣을 수 있어야 한다. 예를 들어 `<input type="text" name="logingname" value=""/>`라는 필드에 대한 값을  `User`의 인스턴스 `u`로부터 가져오려면, `u.loginname`을 `loginname`이르는 `text` 타입의 입력필드의 `value` 애트리뷰트에 넣어야 한다. 
 
2. 폼 -> 객체 변환시 : 위 1의 역 과정이다.

플레이의 `play.api.data`에는 이런 과정을 처리하기 위한 폼 데이터 타입이 정의되어 있다.
(자세한 것은 [폼 튜플 처리에 대한 내 글](http://enshahar.com/2013/02/17/form-처리에-대해-폼-튜플-처리/)을 보라)

```scala
  // 일부러 폼관련 메소드/객체에는 FQN을 사용함
  val userForm = play.api.data.Form(
    play.api.data.Forms.mapping (
      "id" -> play.api.data.Forms.optional(play.api.data.Forms.longNumber),
      "loginname" -> play.api.data.Forms.nonEmptyText,
      "name" -> play.api.data.Forms.nonEmptyText,
      "mobile" -> play.api.data.Forms.text
    )(User.apply)(User.unapply)
  )
```

위와 같은 userForm 선언을 `app/models/User.scala`의 `Users` 객체에 넣자.

### 폼 정의시 사용할 수 있는 매핑들
폼 정의시 사용할 수 있는 각종 매핑에는 다음과 같은 것들이 있다(주로 play.api.data.Forms 객체에 정의되어 있다.)

* String 매핑들

    * `val text`
    * `val nonEmptyText`
    * `def text(minLength: Int = 0, maxLength: Int = Int.MaxValue)`
    * `def nonEmptyText(minLength: Int = 0, maxLength: Int = Int.MaxValue)`

* 수 매핑들 
   
    * `val number`
    * `val longNumber`
    * `def number(min: Int = Int.MinValue, max: Int = Int.MaxValue, strict: Boolean = false)`
    * `def longNumber(min: Long = Long.MinValue, max: Long = Long.MaxValue, strict: Boolean = false)`
    * `val bigDecimal`
    * `def bigDecimal( precision : Int, scale: Int )`

* 날짜 매핑들

    * `val date`
    * `def date(pattern: String, timeZone: java.util.TimeZone = java.util.TimeZone.getDefault)`
    * `val sqlDate`
    * `def sqlDate(pattern: String, timeZone: java.util.TimeZone = java.util.TimeZone.getDefault)`
    * `val jodaDate`
    * `def jodaDate(pattern: String, timeZone: org.joda.time.DateTimeZone = org.joda.time.dateTimeZone.getDefault)`
    * `val jodaLocalDate`
    * `def jodaLocalDate(pattern: String)`


* 다른 매핑을 감싸는 매핑
  
    * `def ignored[A](value: A)`
    * `def optional[A](mapping: Mapping[A])`
    * `def default[A](mapping: Mapping[A], value:A)`
    * `def list[A](mapping: Mapping[A])`
    * `def seq[A](mapping: Mapping[A])`

* 전자우편 주소

    * `val email`

* 불린 필드 매핑

    * `val boolean`
    * `def checked(msg: String)`
    
## 사용자 입력

이제 폼을 정의했으니, 사용자 입력 화면을 구성해보자.

우선 사용자 입력을 받을 컨트롤러 액션을 `conf/routes`에 추가해야한다.
하는김에 예전에 선언했던 `POST /users` 경로의 액션을 `saveUser`로 바꾸고,
새 사용자 입력화면은 `newUser`를 사용하도록 하자.

```scala
# Routes
# This file defines all application routes (Higher priority routes first)
# ~~~~

# Home page
GET     /                           controllers.Application.index

# Map static resources from the /public folder to the /assets URL path
GET     /assets/*file               controllers.Assets.at(path="/public", file)

# Map user related urls - 12/02/2013
GET /users			controllers.Application.users
GET /users/new 		controllers.Application.newUser
GET	/users/:id		controllers.Application.user(id: Long)
POST	/users			controllers.Application.saveUser	
POST	/users/:id/delete	controllers.Application.deleteUser(id: Long)
POST	/users/:id/modify	controllers.Application.modifyUser(id: Long)
```

이제 컨트롤러 함수를 추가하자. `app/controllers/Application.scala`에 
다음과 같은 코드를 넣는다.

```scala
  ... 생략 ...
  // 사용자 폼을 표시한다.
  def newUser = Action {
    Ok(views.html.user.createForm(Users.userForm))
  }

  def saveUser = TODO
  ... 생략 ...
```

이제 createForm이라는 html 템플릿을 넣어야 한다. `app/views/user/createForm.scala.html`을 
넣자.

```
@(userForm: Form[models.User] )

@import helper._

    <h1>사용자 추가</h1>
    
    @form(routes.Application.saveUser()) {
         
        <fieldset>
        
            @inputText(userForm("loginname"), '_label -> "로그인 이름")
            @inputText(userForm("name"), '_label -> "이름")
            @inputText(userForm("mobile"), '_label -> "휴대전화")

        </fieldset>
        
        <div class="actions">
            <input type="submit" value="사용자 추가" class="btn primary"> or 
            <a href="@routes.Application.users()" class="btn">취소</a> 
        </div>
        
    }
```

이 코드에 대해서는 약간 설명이 필요할 수도 있다.

* 첫 줄의 `@(userForm: Form[models.User] )`은 `userForm`이라는 인자를 받는 템플릿임을 선언하는 것이다.
* `@form(routes.Application.saveUser()) { ... } `는 폼 액션을 지정해 폼을 출력하게 한다. 
* `routes.Application.saveUser()`는 URL 매핑(`conf/routes`)에서 `Application.saveUser()`에 대한 URL을 가져온다.
* `@inputText(userForm("loginname"), '_label -> "로그인 이름")`는 텍스트 입력 필드를 만들어낸다. 화면에 `_label`로 지정한 값을 필드 앞에 표시해준다.

이제 이 코드를 다 번영하고 나서 리로드를 하면 다음과 같은 입력 화면을 볼 수 있다. 

여기서 '사용자 추가' 버튼을 눌러도 실제 데이터를 넣는 액션(`saveUser()`)을 구현하지 않았기 때문에 `TODO` 화면만 표시된다.

### 입력 필드 출력 처리 과정(소스분석임. 건너 띄어도 좋음)

입력 필드를 생성하는 코드들은 views.html.helper 패키지 안에 있다.
(소스코드는 `$PLAY_HOME/framework/src/play/src/main/scala/views/helper/`아래 있다.)

`inputText()`의 경우 html파일 관례에 따라 inputText.scala.html 안에 들어가 있다.
이해를 돕기 위해 잠시 들여다 보자. 

```scala
@**
 * Generate an HTML input text.
 *
 * Example:
 * {{{
 * @inputText(field = myForm("name"), args = 'size -> 10, 'placeholder -> "Your name")
 * }}}
 *
 * @param field The form field.
 * @param args Set of extra attributes.
 * @param handler The field constructor.
 *@
@(field: play.api.data.Field, args: (Symbol,Any)*)(implicit handler: FieldConstructor, lang: play.api.i18n.Lang)

@inputType = @{ args.toMap.get('type).map(_.toString).getOrElse("text") }

@input(field, args.filter(_._1 != 'type):_*) { (id, name, value, htmlArgs) =>
    <input type="@inputType" id="@id" name="@name" value="@value" @toHtmlArgs(htmlArgs)>
}
```

커맨트 다음의 `@(field: play.api.data.Field, args: (Symbol,Any)*)(implicit handler: FieldConstructor, lang: play.api.i18n.Lang)`를 보면 우리가 이를 호출할 때 사용한 `@inputText(userForm("loginname"), '_label -> "로그인 이름")`과 맞춰볼 수 있다. 뒤의 `args: (Symbol, Any)*`부분은 `'_label -> "로그인 이름"`과 타입이 일치한다. 
첫번째 인자 `field: play.api.data.Field`부분을 살펴보면, `userForm("loginname")`으로 호출했다. `Form` 클래스를 보면 `def apply(key: String): Field`라고 Field타입을 반환하게 되어 있다. 이때 폼이 가지고 있는 제약사항과 형식, 오류, 데이터 등의 정보를 Field에 담아 반환한다.

`@inputType = @{ args.toMap.get('type).map(_.toString).getOrElse("text") }`는 혹시 `args`에 `'type`이라는 심볼로 정의된 값이 있는지 봐서 그 값을 사용하거나, 디폴트로 `"text"`를 사용한다.

그리고 나서, `input()`이라는 메소드에 처리를 넘긴다. 먼저 `input.scala.html`이 있나 살펴보면, `inputText.scala.html`과 같은 디렉터리에 있다.

```scala
@**
 * Prepare a generic HTML input.
 *@
@(field: play.api.data.Field, args: (Symbol, Any)* )(inputDef: (String, String, Option[String], Map[Symbol,Any]) => Html)(implicit handler: FieldConstructor, lang: play.api.i18n.Lang)

@id = @{ args.toMap.get('id).map(_.toString).getOrElse(field.id) }

@handler(
    FieldElements(
        id,
        field,
        inputDef(id, field.name, field.value, args.filter(arg => !arg._1.name.startsWith("_") && arg._1 != 'id).toMap),
        args.toMap,
        lang
    )
)
```

`input()`의 인자 타입을 `inputText`내에서 `input`을 호출한 부분과 비교해 보자.

* 첫번째 인자로 `field`를 넘긴다.
* 두번째 인자로 `'type`을 제외한 나머지 `args`를 넘긴다.
* 마지막 인자로 `(id, name, value, htmlArgs)`를 받아 HTML 코드(`<input type="@inputType" id="@id" name="@name" value="@value" @toHtmlArgs(htmlArgs)>`)를 만드는 함수를 전달하고 있다.

`input()`에서는 인자들을 받아서, `'id`를 빼내고, `handler()`함수에 `FieldElements()`를 생성해 전달한다.

`FieldElements()`는 `input.scala.html`과 같은 디렉터리에 있는 `Helpers.scala`에 정의되어 있다.
`handler()`는 implicit인자이다([A Tour of Scala: Implicit Parameters](http://www.scala-lang.org/node/114)를 살펴보라.). 

잠깐 딴길로 새자....

---
implicit 인자를 직접 지정하지 않고 메소드를 호출하면 다음 순서대로 implicit 객체를 찾는다.

* 현재 위치에서 직접 보이는(lexical scope상으로 상위) 곳에 정의된 implicit 인자와 타입이 일치하는 객체
* implicit 인자의 타입(클래스)의 컴패니언 객체에 정의된 implicit 값 중 타입이 일치하는 객체

아래는 간단하게 위 규칙을 보여주는 예이다. 

```scala
// implicit 매개변수 객체 찾기 규칙을 보여주는 예
// 이를 Test.scala등으로 저장해 컴파일한 후, `scala Test`로 실행해 보시오.
case class User(val x:Int) 

object User {
  implicit val defaultUser:User = User(999)
}
 
object Test {
  implicit val user1 = User(10)
  val user2 = User(20)
   
  def main(args:Array[String])
  {
    def addUser(x:User)(implicit user1:User) = x.x + user1.x
    
    println(addUser(user1)(user2))
    println(addUser(user2))
  }
}

```

---

이제 다시 돌아와서 `handler`의 타입을 보면 `FieldConstructor`이다. `Helper.scala`를 보면 
`implicit val defaultField = FieldConstructor(views.html.helper.defaultFieldConstructor.f)`라는 
정의를 볼 수 있다.

결국, `inputText`는 `input`을 거쳐 `defaultFieldConstructor`라는 템플릿의 함수를 호출하게 된다.

이 템플릿을 보면 다음과 같다.

```scala
@(elements: FieldElements)
<dl class="@elements.args.get('_class) @if(elements.hasErrors) {error}" id="@elements.args.get('_id).getOrElse(elements.id + "_field")">
    <dt><label for="@elements.id">@elements.label(elements.lang)</label></dt>
    <dd>@elements.input</dd>
    @elements.errors(elements.lang).map { error =>
        <dd class="error">@error</dd>
    }
    @elements.infos(elements.lang).map { info =>
        <dd class="info">@info</dd>
    }
</dl>
```

살펴보면 폼에 오류가 있는 경우에는 오류 정보를 표시하고, 그렇지 않은 경우에는 정보를 표시하는 형태임을 알 수 있다.

## 사용자 입력 처리

이제 사용자 입력 폼의 작성이 끝났다. 폼에 사용자 정보를 입력하고 등록 버튼을 누르면, 데이터베이스에 사용자 정보를 업데이트해야 한다.
이 작업은 slick(예전에는 scalaQuery였음)을 사용할 것이다. 이에 대해서는 다음에 살펴보도록 하겠다.
