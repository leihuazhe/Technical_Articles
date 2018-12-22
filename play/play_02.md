# 🎮 Play 入门与学习（二）
> 本章将讲解 Action 的组成和原理,并且介绍了几种定义通用 Action 的方法。

## Action composition
> Action 组成结构

### Custom action builders
>  自定义 action builders
> 我们有多种方法来声明一个 `Action`，使用 `request` 参数,不使用 `request` 参数, 使用 `body parser` 等等。实际上，并不止这些，我们将在异步编程一节进行讲述。


这些用于构建 `actions` 的方法实际上都是由一个名为 `ActionBuilder` 的 `trait` 定义的。而我们用来声明 `Action` 的 `Action Object` 只是这个 `trait` 的一个实例。通过实现自己的 `ActionBuilder`，您可以声明可重用的 `action stack`，然后可以使用它们来构建 `actions`。

让我们从日志修饰符(`logging decorator`) 的简单示例开始，我们希望记录对 `action` 的每次调用日志。

第一种方法是在 `invokeBlock` 方法中实现这个功能，`ActionBuilder` 构建的每一个 `action` 都会调用这个方法。

```scala
import play.api.mvc._

class LoggingAction @Inject() (parser: BodyParsers.Default)(implicit ec: ExecutionContext) extends ActionBuilderImpl(parser) {
    
 override def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result])={
    Logger.info("Calling action")
    block(request)
 }
}
```



现在，我们可以在 `controllers` 中使用依赖注入来获取 `LoggingAction` 的一个实例，并且以使用普通 `Action` 的方式来使用它。

```scala
class MyController @Inject()(loggingAction: LoggingAction,
                             cc:ControllerComponents)
  extends AbstractController(cc) {
  def index = loggingAction {
    Ok("Hello World")
  }
}
```

Since `ActionBuilder` provides all the different methods of building actions, this also works with, for example, declaring a custom body parser:



由于 `ActionBuilder` 提供了创建 `actions` 所有不同的方法，所以我们也可以在自定义的 `Action` 中使用 `body parser` 等普通 `Action` 的功能。

```scala
def submit = loggingAction(parse.text) { request =>
  Ok("Got a body " + request.body.length + " bytes long")
}
```



### 组合 actions 

>  Composing actions

在大多数应用程序中,我们希望有多个 `action builders`, 比如一些做不同类型的 `authentication`, 一些提供不同类型的通用功能组件,等等。

在这种情况下,我们不想为每一个 `action builder` 重写 `loggingAction`，我们需要定义一个可重用的方式来简化代码。可重用的操作代码可以通过包装操作（`wrapping actions`）来实现

```scala
import play.api.mvc._

case class Logging[A](action: Action[A]) extends Action[A] {

  def apply(request: Request[A]): Future[Result] = {
    Logger.info("Calling action")
    action(request)
  }

  override def parser = action.parser
  override def executionContext = action.executionContext
}
```

We can also use the `Action` action builder to build actions without defining our own action class:

我们也可以使用 `Action` action builder 来 创建 `actions` 而不需要定义我们自己的 `action class`

```scala
import play.api.mvc._

def logging[A](action: Action[A])= Action.async(action.parser) { request =>
  Logger.info("Calling action")
  action(request)
}
```

可以使用 `composeAction` 方法将 `Action` 混合到 `action builders` 中

```scala
class LoggingAction @Inject() (parser: BodyParsers.Default)(implicit ec: ExecutionContext) extends ActionBuilderImpl(parser) {
  override def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    block(request)
  }
  override def composeAction[A](action: Action[A]) = new Logging(action)
}
```

通过这样 `code` 后使用，效果和之前的例子一样

```scala
def index = loggingAction {
  Ok("Hello World")
}
```

我们也可以在不使用 `action builder` 的情况下将 `action` 混合到 `wrapping actions` 中

```scala
def index = Logging {
  Action {
    Ok("Hello World")
  }
}
```



### More complicated actions

> 更复杂的 Action

到目前为止，我们只展示了完全不会影响请求的 actions。当然，我们也可以对传入的请求对象进行读取和修改。

```scala
import play.api.mvc._
import play.api.mvc.request.RemoteConnection

def xForwardedFor[A](action: Action[A]) = Action.async(action.parser) { request =>
  val newRequest = request.headers.get("X-Forwarded-For") match {
    case None => request
    case Some(xff) =>
      val xffConnection = RemoteConnection(xff, request.connection.secure, None)
      request.withConnection(xffConnection)
  }
  action(newRequest)
}
```

**Note:** Play already has built in support for `X-Forwarded-For` headers.



我们可以 `block` 请求

