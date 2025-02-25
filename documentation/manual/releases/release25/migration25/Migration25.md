<!--- Copyright (C) 2009-2016 Typesafe Inc. <http://www.typesafe.com> -->
# Play 2.5 Migration Guide

This is a guide for migrating from Play 2.4 to Play 2.5. If you need to migrate from an earlier version of Play then you must first follow the [[Play 2.4 Migration Guide|Migration24]].

As well as the information contained on this page, there are is more detailed migration information for some topics:

- [[Streams Migration Guide|StreamsMigration25]] – Migrating to Play's new streaming library.
- [[Java Migration Guide|JavaMigration25]] - Migrating Java applications.

**TODO: Review all headings to make them clear, succinct and consistent. If heading names change, check that backlinks from the Higlights still work.**

**TODO: Review all sections and make sure they have the following structure: a high level description at the start and then a practical *How to migrate* section explaining concrete steps that are needed.**

**TODO: Arrange sections into priority order so that the bits that most people need to know about come first. Migration that's necessary can come first, migration that's optional (e.g. deprecation) can come later. Consider putting large sections (e.g. Java 8 migration, Akka streams migration) onto separate pages.**

## sbt upgrade to 0.13.9

Play 2.5 now requires a minimum of sbt 0.13.9. The 0.13.9 release of sbt has a number of [improvements and bug fixes](https://github.com/sbt/sbt/releases/tag/v0.13.9).

### How to migrate

Update your `project/build.properties` so that it reads:

```
sbt.version=0.13.9
```

## Scala 2.10 support discontinued

Play 2.3 and 2.4 supported both Scala 2.10 and 2.11. Play 2.5 has dropped support for Scala 2.10 and now only supports Scala 2.11. There are a couple of reasons for this:

1. Play 2.5's internal code makes extensive use of the [scala-java8-compat](https://github.com/scala/scala-java8-compat) library, which only supports Scala 2.11. The *scala-java8-compat* has conversions between many Scala and Java 8 types, such as Scala `Future`s and Java `CompletionStage`s. (You might find this library useful for your code too.)

2. The next version of Play will probably add support for Scala 2.12. It's time for Play to move to Scala 2.11 so that the upcoming transition to 2.12 will be easier.

### How to migrate

**TODO: Describe changes to build.sbt needed to upgrade a Play project from Scala 2.10 to Scala 2.11, emphasize that both Java and Scala users may need to change this setting, link to any docs that describe what's new in Scala 2.11**


## Change to Logback configuration

As part of the change to remove Play's hardcoded dependency on Logback [[(see Highlights)|Highlights25#Support-for-other-logging-frameworks]], one of the classes used by Logback configuration had to be moved to another package.

### How to migrate

You will need to update your Logback configuration files (`logback*.xml`) and change any references to the old `play.api.Logger$ColoredLevel` to the new `play.api.libs.logback.ColoredLevel` class.

The new configuration after the change will look something like this:

```xml
<conversionRule conversionWord="coloredLevel" converterClass="play.api.libs.logback.ColoredLevel" />
```

You can find more details on how to set up Play with different logging frameworks are in [[Configuring logging|SettingsLogger#Using-a-Custom-Logging-Framework]] section of the documentation.


## Upgrade to AsyncHttpClient 2

Play WS has been upgraded to use [AsyncHttpClient](https://github.com/AsyncHttpClient/async-http-client) 2.  This is a major upgrade that uses Netty 4.0. Most of the changes in AHC 2.0 are under the hood, but AHC has some significant refactorings which require breaking changes to the WS API:

* [`AsyncHttpClientConfig`] replaced by [`DefaultAsyncHttpClientConfig`](https://static.javadoc.io/org.asynchttpclient/async-http-client/2.0.0-RC7/org/asynchttpclient/DefaultAsyncHttpClientConfig.html).
* [`allowPoolingConnection`](https://static.javadoc.io/com.ning/async-http-client/1.9.32/com/ning/http/client/AsyncHttpClientConfig.html#allowPoolingConnections) and `allowSslConnectionPool` are combined in AsyncHttpClient into a single `keepAlive` variable.  As such, `play.ws.ning.allowPoolingConnection` and `play.ws.ning.allowSslConnectionPool` are not valid and will throw an exception if configured.  
* [`webSocketIdleTimeout`](https://static.javadoc.io/com.ning/async-http-client/1.9.32/com/ning/http/client/AsyncHttpClientConfig.html#webSocketTimeout) has been removed, so is no longer available in `AhcWSClientConfig`.
* [`ioThreadMultiplier`](https://static.javadoc.io/com.ning/async-http-client/1.9.32/com/ning/http/client/AsyncHttpClientConfig.html#ioThreadMultiplier) has been removed, so is no longer available in `AhcWSClientConfig`.
* [`FluentCaseInsensitiveStringsMap`](https://static.javadoc.io/com.ning/async-http-client/1.9.32/com/ning/http/client/FluentCaseInsensitiveStringsMap.html) class is removed and replaced by Netty's `HttpHeader` class.
* [`Realm.AuthScheme.None`](https://static.javadoc.io/com.ning/async-http-client/1.9.32/com/ning/http/client/Realm.AuthScheme.html#NONE) has been removed, so is no longer available in `WSAuthScheme`.

In addition, there are number of small changes:

* In order to reflect the proper AsyncHttpClient library name, package `play.api.libs.ws.ning` was renamed into `play.api.libs.ws.ahc` and `Ning*` classes were renamed into `Ahc*`.  In addition, the AHC configuration settings have been changed to `play.ws.ahc` prefix, i.e. `play.ws.ning.maxConnectionsPerHost` is now `play.ws.ahc.maxConnectionsPerHost`.
* The deprecated interface `play.libs.ws.WSRequestHolder` has been removed.
* The `play.libs.ws.play.WSRequest` interface now returns `java.util.concurrent.CompletionStage` instead of `F.Promise`.
* Static methods that rely on `Play.current` or `Play.application` have been deprecated.

### How to migrate

**TODO: Check the list of classes and config settings that have changed and document them here. Check which bits are deprecated. I think config using the old name will give a warning at runtime.** More info in the PR: https://github.com/playframework/playframework/pull/5224. **Some text we might want to use:** "package `play.api.libs.ws.ning` was renamed into `play.api.libs.ws.ahc` and `Ning*` classes were renamed into `Ahc*`."


## Deprecated `GlobalSettings`

**TODO: Describe deprecation and the rationale**

### Migration

**TODO: Explain how to migrate to a DI alternative, providing links**


### How to migrate

**TODO: Explain how to change iteratees to streams. Explain the deprecation/removal plan for iteratees. Do we need separate sections for result streaming and HttpEntities, body parsers and Accumulators, WS and WebSockets?**


## Removed Plugins API

The Plugins API was deprecated in Play 2.4 and has been removed in Play 2.5. The Plugins API has been superceded by Play's dependency injection and module system which provides a cleaner and more flexible way to build reusable components.

### How to migrate

Read about how to create reusable components using dependency injection in either [[Scala|ScalaDependencyInjection]] or [[Java|JavaDependencyInjection]].

A plugin will usually be a class that is:

* a singleton ([[Java|JavaDependencyInjection#Singletons]] / [[Scala|ScalaDependencyInjection#Singletons]])
* is a dependency for some classes and has dependencies on other components ([[Java|JavaDependencyInjection#Declaring-dependencies]] / [[Scala|ScalaDependencyInjection#Declaring-dependencies]])
* optionally bound in a module ([[Java|JavaDependencyInjection#Programmatic-bindings]] / [[Scala|ScalaDependencyInjection#Programmatic-bindings]])
* optionally started when the application starts ([[Java|JavaDependencyInjection#Eager-bindings]] / [[Scala|ScalaDependencyInjection#Eager-bindings]])
* optionally stopped when the application stops ([[Java|JavaDependencyInjection#Stopping/cleaning-up]] / [[Scala|ScalaDependencyInjection#Stopping/cleaning-up]])

> Note: As part of this effort, the [[modules directory|ModuleDirectory]] has been updated to only include up-to-date modules that do not use the Plugin API. If you have any corrections to make to the module directory please let us know.

## Routes generated with InjectedRoutesGenerator

Routes are now generated using the dependency injection aware `InjectedRoutesGenerator`, rather than the previous `StaticRoutesGenerator` which assumed controllers were singleton objects.  

To revert back to the earlier behavior (if you have "object MyController" in your code, for example), please add:

```
routesGenerator := StaticRoutesGenerator
```

to your `build.sbt` file.

## Replaced static controllers with dependency injection

`controllers.ExternalAssets` is now a class, and has no static equivalent. `controllers.Assets` and `controllers.Default` are also classes, and while static equivalents exist, it is recommended that you use the class version.

### How to migrate

The recommended solution is to use classes for all your controllers. The `InjectedRoutesGenerator` is now the default, so the controllers in the routes file are assumed to be classes instead of objects.

If you still have static controllers, you can use `StaticRoutesGenerator` (described above) and add the `@` symbol in front of the route in the `routes` file, e.g.

```
GET  /assets/*file  @controllers.ExternalAssets.at(path = "/public", file)
```

## Deprecated play.Play and play.api.Play methods

The following methods have been deprecated in `play.Play`:

* `public static Application application()`
* `public static Mode mode()`
* `public static boolean isDev()`
* `public static boolean isProd()`
* `public static boolean isTest()`

Likewise, methods in `play.api.Play` that take an implicit `Application` and delegate to Application, such as `def classloader(implicit app: Application)` are now deprecated.

### How to migrate

These methods delegate to either `play.Application` or `play.Environment` -- code that uses them should use dependency injection to inject the relevant class.

You should refer to the list of dependency injected components in the [[Play 2.4 Migration Guide|Migration24#Dependency-Injected-Components]] to migrate built-in play components.

For example, the following code injects an environment and configuration into a Controller in Scala:

```
class HomeController @Inject() (environment: play.api.Environment, configuration: play.api.Configuration) extends Controller {

  def index = Action {
    Ok(views.html.index("Your new application is ready."))
  }

  def config = Action {
    Ok(configuration.underlying.getString("some.config"))
  }

  def count = Action {
    val num = environment.resource("application.conf").toSeq.size
    Ok(num.toString)
  }
}
```

### Handling legacy components

Generally the components you use should not need to depend on the entire application, but sometimes you have to deal with legacy components that require one. You can handle this by injecting the application into one of your components:

```
class FooController @Inject() (implicit app: Application) extends Controller {
  def bar = Action {
    Ok(Foo.bar(app))
  }
}
```

Even better, you can make your own `*Api` class that turns the static methods into instance methods:

```
class FooApi @Inject() (implicit app: Application) {
  def bar = Foo.bar(app)
  def baz = Foo.baz(app)
}
```

This allows you to benefit from the testability you get with DI and still use your library that uses global state.

## CSRF filter changes

In order to make Play's CSRF filter more resilient to browser plugin vulnerabilities and new extensions, the default configuration for the CSRF filter has been made far more conservative.  The changes include:

* Instead of blacklisting `POST` requests, now only `GET`, `HEAD` and `OPTIONS` requests are whitelisted, and all other requests require a CSRF check.  This means `DELETE` and `PUT` requests are now checked.
* Instead of blacklisting `application/x-www-form-urlencoded`, `multipart/form-data` and `text/plain` requests, requests of all content types, including no content type, require a CSRF check.  One consequence of this is that AJAX requests that use `application/json` now need to include a valid CSRF token in the `Csrf-Token` header.
* Stateless header-based bypasses, such as the `X-Request-With`, are disabled by default.

There's a new config option to bypass the new CSRF protection for requests with certain headers. This config option is turned on by default for the Cookie and Authorization headers, so that REST clients, which typically don't use session authentication, will still work without having to send a CSRF token.

However, since the config option allows through *all* requests without those headers, applications that use other authentication schemes (NTLM, TLS client certificates) will be vulnerable to CSRF. These applications should disable the config option so that their authenticated (cookieless) requests are protected by the CSRF filter.

Finally, an additional option has been added to disable the CSRF check for origins trusted by the CORS filter. Please note that the CORS filter must come *before* the CSRF filter in your filter chain for this to work!

Play's old default behaviour can be restored by adding the following configuration to `application.conf`:

```
play.filters.csrf {
  header {
    bypassHeaders {
      X-Requested-With = "*"
      Csrf-Token = "nocheck"
    }
    protectHeaders = null
  }
  bypassCorsTrustedOrigins = false
  method {
    whiteList = []
    blackList = ["POST"]
  }
  contentType.blackList = ["application/x-www-form-urlencoded", "multipart/form-data", "text/plain"]
}
```

For more details, please read the CSRF documentation for [[Java|JavaCsrf]] and [[Scala|ScalaCsrf]].
