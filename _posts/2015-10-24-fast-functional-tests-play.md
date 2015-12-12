---
layout: post
title: Writing faster functional tests for Play applications
---

Play Framework supports functional testing out of the box. There are helpers for both
Scala ([specs2](https://www.playframework.com/documentation/2.4.x/ScalaFunctionalTestingWithSpecs2), 
[ScalaTest](https://www.playframework.com/documentation/2.4.x/ScalaFunctionalTestingWithScalaTest)) and Java
[JUnit](https://www.playframework.com/documentation/2.4.x/JavaFunctionalTest). The basic idea is to run the code under
test inside a "fake" application. For the code it will look like it is running in a normal Play application with access
to plugins, configuration parameters and other parts of the runtime environment. A fake application can be started with
or without a HTTP server. Everything works very well, but there is one issue - instantiation of a fake application takes
a long time. The reason is the fake application is in fact quite real and does a lot of what a real application would do
(read the configuration, load the plugins etc). Fake application startup time matters because the helpers provided by
Play assume that each test requires it's own application. As the number of functional tests grows the fake application
start time overhead becomes significant. This concern is somewhat addressed in ScalaTest (multiple tests can share the
same fake application), but not for other testing frameworks.

In this post I will show a different approach to functional testing of Play applications. Instead of using multiple fake
applications we will run tests against a single instance of a real application.

Let's start from a simple play application that we can use for testing;

```scala
package net.yefremov.sample

import play.api.mvc._

object Application extends Controller {

  def foo = Action {
    Ok("foo")
  }

  def bar = Action {
    Ok("bar")
  }
}
```

And here is the corresponding routes file:

```
GET     /foo                    net.yefremov.sample.Application.foo
GET     /bar                    net.yefremov.sample.Application.bar
```

Now we have a very simple application that returns "foo" on /foo and "bar" on /bar. Let's create a functional test for that.

Before we create a test we need to change the build to support functional testing. That is not strictly required, but I
prefer to clearly separate unit and functional tests. In SBT it is typically done using the "it" configuration. To get it 
working modify your `build.sbt` to contain the following items:

```scala
Defaults.itSettings

unmanagedSourceDirectories in IntegrationTest <<=
    (baseDirectory in IntegrationTest)(base =>  Seq(base / "it")),

libraryDependencies +=
    "com.typesafe.play" %% "play-test" % play.core.PlayVersion.current % "it",

lazy val root = project.in(file(".")).configs(IntegrationTest)
```

With the above changes applied we can have unit tests in the default `test` folder and functional test in the `it` folder.
To execute functional tests you can run `play it:test`.
 
Now let's create a simple functional test.

```scala
@RunWith(classOf[JUnitRunner])
class IntegrationSpec extends Specification with FutureAwaits with DefaultAwaitTimeout {

  val baseUrl = "http://localhost:9000"

  "application" should {

    "return 'foo' from /foo" in {
      val response = await(WS.url(s"$baseUrl/foo").get())
      response.body must beEqualTo("foo")
    }

    "return 'bar' from /bar" in {
      val response = await(WS.url(s"$baseUrl/bar").get())
      response.body must beEqualTo("bar")
    }
  }
}
```

The test uses [WS API](https://www.playframework.com/documentation/2.4.x/ScalaWS) to hit the application
via the HTTP interface. One important part to notice is it does not use
[WithServer](https://www.playframework.com/documentation/2.4.x/api/scala/index.html#play.api.test.WithServer) to start a
fake application. It requires you to start the application yourself before running the test. Thus the test will fail if
executed by simply running `play it:test`.
 
In order to make the test pass we need to start our application before running the test and shut the application down
after the test finishes. It could be done manually, but there is a better way. There are very convenient hooks available
in SBT: [sbt.Tests#Setup](http://www.scala-sbt.org/0.13/api/index.html#sbt.Tests$$Setup) and
[sbt.Tests#Cleanup](http://www.scala-sbt.org/0.13/api/index.html#sbt.Tests$$Cleanup). We will use them to start and stop
the application when running integration tests. We can see how it works by adding the following to `build.sbt`:

```scala
testOptions in IntegrationTest += Tests.Setup(() => println("setup"))

testOptions in IntegrationTest += Tests.Cleanup(() => println("cleanup"))
```

Now when you run `play it:test` you will see the messages above printed in the console. Next we need to replace simple
debug messages with real code to start and stop the application. There is just one issue with that. We can not directly
run `play run` using [ProcessBuilder](http://www.scala-sbt.org/0.13/api/index.html#sbt.ProcessBuilder). That will block
execution of tests until the app shuts down. The easiest solution for that is to use
[Unix `screen` command](https://www.gnu.org/software/screen/manual/screen.html).

```scala
"screen -dmSL playFunctionalTest play run".run()
```

To stop the application under test we will find the corresponding screen and kill it.

```scala
"screen -S playFunctionalTest -X quit".run()
```

This is pretty much it. One last part that is left is to wait for the application to start up before executing tests. It
can be done by hitting an application URL in a loop until the application responds. This will also help to warm up the
application because Play only compiles code after the first request hits the application.

```scala
private def isAppRunning(appUrl: URL): Boolean = {
  try {
    val connection = appUrl.openConnection().asInstanceOf[HttpURLConnection]
    connection.setRequestMethod("GET")
    connection.connect()
    true
  } catch {
    case NonFatal(e) =>
      println(s"${e.getClass.getSimpleName}: ${e.getMessage}")
      false
  }
}
```

After putting everything together we can run `play it:test` and see our tests passing.

```
[play-functional-testing] $ it:test
Launching the app...
screen -dmSL playFuncTest play run -Dhttp.port=9000
Waiting for the app to start up...
ConnectException: Connection refused
Waiting for the app to start up...
ConnectException: Connection refused
Waiting for the app to start up...
ConnectException: Connection refused
Waiting for the app to start up...
The app is now ready
[info] IntegrationSpec
[info] application should
[info] + return 'foo' from /foo
[info] + return 'bar' from /bar
[info] Total for specification IntegrationSpec
[info] Finished in 18 ms
[info] 2 examples, 0 failure, 0 error
Killing the app...
screen -S playFuncTest -X quit
Waiting for the app to shutdown...
Waiting for the app to shutdown...
ConnectException: Connection refused
[info] Passed: Total 2, Failed 0, Errors 0, Passed 2
[success] Total time: 22 s, completed Dec 12, 2015 2:05:22 PM
```

Complete sample application code can bound here: https://github.com/dmitriy-yefremov/play-functional-testing

Please let me know what you think and how this solution can be improved!