```scala
import play.api.mvc._
import play.api.mvc.Results._

def onlyHttps[A](action: Action[A]) = Action.async(action.parser) { request =>
  request.headers.get("X-Forwarded-Proto").collect {
    case "https" => action(request)
  } getOrElse {
    Future.successful(Forbidden("Only HTTPS requests allowed"))
  }
}
```

最后我们还能修改返回的结果

```scala
import play.api.mvc._

def addUaHeader[A](action: Action[A]) = Action.async(action.parser) { request =>
  action(request).map(_.withHeaders("X-UA-Compatible" -> "Chrome=1"))
}
```



### Different request types

> 不同的请求类型

虽然 `action composition` 允许您在 `HTTP` request 和 response 级别执行额外的处理，但是您通常希望构建数据转换管道( `pipelines`)，以便向请求本身添加上下文(`context`) 或 执行验证(`perfom validation`)。

`ActionFunction` 可以看作是作用在 request 上的一个函数，在输入请求类型和传递到下一层的输出类型上都**参数化**。

每个操作函数都可以表示模块处理，例如身份验证、对象的数据库查找、权限检查或希望跨操作组合和重用的其他操作。



一些实现了实现 `ActionFunction` 的预定义的 `trait` 对于不同类型的处理非常有用。

- `ActionTransformer`
  -  can change the request, for example by adding additional information.
  - 可以改变一个请求，例如为此请求添加额外信息等。
- `ActionFilter`
  - can selectively intercept requests, for example to produce errors, without changing the request value.
  - 可以选择性的拦截请求，例如在不改变请求的前提下产生一个错误。
- `ActionRefiner`
  - is the general case of both of the above.
  - 上面两个 trait 的通用父类(trait)。
- `ActionBuilder`
  - is the special case of functions that take `Request` as input, and thus can build actions.
  - 以 "`Request`" 作为输入的函数的一种特殊情况，并且是否可以构建 actions。

```scala
trait ActionRefiner[-R[_], +P[_]] extends ActionFunction[R, P] {
  /**
   * 确定怎么处理一个请求，这是继承 ActionRefiner 后需要实现的主方法。
   *	 
   * 它可以决定立即拦截请求并返回结果(Left)，或者继续处理类型为P的新参数(Right)。
   *
   * @return Either a result or a new parameter to pass to the Action block
   */
  protected def refine[A](request: R[A]): Future[Either[Result, P[A]]]

  final def invokeBlock[A](request: R[A], block: P[A] => Future[Result]) =
    refine(request).flatMap(_.fold(Future.successful, block))(executionContext)
}


trait ActionFilter[R[_]] extends ActionRefiner[R, R] {
  /**
   * 确定是否处理请求。这是 ActionFilter 必须实现的主要方法
   *
   * 它可以决定立即拦截请求并返回结果(Some)，或者继续处理(None)。
   *
   * @return An optional Result with which to abort the request
   */
  protected def filter[A](request: R[A]): Future[Option[Result]]

  final protected def refine[A](request: R[A]) =
    filter(request).map(_.toLeft(request))(executionContext)
}

trait ActionTransformer[-R[_], +P[_]] extends ActionRefiner[R, P] {
  /**
   *扩展或转换现有请求。这是ActionTransformer必须实现的主要方法
   * @return The new parameter to pass to the Action block
   */
  protected def transform[A](request: R[A]): Future[P[A]]

  final def refine[A](request: R[A]) =
    transform(request).map(Right(_))(executionContext)
}
```

我们还可以通过实现 `invokeBlock` 方法定义自己的 `ActionFunction`。这样通常可以方便地创建请求的输入和输出类型实例(使用 `WrappedRequest`)，但这并不是严格必需的。



#### Authentication

> `action functions` 最常见的用例之一是身份验证。我们可以很容易地实现我们自己的身份验证操作转换器，它从原始请求确定用户并将其添加到新的 `UserRequest`。注意，这也是一个 `ActionBuilder`，因为它接受一个简单的请求作为输入

```scala
import play.api.mvc._

class UserRequest[A](val username: Option[String], request: Request[A]) extends WrappedRequest[A](request)

class UserAction @Inject()(val parser: BodyParsers.Default)(implicit val executionContext: ExecutionContext)
  extends ActionBuilder[UserRequest, AnyContent] with ActionTransformer[Request, UserRequest] {
  def transform[A](request: Request[A]) = Future.successful {
    new UserRequest(request.session.get("username"), request)
  }
}
```

内置的 `authentication action builder` 只是一个方便的帮助程序，它可以最小化实现简单情况下身份验证所需的代码，其实现与上面的示例非常相似。

由于编写自己的身份验证帮助程序很简单，所以如果内置的帮助程序不适合您的需要，我们建议这样做。






































