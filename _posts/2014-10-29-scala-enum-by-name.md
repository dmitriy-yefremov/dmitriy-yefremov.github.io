---
layout: post
title: Retrieving Scala enumeration constants by name
---

In many cases you may need to get an enumeration constant by it's name. It is very easy to do it if you extend the standard Scala [Enumeration](http://www.scala-lang.org/api/current/scala/Enumeration.html) class to create your enumeration. Below is a typical enumeration implementation:

```scala
object FunninessLevel extends Enumeration {
  type FunninessLevel = Value
  val LOL, ROFL, LMAO = Value
}
```

To retrieve a constant by it's name you simply call `withName` on the object:

```scala
val level = FunninessLevel.withName("LOL")
```

So far so good?

It gets more difficult if you need to retrieve a constant for any given enumeration type at run time (e.g. JSON serialization). Java has a very convenient way of doing it: [Enum.valueOf](http://docs.oracle.com/javase/7/docs/api/java/lang/Enum.html#valueOf(java.lang.Class,%20java.lang.String)). But I could find nothing like that for Scala. So I ended up building my own helpers.

```scala
import scala.reflect.runtime.universe._

/**
 * Scala [[Enumeration]] helpers implementing Scala versions of
 * Java's [[java.lang.Enum.valueOf(Class[Enum], String)]].
 * @author Dmitriy Yefremov
 */
object EnumReflector {

  private val mirror: Mirror = runtimeMirror(getClass.getClassLoader)

  /**
   * Returns a value of the specified enumeration with the given name.
   * @param name value name
   * @tparam T enumeration type
   * @return enumeration value, see [[scala.Enumeration.withName(String)]]
   */
  def withName[T <: Enumeration#Value: TypeTag](name: String): T = {
    typeOf[T] match {
      case valueType @ TypeRef(enumType, _, _) =>
        val methodSymbol = factoryMethodSymbol(enumType)
        val moduleSymbol = enumType.termSymbol.asModule
        reflect(moduleSymbol, methodSymbol)(name).asInstanceOf[T]
    }
  }

  /**
   * Returns a value of the specified enumeration with the given name.
   * @param clazz enumeration class
   * @param name value name
   * @return enumeration value, see [[scala.Enumeration#withName(String)]]
   */
  def withName(clazz: Class[_], name: String): Enumeration#Value = {
    val classSymbol = mirror.classSymbol(clazz)
    val methodSymbol = factoryMethodSymbol(classSymbol.toType)
    val moduleSymbol = classSymbol.companionSymbol.asModule
    reflect(moduleSymbol, methodSymbol)(name).asInstanceOf[Enumeration#Value]
  }

  private def factoryMethodSymbol(enumType: Type): MethodSymbol = {
    enumType.member(newTermName("withName")).asMethod
  }

  private def reflect(module: ModuleSymbol, method: MethodSymbol)(args: Any*): Any = {
    val moduleMirror = mirror.reflectModule(module)
    val instanceMirror = mirror.reflect(moduleMirror.instance)
    instanceMirror.reflectMethod(method)(args:_*)
  }

}
```

Here is a client code example using TypeTag:

```scala
val level = EnumReflector.withName[FunninessLevel.Value]("LOL")
```

And another example with a class instance:

```scala
val level = EnumReflector.withName(FunninessLevel.getClass, "ROFL")
```

I'm quite happy with the TypeTag based implementation. For the class based implementation I would prefer to use `classOf[FunninessLevel.Value]`, but it seems it is impossible. There is no specific value class created for different enumeration types. So `classOf[FunninessLevel.Value]` returns a reference to the base `Enumeration#Value` class and there is no way to get a reference to the actual enumeration object.

Do you have any suggestion on how to improve this code?
