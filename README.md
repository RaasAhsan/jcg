jcg
===

`jcg` is a library that automatically generates `JsWriters` and `JsReaders` from `json` for your models.

Motivation
----------

This tool is meant to be used in conjunction with [json](https://github.com/plasmaconduit/json), which exposes two important traits, `JsWriter` and `JsReader`,
which can be used to implement a serializer and deserializer to and from JSON. Here is an example for a trivial model.

```scala
import com.plasmaconduit.json._
import com.plasmaconduit.validation._

final case class PhoneNumber(value: String)

object PhoneNumber {
  implicit object PhoneNumberJsWriter extends JsWriter[PhoneNumber] {
    override def write(p: PhoneNumber): JsValue = JsObject(
      "value" -> value
    )
  }

  implicit object PhoneNumberJsReader extends JsReader[PhoneNumber] {
    type JsReaderFailure = PhoneNumberJsReaderError

    sealed trait PhoneNumberJsReaderError
    case object PhoneNumberNotJsonObject extends PhoneNumberJsReaderError
    case object PhoneNumberNumberInvalidError extends PhoneNumberJsReaderError
    case object PhoneNumberNumberMissingError extends PhoneNumberJsReaderError

    val numberExtractor = JsonObjectValueExtractor[String, PhoneNumberJsReaderError](key = "number", missing = PhoneNumberNumberMissingError, invalid = _ => PhoneNumberNumberInvalidError)

    override def read(value: JsValue): Validation[PhoneNumberJsReaderError, org.company.app.models.phone.PhoneNumber] = {
      value match {
        case JsObject(map) => {
          for (number <- numberExtractor(map)) yield org.company.app.models.phone.PhoneNumber(number = number)
        }
        case _ => Failure(PhoneNumberNotJsonObject)
      }
    }
  }
}
```

It is obvious that this is quite a bit of boilerplate to get a `JsWriter` or a `JsReader` working, but even more importantly, adding additional fields is time consuming.
But, as your codebase grows with more and more models, writing implementations for these traits becomes fairly repetitive and time consuming. All those error objects
are fairly handy, but are quite irritating to write for all of the fields.

Instead of falling back to reflection on-the-fly to serialize/deserialize to and from JSON (thereby giving up type safety), we realized that all the components of a `JsWriter`
or a `JsReader` can be directly inferred from the names and types of the parameters of a model. Therefore, given a Scala AST of a model, we can automatically generate code for them
to do the job.

`jcg` scans your Scala codebase for models and generates code (`JsWriters` and `JsReaders`) and dumps it to a directory of your choice.


Install
-------

Add the following to your `Build.scala`
```
resolvers += "Plasma Conduit Repository" at "http://dl.bintray.com/plasmaconduit/releases",
libraryDependencies += "com.plasmaconduit" %% "jcg-traits" % "0.3.0"
```

Usage
-----

First, specify the models for which you want to generate a `JsReader` or `JsWriter` by extending the `GenReader` or `GenWriter` trait and choose
a reader and writer representation (more on this later).

```scala
import com.plasmaconduit.json.codegen.traits._

final case class User(id: Int, username: String, password: String, email: String, items: List[Item]) extends GenWriter

final case class Item(id: Int, name: String) extends GenReader with GenWriter

final case class PhoneNumber(value: String) extends GenReader with GenWriter
```

NOTE: Ideally, all your models specified for generation should form a closed set (excluding the basic Js types defined in `json`).
If you define your own `JsReader` or `JsWriter` for a model manually (that is used in another model), the imports have to be resolved manually.
This may or may not be an issue in the future depending on how much code analysis we want `jcg` to do.

Run the tool in your root directory to generate all the code you don't have to write anymore!

```
java -cp "jcg-0.1.0.jar:." com.plasmaconduit.json.codegen.JsGenerator /path/to/jcg/examples src/main/scala org.company.app json
```

This will recursively find models in the package `org.company.app` create a package `json` with the generated code. Now, you can use
the readers and writers wherever you'd like!

```scala
import json.writers.GenJsWriters._
import json.readers.GenJsReaders._

val user: User = ...

// implicit for JsWriter[User] is automatically resolved
val js = JsObject(
  "user" -> user
)

val inputItem: JsValue = ...

// implicit for JsReader[Item] is automatically resolved
val item: Validation[ItemJsReaderError, Item] = inputItem.as[Item]
```

The implicits will automatically be resolved in the generated code files.

Representations
---------------

### GenObjectRep ###

The default representation `jcg` uses for a reader or a writer is `GenObjectRep()`, which treats the model solely as an JSON object. This is
reasonable to have when we have a model with multiple fields, like `User`. You can specify fields to ignore by passing in a List of field names to ignore.

```scala
final case class User(id: Int, username: String, password: String, email: String, items: List[Item]) extends GenWriter {
  val writerRep = GenWriterRep(List("password"))
}
```

### GenParameterRep ###

In some cases, we have a single field model that only acts as a wrapper model, like `PhoneNumber`, which wraps a `String`. With `GenParameterRep`, we can choose
to delegate the model representation to the parameter, so the model itself is not represented as a JSON object, but as a JSON string, for example.

```scala
final case class PhoneNumber(value: String) extends GenReader with GenWriter {
  val readerRep = GenParameterRep
  val writerRep = GenParameterRep
}
```

A `PhoneNumber` will be correctly serialized and deserialized to and from a `JsString`.

IMPORTANT: `GenParameterRep` should only be used on models with single fields. Otherwise, `GenObjectRep` should always be used.

TODO
----
* Implement reader/writer generators for sealed traits.
* Insert comments/warnings about stuff.
