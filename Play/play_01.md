# Play 入门与学习（一）

## Controllers

```scala
@Singleton
class HomeController @Inject()(cc: ControllerComponents) extends AbstractController(cc) {

  def index() = Action { implicit request: Request[AnyContent] =>
    Ok(views.html.index())
  }

  def explore() = Action { implicit request: Request[AnyContent] =>
    Ok(views.html.explore())
  }

  def tutorial() = Action { implicit request: Request[AnyContent] =>
    Ok(views.html.tutorial())
  }
    
}
```



在 `conf` 目录下配置 `routes` 文件，对请求url 进行映射到 `controllers`，和 `springmvc` 的  `@RequestMapping` 很类似。



```properties
# Routes
# This file defines all application routes (Higher priority routes first)
# https://www.playframework.com/documentation/latest/ScalaRouting
# ~~~~

# An example controller showing a sample home page
GET         /                    controllers.HomeController.index
GET         /explore             controllers.HomeController.explore
GET         /tutorial            controllers.HomeController.tutorial
```



通过上述配置之后，我们就可以通过 url 访问到具体的 请求方法了。



### Action

每一个请求是被一个 Action 进行处理了，处理之后返回 Results



### Results



常见的 `results`

```scala
Ok("Got request [" + request + "]")
```

重定向到另一个 `url` 对应的 `Action`

```scala
Redirect("/echo")
```

`mark` 方法还没有完成

```scala
def todo() = TODO
```



自定义 `Result`

```scala
Result(
  header = ResponseHeader(200, Map.empty),
  body = HttpEntity.Strict(ByteString("Hello world!"), Some("text/plain"))
)
```



## Http Routing

> 所有路由信息将定义在 `conf/routes` 文件下,前文有提及到。`router` 是负责将每个传入 `HTTP` 请求转换为 `Action` 的组件。



每一个 `Http` 请求被 Play MVC Framework 认为是一个事件。每个请求包含两条主要信息：

- 请求路径，restful 风格
- http 请求方法，类似 Get、Post、Delete、Put 等



### 语法

`conf/routes` 是 `router` 使用的配置文件。该文件列出了应用程序所需的所有 `routes`。每个路由由一个 `HTTP` 方法和 `URI` 模式组成，它们都与 `Action` 的调用相关联。

让我们看看路由定义是什么样的: 



```properties
GET   /clients/:id          controllers.Clients.show(id: Long)
GET   /clients/:id          controllers.Clients.show(id: Long)
```



通过 `->` 来使用不同的路由规则

```properties
->      /api                        api.MyRouter
```

当与字符串插值路由DSL(也称为SIRD路由)结合使用时，或者在处理使用多个路由文件路由的子项目时，这一点尤其有用。



通过 `nocsrf` 来禁用 `CSRF filter`

```properties
+ nocsrf
POST  /api/new              controllers.Api.newThing
```

URL规则:

```properties
#静态 path
GET   /clients/all          controllers.Clients.list()


# 动态 path
GET   /clients/:id          controllers.Clients.show(id: Long)

# 正则匹配模式

GET   /files/*name          controllers.Application.download(name)
```



逆向、反转 routing

```properties
 Redirect(routes.HelloController.echo())
```



## 操作处理结果返回

结果内容类型自动从您指定作为响应体的Scala值推断出来。

通过 `play.api.http.ContentTypeOf` 来实现

```scala
//Will automatically set the Content-Type header to text/plain, while:
val textResult = Ok("Hello World!")

//will set the Content-Type header to application/xml.
val xmlResult = Ok(<message>Hello World!</message>)

//自己定义返回类型
val htmlResult = Ok(<h1>Hello World!</h1>).as("text/html")
val htmlResult2 = Ok(<h1>Hello World!</h1>).as(HTML)

```



### Manipulating Http headers (操纵 Http 头)

```scala
//添加返回头信息
val result = Ok("Hello World!").withHeaders(CACHE_CONTROL -> "max-age=3600",ETAG -> "xx")
```



## Session and Flash

> 存储在 session 中的数据在整个会话期间都是可用的，存储在 flash 作用域的数据只对下一个请求可用。
>
> 需要注意，session 和 flash 的数据不是由服务器存储的，而是使用 cookie 机制添加到每个后续 http 请求中的。这意味着数据大小将非常有限(up to 4kb),并且只能存储字符串值。 cookie 的默认名称是 `PLAY_SESSION`。这可以在 `application.conf` 通过配置 key `play.http.session` 来更改。



### Session 存储

