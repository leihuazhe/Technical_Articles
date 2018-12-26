# 🎮 Play 入门与学习（五）Dependency Injection

> 依赖注入是一种广泛使用的设计模式，有助于将组件的行为与依赖性解析分开。Play支持基于JSR 330（在本页中描述）的运行时依赖注入和在Scala中编译时依赖注入。
之所以调用运行时依赖项注入，是因为依赖关系图是在运行时创建，连接和验证的。 如果找不到特定组件的依赖项，则在运行应用程序之前不会出现错误。


Play支持Guice开箱即用，但可以插入其他JSR 330实现. [Guice wiki](https://github.com/google/guice/wiki/) 是一个很好的资源，可以更多地了解Guice和DI设计模式的功能。

注意：Guice是一个Java库，本文档中的示例使用Guice的内置Java API。 如果您更喜欢Scala DSL，您可能希望使用scala-guice或sse-guice库。

## Motivation

依赖注入实现了几个目标：

- 它允许您轻松绑定同一组件的不同实现。 这对于测试尤其有用，您可以使用模拟依赖项手动实例化组件或注入备用实现。
- 它允许您避免全局静态。 虽然静态工厂可以实现第一个目标，但您必须小心确保正确设置状态。 特别是Play（现已弃用）的静态API需要运行的应用程序，这使得测试的灵活性降低。 并且一次有多个实例可以并行运行测试。

`Guice wiki` 有一些很好的例子可以更详细地解释这一点。



## How it works

Play 提供了许多内置组件，并在诸如 `BuiltinModule` 之类的模块中声明它们。 这些绑定描述了创建 `Application` 实例所需的所有内容，默认情况下包括由 `routes compiler` 生成的 `route`，该 route 将控制器注入到构造函数中。 然后可以将这些绑定转换为在 `Guice` 和其他运行时DI框架中工作。

Play团队维护Guice模块，该模块提供 `GuiceApplicationLoader`。 这为 `Guice` 进行绑定转换，使用这些绑定创建Guice注入器，并从注入器请求Application实例。

There are also third-party loaders that do this for other frameworks, including [Scaldi ](https://github.com/scaldi/scaldi-play) and [Spring](https://github.com/remithieblin/play-spring-loader).

或者，Play提供了一个 `BuiltInComponents` 特性，允许您创建一个纯Scala实现，在编译时将您的应用程序连接在一起。

我们将在下面详细介绍如何自定义默认绑定和应用程序加载器。

## Declaring runtime DI dependencies

> 声明运行时DI依赖项

如果您有一个组件（例如控制器），并且它需要一些其他组件作为依赖项，那么可以使用 `@Inject` 注解来声明它。 `@Inject` 注解可用于字段或构造函数。 我们建议您在构造函数上使用它，例如：

```scala
import javax.inject._
import play.api.libs.ws._

class MyComponent @Inject() (ws: WSClient) {
  // ...
}
```

请注意，`@Inject` 注释必须位于类名之后但在构造函数参数之前，并且必须具有括号。

此外，Guice 确实提供了其他几种类型的注入方式，但构造函数注入通常是 `Scala` 中最清晰，简洁和可测试的，因此我们建议使用它。

`Guice` 能够在其构造函数上使用 `@Inject` 自动实例化任何类，而无需显式绑定它。此功能被称为  [just in time bindings](https://github.com/google/guice/wiki/JustInTimeBindings)，`Guice` 文档中对此进行了更详细的描述。 如果您需要执行更复杂的操作，可以声明自定义绑定，如下所述。



## Dependency injecting controllers

> 依赖注入控制器

There are two ways to make Play use dependency injected controllers.

有两种方法可以使 `Play` 使用依赖注入控制器。

### Injected routes generator

默认情况下（从2.5.0开始），`Play` 将生成一个 router ，该 router 将声明它所路由的所有控制器作为依赖项，允许您的控制器自己依赖注入。

要专门启用注入的路由生成器，请将以下内容添加到 `build.sbt` 中的构建设置：

```sbt
routesGenerator := InjectedRoutesGenerator
```

当使用 `injected routes generator` 时，为操作添加 `@` 符号前缀具有特殊含义，这意味着不是直接注入控制器，而是注入控制器的提供者。 例如，这允许 prototype controllers，以及用于打破循环依赖性的选项。

### Static routes generator

您可以将Play配置为使用 legacy（pre 2.5.0）静态路由生成器，该生成器假定所有操作都是静态方法。 要配置项目，请将以下内容添加到 `build.sbt`：

```sbt
routesGenerator := StaticRoutesGenerator
```

我们建议始终使用注入的路由生成器。 静态路由生成器主要作为辅助迁移的工具存在，因此现有项目不必一次使所有控制器不是静态的。

如果使用静态路由生成器，则可以通过在操作前加上@来指示操作具有注入的控制器，如下所示：

```routes
GET        /some/path           @controllers.Application.index
```

## Component lifecycle

依赖注入系统管理注入组件的生命周期，根据需要创建它们并将它们注入其他组件。以下是组件生命周期的工作原理：

- **每次需要组件时都会创建新实例**。如果组件多次使用，则默认情况下将创建组件的多个实例。如果您只需要组件的单个实例，则需要将其标记为单个实例。
- **实例在需要时会以懒加载的形式创建**。如果组件从未被其他组件使用，则根本不会创建它。这通常是你想要的。对于大多数组件而言，在需要之前创建它们是没有意义的。但是，在某些情况下，您希望直接启动组件，或者即使它们未被其他组件使用也是如此。例如，您可能希望在应用程序启动时向远程系统发送消息或预热缓存。您可以使用急切绑定 (`eager binding`) 强制创建组件。
- **除了正常的垃圾收集之外，实例不会自动清理**。当组件不再被引用时，它们将被垃圾收集，但框架将不会执行任何特殊操作来关闭组件，例如调用 `close` 方法。但是，`Play` 提供了一种特殊类型的组件，称为`ApplicationLifecycle`，它允许您注册组件以在应用程序停止时关闭。

## Singletons

有时，您可能拥有一个包含某些状态的组件，例如缓存，或者与外部资源的连接，或者创建组件可能很昂贵。 在这些情况下，该组件应该是单例的。 这可以使用 `@Singleton` 注解来实现：

```scala
import javax.inject._

@Singleton
class CurrentSharePrice {
  @volatile private var price = 0

  def set(p: Int) = price = p
  def get = price
}
```

## Stopping/cleaning up

`Play` 关闭时可能需要清理某些组件，例如，停止线程池。 Play提供了一个 `ApplicationLifecycle` 组件，可用于在 `Play` 关闭时注册挂钩以停止组件

```scala
import scala.concurrent.Future
import javax.inject._
import play.api.inject.ApplicationLifecycle

@Singleton
class MessageQueueConnection @Inject() (lifecycle: ApplicationLifecycle) {
  val connection = connectToMessageQueue()
  lifecycle.addStopHook { () =>
    Future.successful(connection.stop())
  }

  //...
}
```

`ApplicationLifecycle` 将在创建时以相反的顺序停止所有组件。 这意味着您依赖的任何组件仍然可以安全地用在组件的停止钩子上。 因为您依赖它们，所以它们必须在组件之前创建，因此在组件停止之前不会停止。

注意：确保注册 `stop hook` 的所有组件都是单例非常重要。 注册 `stop` 钩子的任何非单例组件都可能是内存泄漏的来源，因为每次创建组件时都会注册一个新的钩子。



## Providing custom bindings

> 提供自定义绑定

为组件定义 `trait` 被认为是一种好习惯，并且其他类依赖于该 `trait`，而不是组件的实现。 通过这样做，您可以注入不同的实现，例如，在测试应用程序时注入模拟实现。

在这种情况下，DI系统需要知道哪个实现应该绑定到该 `trait`。 我们建议您声明这一点的方式取决于您是将 `Play` 应用程序编写为 `Play` 的最终用户，还是编写其他 `Play` 应用程序将使用的库。

### Play applications

我们建议 `Play` 应用程序使用应用程序正在使用的 `DI` 框架提供的任何机制。 尽管Play确实提供了绑定API，但此API有些限制，并且不允许您充分利用您正在使用的框架的强大功能。

由于Play为Guice提供了开箱即用的支持，下面的示例显示了如何为 `Guice` 提供绑定。

#### Binding annotations

将实现绑定到接口的最简单方法是使用 `Guice` `@ImplementedBy` 注释。 例如：

```scala
import com.google.inject.ImplementedBy

@ImplementedBy(classOf[EnglishHello])
trait Hello {
  def sayHello(name: String): String
}

class EnglishHello extends Hello {
  def sayHello(name: String) = "Hello " + name
}
```

#### Programmatic bindings

在一些更复杂的情况下，您可能希望提供更复杂的绑定，例如当您有一个 `trait` 的多个实现时，这些 `trait` 由 `@Named` 注释限定。 在这些情况下，您可以实现自定义 `Guice` 模块

```scala
import com.google.inject.AbstractModule
import com.google.inject.name.Names

class Module extends AbstractModule {
  def configure() = {

    bind(classOf[Hello])
      .annotatedWith(Names.named("en"))
      .to(classOf[EnglishHello])

    bind(classOf[Hello])
      .annotatedWith(Names.named("de"))
      .to(classOf[GermanHello])
  }
}
```

如果您将此模块称为 `Module` 并将其放在根包中，它将自动注册到 `Play`。 或者，如果要为其指定不同的名称或将其放在不同的包中，可以通过将其全限定名附加到 `application.conf` 中的 `play.modules.enabled` 列表来将其注册到 `Play`

```
play.modules.enabled += "modules.HelloModule"
```

您还可以通过将以下模块添加到已禁用的模块来禁用根软件包中名为 `Module` 的模块的自动注册

```
play.modules.disabled += "Module"
```

#### Configurable bindings

> 可配置的绑定

有时您可能希望在配置 `Guice` 绑定时读取 Play `Configuration` 或使用 `ClassLoader`。 您可以通过将这些对象添加到模块的构造函数来访问这些对象。

在下面的示例中，从配置文件中读取每种语言的 `Hello` 绑定。 这允许通过在 `application.conf` 文件中添加新设置来添加新的 `Hello` 绑定。

```scala
import com.google.inject.AbstractModule
import com.google.inject.name.Names
import play.api.{ Configuration, Environment }

class Module(environment: Environment,configuration: Configuration) 
															extends AbstractModule {
  def configure() = {
    // Expect configuration like:
    // hello.en = "myapp.EnglishHello"
    // hello.de = "myapp.GermanHello"
    val helloConfiguration: Configuration =
      configuration.getOptional[Configuration]("hello").getOrElse(Configuration.empty)
    val languages: Set[String] = helloConfiguration.subKeys
    // Iterate through all the languages and bind the
    // class associated with that language. Use Play's
    // ClassLoader to load the classes.
    for (l <- languages) {
      val bindingClassName: String = helloConfiguration.get[String](l)
      val bindingClass: Class[_ <: Hello] =
        environment.classLoader.loadClass(bindingClassName)
        .asSubclass(classOf[Hello])
      bind(classOf[Hello])
        .annotatedWith(Names.named(l))
        .to(bindingClass)
    }
  }
}
```

注意：在大多数情况下，如果在创建组件时需要访问 `Configuration`，则应将 `Configuration` 对象注入组件本身或组件的 `Provider` 中。 然后，您可以在创建组件时读取“配置”。 在为组件创建绑定时，通常不需要读取`Configuration`。

#### Eager bindings

> 提前绑定

在上面的代码中，每次使用时都会创建新的 `EnglishHello` 和 `GermanHello` 对象。 如果您只想创建一次这些对象，可能因为创建它们很昂贵，那么您应该使用 `@Singleton` 注释。 如果你想创建它们一次并在应用程序启动时急切地创建它们，而不是在需要它们懒加载，那么你就可以使用  [Guice’s eager singleton binding](https://github.com/google/guice/wiki/Scopes#eager-singletons)



```scala
import com.google.inject.AbstractModule
import com.google.inject.name.Names

// A Module is needed to register bindings
class Module extends AbstractModule {
  override def configure() = {

    // Bind the `Hello` interface to the `EnglishHello` implementation as eager singleton.
    bind(classOf[Hello])
      .annotatedWith(Names.named("en"))
      .to(classOf[EnglishHello]).asEagerSingleton()

    bind(classOf[Hello])
      .annotatedWith(Names.named("de"))
      .to(classOf[GermanHello]).asEagerSingleton()
  }
}
```

当应用程序启动时，可以使用 `Eager` 单例来启动服务。 它们通常与关闭钩子组合在一起，以便服务可以在应用程序停止时清理其资源。

```scala
import scala.concurrent.Future
import javax.inject._
import play.api.inject.ApplicationLifecycle

// This creates an `ApplicationStart` object once at start-up and registers hook for shut-down.
@Singleton
class ApplicationStart @Inject() (lifecycle: ApplicationLifecycle) {
  // Shut-down hook
  lifecycle.addStopHook { () =>
    Future.successful(())
  }
  //...
}
```

```scala
import com.google.inject.AbstractModule

class StartModule extends AbstractModule {
  override def configure() = {
    bind(classOf[ApplicationStart]).asEagerSingleton()
  }
}
```

### Play libraries

如果您正在为Play实现一个库，那么您可能希望它与DI框架无关，这样无论在应用程序中使用哪个DI框架，您的库都可以开箱即用。 出于这个原因，Play提供了一个轻量级绑定API，用于以DI框架无关的方式提供绑定。

要提供绑定，请实现Module以返回要提供的绑定序列。 Module trait还提供用于构建绑定的DSL：

```scala
import play.api.inject._

class HelloModule extends Module {
  def bindings(environment: Environment,
               configuration: Configuration) = Seq(
    bind[Hello].qualifiedWith("en").to[EnglishHello],
    bind[Hello].qualifiedWith("de").to[GermanHello]
  )
}
```

通过将该模块附加到reference.conf中的play.modules.enabled列表，可以自动注册该模块：

```
play.modules.enabled += "com.example.HelloModule"
```

Module bindings方法采用Play环境和配置。 如果要动态配置绑定，可以访问这些。
模块绑定支持急切绑定。 要声明一个急切绑定，请在绑定结束时添加.eagerly。



为了最大化跨框架兼容性，请记住以下事项：

- 并非所有DI框架都只支持时间绑定。 确保明确绑定库提供的所有组件。
- 尝试保持绑定键简单 - 不同的运行时DI框架对键是什么以及它应该如何唯一或不唯一有不同的看

### Excluding modules

如果有一个您不想加载的模块，可以通过将其附加到application.conf中的play.modules.disabled属性来将其排除：

```
play.modules.disabled += "play.api.db.evolutions.EvolutionsModule"
```



## Managing circular dependencies

当您的某个组件依赖于依赖于原始组件的另一个组件（直接或间接）时，就会发生循环依赖关系。 例如：

```scala
import javax.inject.Inject

class Foo @Inject() (bar: Bar)
class Bar @Inject() (baz: Baz)
class Baz @Inject() (foo: Foo)
```

在这种情况下，Foo依赖于Bar，它取决于Baz，它依赖于Foo。 因此，您将无法实例化任何这些类。 您可以使用提供程序解决此问题：

```scala
import javax.inject.{ Inject, Provider }

class Foo @Inject() (bar: Bar)
class Bar @Inject() (baz: Baz)
class Baz @Inject() (foo: Provider[Foo])
```

通常，可以通过以更原子的方式分解组件或查找要依赖的更具体的组件来解决循环依赖性。 常见问题是对Application的依赖。 当你的组件依赖于应用程序时，它说它需要一个完整的应用程序来完成它的工作; 通常情况并非如此。 您的依赖项应该位于具有您需要的特定功能的更具体的组件（例如，环境）上。 作为最后的手段，您可以通过注入Provider [Application]来解决问题。



## Advanced: Extending the GuiceApplicationLoader



Play的运行时依赖注入由GuiceApplicationLoader类引导。 该类加载所有模块，将模块提供给Guice，然后使用Guice创建应用程序。 如果要控制Guice如何初始化应用程序，则可以扩展GuiceApplicationLoader类。

您可以覆盖几种方法，但通常需要覆盖构建器方法。 此方法读取ApplicationLoader.Context并创建GuiceApplicationBuilder。 您可以在下面看到构建器的标准实现，您可以按照自己喜欢的方式进行更改。 您可以在有关使用Guice进行测试的部分中找到如何使用GuiceApplicationBuilder。



```scala
import play.api.ApplicationLoader
import play.api.Configuration
import play.api.inject._
import play.api.inject.guice._

class CustomApplicationLoader extends GuiceApplicationLoader() {
  override def builder(context: ApplicationLoader.Context): GuiceApplicationBuilder = {
    val extra = Configuration("a" -> 1)
    initialBuilder
      .in(context.environment)
      .loadConfig(extra ++ context.initialConfiguration)
      .overrides(overrides(context): _*)
  }
}
```

当您覆盖ApplicationLoader时，您需要告诉Play。 将以下设置添加到application.conf：

play.application.loader =“modules.CustomApplicationLoader”
您不仅限于使用Guice进行依赖注入。 通过重写ApplicationLoader，您可以控制应用程序的初始化方式。 在下一节中了解更多信息。