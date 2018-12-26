# 🎮 Play 入门与学习（四）
> 


## Handling asynchronous results

> 从现在开始我们将进入异步 `http` 编程模块，本章将会介绍如何处理异步返回结果。



### Make controllers asynchronous

> 使 controllers 变为异步的

在 `Play Framework` 内部，其机制是自底向上全异步编程模式的，Play 会以异步、非阻塞的方式处理每个请求。

默认配置(`configuration`) 针对异步控制器(`asynchronous controllers`) 进行了优化。换句话说，应用程序代码应该尽量避免在控制器中进行阻塞,这样会导致控制器一直等待那个阻塞的操作。此类阻塞操作的常见示例有JDBC调用、流API、HTTP请求和长计算等。

虽然可以增加默认 `executionContext` 中线程的数量，以允许阻塞控制器处理更多的并发请求，但是遵循建议的保持控制器异步的方法可以更容易地进行扩展，并在负载下保持系统响应。



### Creating non-blocking actions

> 创建非阻塞的 `actions`

由于 `Play` 的工作方式，`action` 代码必须尽可能快，即非阻塞。那么，如果我们在还没有生成结果情形下，如何返回结果呢?答案是使用 `Future`。

A `Future[Result]` will eventually be redeemed with a value of type `Result`. 

By giving a `Future[Result]` instead of a normal `Result`, we are able to quickly generate the result without blocking. 

Play will then serve the result as soon as the promise is redeemed.

The web client will be blocked while waiting for the response, but nothing will be blocked on the server, and server resources can be used to serve other clients.

Using a `Future` is only half of the picture though! 

> 然而，使用`Future`只是这幅画的一半. (使用 Future 只是 Play 异步编程的一半)

If you are calling out to a blocking API such as JDBC,then you still will need to have your ExecutionStage(`执行阶段`) run with a different executor, to move it off Play’s rendering thread pool. 