```scala
//这会将 session 完全替换掉
Ok("Welcome!").withSession("connected" -> "user@gmail.com")
//在现有session 基础上添加 element 内容即可
Ok("Hello World!").withSession(request.session + ("saidHello" -> "yes"))
//通过key remove 部分内容
Ok("Theme reset!").withSession(request.session - "theme")
```

读取 `Session` 内容

```scala
def index = Action { request =>
  request.session.get("connected").map { user =>
    Ok("Hello " + user)
  }.getOrElse {
    Unauthorized("Oops, you are not connected")
  }
}
```

丢弃整个 `Session` 内容

```scala
Ok("Bye").withNewSession
```



### Flash Scope (`Flash`作用域)

> flash scope 和 session 作用很像，但是有两个区别 

- data are kept for only one request
- the Flash cookie is not signed, making it possible for the user to modify it.

session 的 cookie 会进行加密，而 flash 的 cookie 不会进行加密。

`Flash作用域` 应该只用于在简单的 `非ajax` 应用程序上传输 成功/错误消息。由于数据只是为下一个请求保存的，而且在复杂的 `Web` 应用程序中不能保证请求顺序，所以 `Flash作用域` 受竞态条件的限制。(the Flash scope is subject to race conditions.)



### code using flash scope 

```scala
def index = Action { implicit request =>
  Ok {
    request.flash.get("success").getOrElse("Welcome!")
  }
}

def save = Action {
  Redirect("/home").flashing("success" -> "The item has been created")
}
```

To retrieve the Flash scope value in your view, add an implicit Flash parameter

要在视图中检索Flash作用域值，请添加一个隐式Flash参数:

```scala
@()(implicit flash: Flash)
...
@flash.get("success").getOrElse("Welcome!")
...
```



## 请求体解析 Body Parsers 

> 什么是 body parsers

`HTTP请求` 是header 后面跟着 body。header 通常很小——它可以安全地缓冲在内存中，因此在 `Play` 中它是使用`RequestHeader` 类建模的。

然而，`body` 可能非常长，因此不在内存中缓冲，而是建模为流。然而，许多请求体有效负载 (`payloads`) 都很小，并且可以在内存中建模，因此为了将 `body流` 映射到内存中的对象，`Play` 提供了一个`BodyParser` 抽象。

由于 `Play` 是一个异步框架，传统的 `InputStream` 不能用于读取请求体——输入流阻塞了，当您调用 `read` 时，调用它的线程必须等待数据可用。

相反，`Play` 使用一个名为 `Akka Streams` 的异步流库。**Akka Streams** 是 **Reactive Streams** 的一个实现。 

允许许多异步流api 无缝地协同工作,所以尽管传统 `InputStream` 的基础技术不适合使用, 但是`Akka Streams` 和 `Reactive Streams` 的整个生态系统的异步库将为你提供你需要的一切。



### 使用 Body Parsers

> 如果没有显式选择 `body parser`，`Play` 将使用的缺省的 `body parser` 将查看传入的 `Content-Type`，并相应地解析body。
>
> 例如，类型 `application/json` 的内容类型将被解析为 `JsValue`，而类型 `application/x-www-form- urlencoding` 的内容类型将被解析为 `Map[String, Seq[String]]`

默认的 `Body Parser` 生成 `AnyContent` 类型的 `Body`。`AnyContent` 支持的各种类型, 可以通过 `as方法` 访问，例如 `asJson`，它返回一个 Option[body类型]

```scala
def save = Action { request: Request[AnyContent] =>
  val body: AnyContent = request.body
  val jsonBody: Option[JsValue] = body.asJson

  // Expecting json body
  jsonBody.map { json =>
    Ok("Got: " + (json \ "name").as[String])
  }.getOrElse {
    BadRequest("Expecting application/json request body")
  }
}
```



下面是默认 `body parser` 支持的类型映射 (`The following is a mapping of types supported by the default body parser`)

