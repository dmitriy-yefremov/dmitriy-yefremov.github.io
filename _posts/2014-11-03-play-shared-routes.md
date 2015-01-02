---
layout: post
title: Sharing reverse router across multiple Play projects
---

I was working on a couple of fairly complicated Play application recently. In order to keep complexity at an acceptably
low level we broke the apps into sub-modules. Every sub-module is a Play application on it's own and can be worked on,
launched and tested separately. Then there is an aggregating app that takes care of making everyone play together.
More about composing Play apps out of individual modules [here](http://engineering.linkedin.com/play/composable-and-streamable-play-apps)
and [here](https://www.playframework.com/documentation/2.2.x/SBTSubProjects).

This is how the project structure may look like:

```
main-app
  └ app
    └ controllers
      └ MainController.scala
  └ conf
    └ application.conf
    └ routes
module-foo
  └ app
    └ controllers
      └ FooController.scala    
  └ conf
    └ foo.routes    
module-bar
  └ app
    └ controllers
      └ BarController.scala    
  └ conf
    └ bar.routes            
project
  └ Build.scala 
 
``` 

There is one issue with this approach. Individual modules are fully isolated (and that's what we are after), but sometimes
they need to generate links to each other. Routes files are also split per module so there is no way to access other
module's reverse router. There is an [issue](https://github.com/playframework/playframework/issues/1390) opened for that
quite some time ago. But at the time of writing it is not resolved yet.

When I started to search for potential solutions I found just one suggested by [@godenji](https://github.com/godenji).
The solution introduces it's own parser for Play routes files and also a code generator to create reverse router objects.
I was concerned about having a custom parser so decided to try to find another solution myself.

Here is what I came up with. It's not perfect, but may work well in some cases and only uses Play's parsers/generators.

The idea is to extract all routing into a separate library project that is shared across all modules. It only contains
routes files and have no dependencies (but every module depends on it). There is a few tricks to make it working.

First, the routes compiler needs to have access to the controllers in order to validate the routes and generate a router.
Just create traits for controllers that need to be linked from other modules and put them in the routes project. 

Here is how the project looks now.
 
```
main-app
  └ app
    └ controllers
      └ MainController.scala
  └ conf
    └ application.conf
    └ routes
module-foo
  └ app
    └ controllers
      └ FooController.scala    
module-bar
  └ app
    └ controllers
      └ BarController.scala       
routes
  └ app
    └ controllers
      └ FooControllerApi.scala
      └ BarControllerApi.scala             
  └ conf
    └ foo.routes    
    └ bar.routes    
project
  └ Build.scala 
 
``` 

Second, you cannot use traits in a routes file, only concrete objects. This can be solved by using Play's managed controllers.
The feature was developed to support [dependency injection](https://www.playframework.com/documentation/2.2.x/ScalaDependencyInjection)
in Play, but works well for us.

In order to use a trait in a routes file add `@` to the route.

```
GET       /foo/hello        @FooControllerApi.helloFoo(name)
GET       /bar/hello        @BarControllerApi.helloBar(name)
```

When the compiler sees a route like that it will add a call to
[play.api.GlobalSettings#getControllerInstance](https://www.playframework.com/documentation/2.2.x/api/scala/index.html#play.api.GlobalSettings)
to get an instance implementing the trait to delegate the call to. So we need to implement this call.

```scala
object Global extends GlobalSettings {

  private val controllerMapping = Map[Class[_], Controller](
    classOf[FooControllerApi] -> FooController,
    classOf[BarControllerApi] -> BarController
  )

  override def getControllerInstance[T](controllerClass: Class[T]): T = {
    controllerMapping(controllerClass).asInstanceOf[T]
  }

}

```

This is pretty much it. You can find a sample application [here](https://github.com/dmitriy-yefremov/play-shared-routes).

Again, this approach is not perfect. It requires some overhead: a separate project for routes, traits for controller interfaces
and mapping of trait classes to corresponding instance objects. The latter can be solved by a classpath scanner that would find
the implementations at start up time and generate the map automatically.

Please let me know what do you think.