You can do this by creating a subclass of `play.api.libs.concurrent.CustomExecutionContext`with a reference to the [custom dispatcher](https://doc.akka.io/docs/akka/2.5/dispatchers.html?language=scala).

> 这里意思是如果针对长时间阻塞的任务，比如JDBC，像上述方式操作，我们只不过是把当前任务的执行放到了另外一条线程中继续阻塞了而已，因此仍然是假异步。这时候，我们可以定义自定义的`CustomExecutionContext`



```scala
import play.api.libs.concurrent.CustomExecutionContext

//请确保使用"Scala Dependency Injection"文档页面中列出的 custom binding 技术之一
//将 new context 绑定到当前 trait 
trait MyExecutionContext extends ExecutionContext

class MyExecutionContextImpl @Inject()(system: ActorSystem)
  extends CustomExecutionContext(system, "my.executor") with MyExecutionContext

class HomeController @Inject()(myExecutionContext: MyExecutionContext, val controllerComponents: ControllerComponents) extends BaseController {
  def index = Action.async {
    Future {
      // Call some blocking API
      Ok("result of blocking call")
    }(myExecutionContext)
  }
}
```



### How to create a Future[Result]

> 要创建一个 `Future[Result]`，我们首先需要另一个`future`, 这个 `future` 会给我们返回我们需要计算的实际的结果。

```scala
val futurePIValue: Future[Double] = computePIAsynchronously()
val futureResult: Future[Result] = futurePIValue.map { pi =>
  Ok("PI value computed: " + pi)
}
```

All of Play’s asynchronous API calls give you a `Future`. This is the case whether you are calling an external web service using the `play.api.libs.WS` API, or using Akka to schedule asynchronous tasks or to communicate with actors using `play.api.libs.Akka`.

> `PlayFramework` 所有异步的 `API` 返回的结果都是一个 `Future`，无论你是通过 `play.api.libs.WS` API 调用一个外部的 `web` 服务，或者使用Akka调度异步任务，或者使用 `play. API .lib .Akka`与 `actor` 通信等等，返回都是 `Future`。

下面是一个通过异步模式执行一段阻塞的代码，并返回一个 Future 简单的例子

```scala
val futureInt: Future[Int] = scala.concurrent.Future {
  intensiveComputation()
}
```

#### 注意点

It’s important to understand which thread code runs on with futures. In the two code blocks above, there is an import on Plays default execution context. This is an implicit parameter that gets passed to all methods on the future API that accept callbacks. The execution context will often be equivalent to a thread pool, though not necessarily.

> 理解哪些线程代码在 Future 上运行是很重要的。在上面的两个代码块中，在 Play 默认 executionContext 上有一个导入。这是一个隐式参数，传递给 future API上所有接受回调的方法。executionContext 通常等价于线程池，但也不一定。



You can’t magically turn synchronous IO into asynchronous by wrapping it in a `Future`. If you can’t change the application’s architecture to avoid blocking operations, at some point that operation will have to be executed, and that thread is going to block. So in addition to enclosing the operation in a `Future`, it’s necessary to configure it to run in a separate execution context that has been configured with enough threads to deal with the expected concurrency. See [Understanding Play thread pools](https://www.playframework.com/documentation/2.6.x/ThreadPools) for more information, and download the [play example templates](https://playframework.com/download#examples) that show database integration.

> 你不能通过将代码块包装在一个 Future下来神奇地把**同步IO**变成异步的。如果您不能更改应用程序的体系结构以避免阻塞操作，那么在某个时刻，该操作将不得不执行，而该线程将阻塞。
>
> 因此，除了将操作封装在 `Future` 中之外，还需要将其配置为在一个单独的 `executionContext` 中运行，该上下文中已经配置了足够的线程来处理预期的并发。有关更多信息，请参见[了解Play线程池](https://www.playframework.com/documentation/2.6.x/ThreadPools)，并下载显示数据库集成的[Play示例模板](https://playframework.com/download#example)



It can also be helpful to use Actors for blocking operations.

Actors provide a clean model for handling timeouts and failures, setting up blocking execution contexts, and managing any state that may be associated with the service. 

Also Actors provide patterns like `ScatterGatherFirstCompletedRouter` to address simultaneous cache and database requests and allow remote execution on a cluster of backend servers. But an Actor may be overkill depending on what you need.

> 对于阻塞操作场景，使用 Actors 模式是一个不错的选择。`Actors` 提供了一个简单一个简单的模型，来处理超时、故障、设置阻塞的 `executionContext` 以及与服务关联的任何状态。
>
> Actors 还提供了像 "ScatterGatherFirstCompletedRouter" 这样的模式来处理同步缓存和数据库请求，并允许在后端服务器集群上远程执行。但是一个 actor 可能会因为你的需要而 overkill。



### Returning futures

> 返回 futures

While we were using the `Action.apply` builder method to build actions until now, to send an asynchronous result we need to use the `Action.async` builder method:

当我们使用 `Action.apply` 时，将 `builder` 方法应用于 `actions` ，到目前为止，要发送异步结果，我们需要使用异步的 Actiton builder 方法

```scala
def index = Action.async {
  val futureInt = scala.concurrent.Future { intensiveComputation() }
  futureInt.map(i => Ok("Got result: " + i))
}
```



### Actions are asynchronous by default

> 默认情况下，`Actions` 是异步的。例如，在下面的控制器代码中，代码的`{Ok(…)}` 部分不是控制器的方法体。它是一个匿名函数，被传递给 `Action` 对象的 `apply`方法，该方法创建一个 `Action` 类型的对象。在内部，您编写的匿名函数将被调用，其结果将返回在 `Future`中。

```scala
def echo = Action { request =>
  Ok("Got request [" + request + "]")
}
```

**Note:** Both `Action.apply` and `Action.async` create `Action` objects that are handled internally in the same way. There is a single kind of `Action`, which is asynchronous, and not two kinds (a synchronous one and an asynchronous one). The `.async` builder is just a facility to simplify creating actions based on APIs that return a `Future`, which makes it easier to write non-blocking code.

#### 注意

`Action.apply` 和 `Action.async` 在内部都以相同的方式来处理 `Action` 对象。

有一种单独的 Action，它是异步的，但是不是a synchronous one and an asynchronous one 中的一种。

`.async` builder只是一个工具，用于在创建 `actions` 时基于返回 `Future` result 的 api 的操作进行简化，这使得编写非阻塞代码更加容易。



### Handling time-outs

> 处理超时情况

正确处理超时，避免 `web` 浏览器阻塞并在出现问题时等待，这通常很有用。您可以使用 `play.api.libs.concurrent.Futures` 来将非阻塞的超时包装在一个 `Futures` 中

```scala
import scala.concurrent.duration._
import play.api.libs.concurrent.Futures._

def index = Action.async {
  // 你可以隐式的提供一个超时参数，这题哦你歌唱可以通过controller的构造参数来达到。
  intensiveComputation().withTimeout(1.seconds).map { i =>
    Ok("Got result: " + i)
  }.recover {
    case e: scala.concurrent.TimeoutException =>
      InternalServerError("timeout")
  }
}
```

#### 注意

超时(Timeout)与取消(cancellation) 是不同的。对于超时而言，即使出现了超时，给定的 `Future` 仍然会完成，即使未返回已完成的值。









