- **text/plain**: `String`, accessible via `asText`.
- **application/json**: [`JsValue`](https://static.javadoc.io/com.typesafe.play/play-json_2.12/2.6.9/play/api/libs/json/JsValue.html), accessible via `asJson`.
- **application/xml**, **text/xml** or **application/XXX+xml**: `scala.xml.NodeSeq`, accessible via `asXml`.
- **application/x-www-form-urlencoded**: `Map[String, Seq[String]]`, accessible via `asFormUrlEncoded`.
- **multipart/form-data**: [`MultipartFormData`](https://www.playframework.com/documentation/2.6.x/api/scala/play/api/mvc/MultipartFormData.html), accessible via `asMultipartFormData`.
- Any other content type: [`RawBuffer`](https://www.playframework.com/documentation/2.6.x/api/scala/play/api/mvc/RawBuffer.html), accessible via `asRaw`.



默认的 body parser 在解析之前会 `try to determine` 请求是否具有 `body`。

根据 `HTTP` 规范(spec)，内容长度(Content-Length) 或 传输编码标头(Transfer-Encoding) 的出现都表示主体的存在，因此解析器只在出现其中一个标头时进行解析，或者在显式设置非空主体时在 FakeRequest 上进行解析。


如果希望在所有情况下解析主体，可以使用下面描述的anyContent主体解析器。



### 显示的选择一个 Body Parser

如果希望显式地选择主体解析器，可以将 body parser 传递给 Action 的 apply或 async 方法。

Play 提供了许多开箱即用的 body parser (`Play provides a number of body parsers out of the box`)，

这是通过 `PlayBodyParsers` trait 提供的，它可以注入到您的控制器中。



例子，如果要定义一个 request body 期望是 Json Body 的 Action

```scala
//如果不是 Json类型会返回 415 Unsupported Media Type , 类似 Springmvc 在参数前面加 @RequestBody
def save = Action(parse.json) { request: Request[JsValue]  =>
  Ok("Got: " + (request.body \ "name").as[String])
}
```

注意，上述 body 的类型为 `JsValue`, 它不是 `Option` 类型的了。原因是 json body parser 会验证请求是否具有 `application/json` 的 `Content-Type`，如果不是，则会返回 415 的错误，即 `415 Unsupported Media Type`,这样我们就不用再次检查了。

这样依赖，这个方法将要对请求的type 有严格的限制了，客户端要清楚这一点。如果希望有更宽松的做法，即不是 ``application/json` 类型的 `Content-Type` 也能够进行解析，可以使用如下方法：

```scala
//不会返回 415，会尝试进行解析
def save = Action(parse.tolerantJson) { request: Request[JsValue]  =>
  Ok("Got: " + (request.body \ "name").as[String])
}
```

另一个例子，保存文件

```scala
def save = Action(parse.file(to = new File("/tmp/upload"))) { request: Request[File]  =>
  Ok("Saved the request content to " + request.body)
}
```



### 组合 Body Parsers (Combining body parsers)

在上一个保存文件的示例中，所有请求 bodies 都存储在同一个文件中 (`/tmp/upload`)，这是存在问题的。

我们可以编写另一个自定义主体解析器，它从请求会话中提取用户名，为每个用户提供一个惟一的文件

```scala
val storeInUserFile: BodyParser[File] = parse.using { request =>
  request.session.get("username").map { user =>
    parse.file(to = new File("/tmp/" + user + ".upload"))
  }.getOrElse {
    sys.error("You don't have the right to upload here")
  }
}
//自定义 body parser
def save: Action[File] = Action(storeInUserFile) { request =>
  Ok("Saved the request content to " + request.body)
}
```

上面我们并没有真正编写自己的 `BodyParser`，而只是组合现有的 `BodyParser`。这通常就足够了，应该涵盖大多数用例。高级主题部分将介绍从头编写 `BodyParser`



### Max content length

基于文本的 body parsers (例如 text、json、xml 或 formurlencoding) 需要有一个最大内容长度。因为它们必须将所有内容加载到内存中。默认情况下，它们将解析的最大内容长度是 `100KB`。可以通过指定 `play.http.parser` 来覆盖它。可以通过在 `application.conf` 指定 `maxMemoryBuffer`属性来改变这个大小。

```properties
# application.conf
play.http.parser.maxMemoryBuffer=128K
```



对于将内容缓冲 (`buffer`) 到磁盘上的 body parser，例如 `raw parser` 或 `multipart/form-data` 等，它们的最大内容长度使用 `play.http.parser` 的 `maxDiskBuffer` 来进行指定。

`maxDiskBuffer`属性，默认值为`10MB`。`multipart/form-data` 解析器还强制为数据字段的聚合设置文本最大长度属性。

您还可以通过在 `Action` 中显示指定来覆盖默认的最大长度。

```scala
// Accept only 10KB of data.
def save = Action(parse.text(maxLength = 1024 * 10)) { request: Request[String]  =>
  Ok("Got: " + text)
}
```

You can also wrap any body parser with `maxLength`

```scala
// Accept only 10KB of data.
def save = Action(parse.maxLength(1024 * 10, storeInUserFile)) { request =>
  Ok("Saved the request content to " + request.body)
}
```



### Writing a custom body parser

> 我们可以实现  `BodyParser` 特质来自定义 body parser，这个 trait 是一个很简单的函数



```scala
trait BodyParser[+A] extends (RequestHeader => Accumulator[ByteString, Either[Result, A]])
```

- 接收一个 `RequestHeader`，这样我们可以检查关于请求的信息，它通常用于获取 Content-Type,这样 body 就会被正确的解析。

- 函数的返回类型是 `Accumulator`，它是一个累加器。`Accumulator` 是 围绕 `Akka Streams Sink` 的一层薄层。`Accumulator` 异步地将 `streams of elements` 加到结果中，它可以通过传入 `Akka Streams Source` 来运行。这将返回一个 `Future`，并且在累加器完成时，`future `得到完成。
- `Accumulator` 本质上和 Sink[E,Future[A]] 一样，事实上，Accumulator 是 Sink 的一个包装器。但是最大的区别是 Accumulator 提供了方便的方法，比如 map、mapFuture、recover 等等。
- `Accumulator` 用于将结果作为一个 `promise` 来处理，其中 Sink 要求将所有的操作包装在 mapMaterializedValue 的调用中。
- Accumulator 的 apply 返回 ByteString 类型的元素，这些元素本质上是字节数组，但与 byte[] 不同的是，`ByteString` 是不可变的，它的许多操作(如 slicing 或者 appending) 都是在固定的时间内完成的。



`Accumulator` 的返回类型是 `Either[Result, A]`，它要么返回一个 `Result`，要么返回一个类型 `A`。通常在发生错误的情况下返回 `Result`。例如，如果 body 解析失败，比如 Content-Type 与 Body Parser 接受的类型不匹配，或者内存缓冲区溢出了。当 `body parser` 返回一个 `result` 时，这将缩短 `action` 的处理时间，`action` 的结果将里脊返回，并且永远不会调用该操作。



#### 将 Body 引向别处 Directing the body elsewhere 

> 编写 body parser 的一个常见用例是，当您实际上不想解析主体时，而是想将它以流的形式引入到其他地方(`stream it elsewhere`)。为此，您可以定义一个自定义 body parser



```scala
import javax.inject._
import play.api.mvc._
import play.api.libs.streams._
import play.api.libs.ws._
import scala.concurrent.ExecutionContext
import akka.util.ByteString

class MyController @Inject() (ws: WSClient, val controllerComponents: ControllerComponents)(implicit ec: ExecutionContext) extends BaseController {

  def forward(request: WSRequest): BodyParser[WSResponse] = BodyParser { req =>
    Accumulator.source[ByteString].mapFuture { source =>
      request
    	.withBody(source)
        .execute()
        .map(Right.apply)
    }
  }

  def myAction = Action(forward(ws.url("https://example.com"))) { req =>
    Ok("Uploaded")
  }
}
```



#### 使用 Akka Streams 进行自定义的解析 

> Custom parsing using Akka Streams

在很少的情况下，可能需要使用 `Akka Streams ` 编写自定义解析器。在大多数情况下，先用 `ByteString` 缓冲 body 就足够了，这通常提供一种简单得多的解析方法，因为您可以对主体使用强制方法和随机访问。


但是，当这不可行时，例如需要解析的主体太长而无法装入内存时，则可能需要编写自定义主体解析器。

关于如何使用 `Akka Streams` 的完整描述超出了本文档的范围——最好从阅读 `Akka Streams` 文档开始。但是，下面显示了一个 `CSV` 解析器，它基于`Akka Streams` 烹饪书中 `ByteStrings` 文档流的解析行。



```scala
import play.api.mvc.BodyParser
import play.api.libs.streams._
import akka.util.ByteString
import akka.stream.scaladsl._

val csv: BodyParser[Seq[Seq[String]]] = BodyParser { req =>

  // A flow that splits the stream into CSV lines
  val sink: Sink[ByteString, Future[Seq[Seq[String]]]] = Flow[ByteString]
    //我们按照 new line character 进行分割，每行最多允许 1000个字符。
    .via(Framing.delimiter(ByteString("\n"), 1000, allowTruncation = true))
    //把每一行变成一个String，用逗号分隔
    .map(_.utf8String.trim.split(",").toSeq)
    // 现在我们使用 fold 将其变为一个列表 List
    .toMat(Sink.fold(Seq.empty[Seq[String]])(_ :+ _))(Keep.right)

  // 将 body 转为为 Either right 返回
  Accumulator(sink).map(Right.apply)
}
```































































 











































