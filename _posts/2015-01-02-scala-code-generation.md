---
layout: post
title: 3 approaches to Scala code generation
---

There are two general ways to generate code: string templates and abstract syntax tree building. In Scala world
we have a couple of options to build an abstract syntax tree. So, in this post I'm going to compare three different approaches:

1. Generating code using string templates ([Twirl](https://github.com/playframework/twirl)).
2. Generating code from an abstract syntax tree ([treehugger](http://eed3si9n.com/treehugger/)).
3. Building an abstract syntax tree and sending it directly to the compiler ([Scala macros](http://scalamacros.org/)).

To do an apple-to-apple comparison I created a sample project implementing all three approaches. The project contains a
primitive data type schema and three different code generators producing classes for the given data schemas. You can find
the full source code in [the GitHub repo](https://github.com/dmitriy-yefremov/scala-code-gen).

Here is the data schema definition. It lets you to define a type, give it a name and provide the list of fields.

```scala
case class TypeSchema(name: TypeName, comment: String, fields: Seq[Field])

case class TypeName(fullName: String) {

  def packageName: String

  def shortName: String

}

case class Field(name: String, valueType: TypeName)
```

Below is a sample schema definition.

```json
{
  "name": {
    "fullName": "net.yefremov.sample.codegen.Foo"
  },
  "comment": "Test schema to play with the generators",
  "fields": [
    {
      "name": "bar",
      "valueType": {
        "fullName": "String"
      }
    },
    {
      "name": "baz",
      "valueType": {
        "fullName": "Int"
      }
    }
  ]
}
```

And here is a corresponding class to be generated. Please note an extra method returning the source schema. It is added
to have an example of methods generation.

```scala
/**
 * Test schema to play with the generators
 */
case class Foo(bar: String, baz: Int) = {

  def schema: String = """{"name":{"fullName":"net.yefremov.sample.codegen.Foo"},"comment":"Test schema to play with the generators","fields":[{"name":"bar","valueType":{"fullName":"String"}},{"name":"baz","valueType":{"fullName":"Int"}}]}"""

}
```

### 1. String templates

This is a very simple, lo-tech approach to generate Scala (or any other language) source code. It comes down to just a
bunch of `printf` statements. To make things a little cleaner we can use more advanced template engines. For this project
I chose [Twirl](https://github.com/playframework/twirl). It is Play Framework template engine. The main reason to
prefer Twirl over any other template engine is that it plays very well with Scala. The syntax is Scala-ish and you can
put pieces of Scala code directly into your templates (similar to Java blocks in JSP).

Below is the Twirl based code generator implementation.

```
@(schema: net.yefremov.sample.codegen.schema.TypeSchema)

@import _root_.net.yefremov.sample.codegen.template.TwirlGenerator
@import _root_.net.yefremov.sample.codegen.schema.TypeSchema

package @schema.name.packageName

/**
 * @schema.comment
 */
case class @(schema.name.shortName) (
    @for((field, index) <- schema.fields.zipWithIndex) {
        @field.name: @field.valueType.fullName @if(index < schema.fields.size - 1) { , }
    }
) {

  def schema: String = "@TypeSchema.toEscapedJson(schema)"

}
```

As you can see it is quite straightforward. You first need to handcraft the desired output and then replace configurable
pieces with template placeholders. These placeholders are replaced with the actual data when the template is evaluated.

The template looks just like the output and anyone can tell what is going on without any previous knowledge about Twirl.
Probably the only cumbersome part is the field parameters loop. It takes too much efforts to not include a comma after
the last parameter. It could be simplified with `fields.map(...).mkString(", "), but I decided to show more of Twirl's
native syntax.

#### Pros

* Does the job. There are successful projects using this approach.
* Very easy to get started, almost no learning curve.

#### Cons

* Can get quite messy if there is too much logic stuffed into one template.
* No knowledge about the syntax of the generated code (can easily produce code that does not compile).

### 2. Abstract syntax tree to source code

An [abstract syntax tree](http://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST) is a simplified syntactic representation
of the source code. An AST is the output of the syntax analysis phases of a compiler and the input to the semantic analysis
phase.

An AST is a much more structured matter then the source code. That may simplify the task of code generation. Instead of
generating actual source code we can generate its AST. And then either convert it back into the source code or proceed
with compilation.

To generate source from an AST I used a library called [treehugger](http://eed3si9n.com/treehugger/). It is a fork of
scalac code with some extensions.

Below is a treehugger based code generator implementation.

```scala
class TreehuggerGenerator {

  def generate(schema: TypeSchema): String = {
    // register new type
    val classSymbol = RootClass.newClass(schema.name.shortName)

    // generate list of constructor parameters
    val params = schema.fields.map { field =>
      val fieldName = field.name
      val fieldType = toType(field.valueType)
      PARAM(fieldName, fieldType): ValDef
    }

    // generate class definition
    val tree = BLOCK(
      CASECLASSDEF(classSymbol).withParams(params).tree.withDoc(schema.comment) :=  BLOCK(
        DEF("schema", StringClass) := LIT(TypeSchema.toJson(schema))
      )
      ).inPackage(schema.name.packageName)

    // pretty print the tree
    treeToString(tree)
  }

  private def toType(fieldType: TypeName): Type = {
    fieldType.fullName match {
      case "String" => StringClass
      case "Int" => IntClass
      case "Boolean" => BooleanClass
    }
  }

}
```

The code is somewhat complicated. Thanks to the great documentation on the project's [web site](http://eed3si9n.com/treehugger/)
it didn't take me too long to code this example. But after implementing a working generator I'm still not sure I fully
understand the code I wrote. Every individual part of the code looks simple and reasonable. But it's hard to understand
bigger pieces of code using treehugger.

Another thing that made me struggle with is IntelliJ not being able to compile the sample code and giving me different errors.
This is definitely an IntelliJ's issue, as SBT compiled everything just fine. But development experience becomes much
less pleasant without all the aids of a modern IDE.

#### Pros

* The library takes care of Scala syntax (e.g. you don't need to explicitly escape strings).
* The generated code is supposed to compile (you can't generate syntactically invalid code).
* The code is written in Scala  (that is beneficial for complex projects with a lot of logic, you can easily express any
 kind of conditions, extract reusable blocks, compose things and so on).

#### Cons

* Difficult initial learning process.
* The code is quite complicated.
* Not much support from the IDE.

### 3. Abstract syntax tree to byte code

As mentioned earlier an AST can be passed directly to the next compilation phase instead of generating source code out
of it. Strictly speaking this is not a Scala code generation method. There is no Scala source code produced. But in many
cases the source code itself is not important, the compiled output is what really needed. In these cases
[Scala macros](http://scalamacros.org/) can be used.

Scala macros are functions that are called by the compiler during the compilation process. They have access to the AST
and can manipulate it (including appending to it). There used to be two kinds of macros: [def](http://docs.scala-lang.org/overviews/macros/overview.html)
and [type](http://docs.scala-lang.org/overviews/macros/typemacros.html). The first one is used to generate functions, the
second one - new types. As of August 2013 type macros are [deprecated](http://scalamacros.org/news/2013/08/05/macro-paradise-2.0.0-snapshot.html).
The closest alternative that could be used for the purpose of this comparison is [macro annotations](http://docs.scala-lang.org/overviews/macros/annotations.html).

Below is a macro annotations based code generator implementation.

```scala
class FromSchema(schemaFile: String) extends StaticAnnotation {

  def macroTransform(annottees: Any*) = macro QuasiquotesGenerator.generate

}


object QuasiquotesGenerator {

  def generate(c: Context)(annottees: c.Expr[Any]*) = {

    import c.universe._

    // retrieve the schema path
    val schemaPath = c.prefix.tree match {
      case Apply(_, List(Literal(Constant(x)))) => x.toString
      case _ => c.abort(c.enclosingPosition, "schema file path is not specified")
    }

    // retrieve the annotate class name
    val className = annottees.map(_.tree) match {
      case List(q"class $name") => name
      case _ => c.abort(c.enclosingPosition, "the annotation can only be used with classes")
    }

    // load the schema from JSON
    val schema = TypeSchema.fromJson(schemaPath)

    // produce the list of constructor parameters (note the "val ..." syntax)
    val params = schema.fields.map { field =>
      val fieldName = newTermName(field.name)
      val fieldType = newTypeName(field.valueType.fullName)
      q"val $fieldName: $fieldType"
    }

    val json = TypeSchema.toJson(schema)

    // rewrite the class definition
    c.Expr(
      q"""
        case class $className(..$params) {

          def schema = ${json}

        }
      """
    )
  }

}
```

The implementation consists of two parts: the annotation that is used as a marker in the source code to trigger macro
expansion and the code generator itself. You could expect the generator implementation here to look just like the one
based on treehugger. But to make things even more ~~confusing~~ interesting I used a relatively new Scala feature called
[quasiquotes](http://docs.scala-lang.org/overviews/quasiquotes/intro.html). It lets you write snippets of code that
get automatically converted into the corresponding syntax trees. You can also use $-placeholders that are lifted by the
compiler. So the code looks very much like a string template but with all the goodies of type safety and syntax checking.

Here is an example of how to use the generator.

```scala
@FromSchema("sample/src/main/resources/Foo.json")
class Foo
```

#### Pros

* Official Scala tool to generate code.
* Quite readable AST code with quasiquotes.
* Takes care of Scala syntax (e.g. you don't need to explicitly escape strings).
* The generated code is supposed to compile (you can't generate syntactically invalid code).
* The code is written in Scala  (that is beneficial for complex projects with a lot of logic, you can easily express any
 kind of conditions, extract reusable blocks, compose things and so on).

#### Cons

* Quasiquotes expressions syntax does not always match normal Scala.
* Macro annotations require a stub class to be rewritten (you have to have something to annotate).
* Difficult initial learning process.
* The code is quite complicated.
* Not much support from the IDE.

### Summary

After playing with all three code generator implementations I'd say that string templates based approach is my favorite. For
not too complicated projects it should be enough. For a really complicated code generator with a lot of logic and code
reuse I'd look at AST generation. Both treehugger and Scala macros are great projects. Treehugger may be a cleaner
solution to generate completely new types. Scala macros is great at rewriting/augmenting existing code.

Please check out [the sample project](https://github.com/dmitriy-yefremov/scala-code-gen) and let me know what you think.