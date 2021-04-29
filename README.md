<a href="https://typelevel.org/cats/"><img src="https://typelevel.org/cats/img/cats-badge.svg" height="40px" align="right" alt="Cats friendly" /></a>
<br/>

Unleashing the Power of Http Apis: The Http4s Library 
=====================================================

Once we learned how to define Monoids, Semigroups, Applicative, Monads, and so on, it's time to understand how to use them to build a production-ready application. Nowadays, Many applications expose APIs over an HTTP channel. So, it's worth spending some time studying libraries implementing such use case.

If we learned the basics of functional programming using the Cats ecosystem, it's straightforward to choose the `http4s` library to implement HTTP endpoints. Let's see how to implement some aspects of HTTP APIs using it.

## Introduction

First thing first, we need an excellent example to work with. In this case, we need a domain model easy enough to focus on the exposition of simple APIs.

So, imagine we've just finished watching the Snyder Cut of the Justice League movie, and we are very excited about the film. We really want to tell the world how much we enjoyed the movie, and then we decide to build our personal "Rotten Tomatoes" application (ugh...).

As we are very experienced developers, we chose to start coding from the backend. Hence, we began by defining the resources we need in terms of the domain.

Indeed, we need a `Movie`, and a movie has a `Director`, many `Actors`, a `Synopsis`, and finally a `Review`. Without dwelling on the details, we can identify the following APIs among the others:

* Getting all movies of a director (i.e., Zack Snyder) made during a given year
* Getting the list of actors of a movie
* Adding a new director to the application


As we just finished the fantastic course on [Cats on Rock The JVM](https://rockthejvm.com/p/cats), we want to use a library built on the Cats ecosystem and exposing the above APIs. Fortunately, the [http4s](https://http4s.org/) library is what we are looking for. So, let's get a deep breath and start diving into the world of functional programming applied to HTTP APIs.

## Library Setup

We are going to use version 0.21.23 of the http4s library. Even if Scala 3 is almost upon us, this version of http4s uses Scala 2.13 as the target language.

The dependencies we must add in the `build.sbt` file are a lot:

```sbt
val Http4sVersion = "0.21.23"
val LogbackVersion = "1.2.3"
val MunitVersion = "0.7.20",
val MunitCatsEffectVersion = "0.13.0"
libraryDependencies ++= Seq(
  "org.http4s"      %% "http4s-blaze-server" % Http4sVersion,
  "org.http4s"      %% "http4s-circe"        % Http4sVersion,
  "org.http4s"      %% "http4s-dsl"          % Http4sVersion,
  "org.scalameta"   %% "munit"               % MunitVersion           % Test,
  "org.typelevel"   %% "munit-cats-effect-2" % MunitCatsEffectVersion % Test,
  "ch.qos.logback"  %  "logback-classic"     % LogbackVersion,
  "org.scalameta"   %% "svm-subs"            % "20.2.0"
)
```

The best way to understand each dependency is to explain it along the way. So, let's start the journey along with the `http4s` library.

## Http4s Basics

The `http4s` library grounds its function on the concepts of `Request` and `Response`. Indeed, using the library, we respond to a `Request` utilizing a function of type `Request => Response`. We call this function a route. In fact, a server is nothing more than a set of routes.

Very often, producing a `Response` from a `Request` means interacting with databases, external services, and so on, which may have some side effects. However, as diligent functional developers, we aim to maintain the referential transparency of our functions. Hence, the library surrounds the `Response` type into an effect `F` (more to come...), changing the route definition in `Request => F[Response]`.

Nevertheless, not all the `Request` will find a route to a `Response`. So, we need to take into consideration this fact, defining a route as a function of type `Request => F[Option[Response]]`. Using a monad transformer, we can simplify the route type in `Request => OptionT[F, Response]`.

Finally, using the types Cats provides us, we can rewrite the type `Request => OptionT[F, Response]` using the Kleisli monad transformer. Remembering that the type `Kleisli[F[_], A, B]` is just a wrapper around the function `A => F[B]`, our route definition becomes `Kleisli[OptionT[F, *], Request, Response]`. Easy, isn't it?

Fortunately, the `http4s` library defines a type alias for the Kleisli monad transformer that is easier to understand for human beings: `HttpRoutes[F]`.

Hence, defining the routes we need for our new shining website is sufficient to instantiate some ways using the above type. Awesome. So, it's time to start our journey. Let's implement the endpoint returning the list of movies associated with a particular director.

## Route Definition

We can image the route that returns the list of movies of a director as something similar to the following:

```
GET /movies?director=Zack%20Snyder
```

Every route corresponds to an instance of the `HttpRoutes[F]` type. Again, the `http4s` library helps us define such routes, providing us with a dedicated DSL, the `http4s-dsl`.

Through the DSL, we build an `HttpRoutes[F]` using pattern matching as a sequence of case statements. So, let's do it:

```scala
HttpRoutes.of[F] {
  case GET -> Root / "movies" :? DirectoryQueryParamMatcher(director) +& YearQueryParamMatcher (year) => ???
}
```

As we can see, the DSL is very straightforward. We surround the definition of all the routes in the `HttpRoutes.of` constructor. As we said, we parametrize the routes' definition with an effect `F`, as we probably have to retrieve information from some external resource.

Then, each `case` statement represents a specific route, and it matches a `Request` object. Hence, the DSL provides many deconstructors for a `Request` object, and everyone is associated with a proper `unapply` method.

First, the deconstructor that we can use to extract the HTTP method of a `Request`s is called `->` and decomposes them as a couple containing a `Method` (i.e. `GET`,`POST`, `PUT`, and so on), and a `Path`:

```scala
// From the Http4s DSL
object -> {
  def unapply[F[_]](req: Request[F]): Some[(Method, Path)] =
    Some((req.method, Path(req.pathInfo)))
}
```

The definition of the route continues extracting the rest of the information of the provided URI. If the path is absolute, we use the `Root` object, which simply consumes the leading `'/'` character at the beginning of the path.

Each part of the remaining `Path` is read using a specific extractor. We use the `/` extractor to pattern match each piece of the path. In this case, we match pure a `String`, that is `movie`.  However, we will see in a minute that it is possible to pattern match also directly in a variable.

### Handling Query Parameters

The route we are matching contains a query parameter. We can introduce the match of query parameters using the `:?` extractor. Even though there are many ways of extracting query params, the library sponsors the use of _matchers_. In detail, extending the `abstract class QueryParamDecoderMatcher`, we can pull directly into a variable given parameters:

```scala
object DirectorQueryParamMatcher extends QueryParamDecoderMatcher[String]("director")
```

As we can see, the `QueryParamDecoderMatcher` requires the name of the parameter and its type. Then, the library provides the parameter's value directly in the specified variable, which in our example is called `director`.

If we have to handle more than one query parameter, we can use the `+&` extractor, as we did in our example.

It's possible to manage also optional query parameters. In this case, we have to change the base class of our matcher, using the `OptionalQueryParamDecoderMatcher`:

```scala
object YearQueryParamMatcher extends OptionalQueryParamDecoderMatcher[Year]("year")
```

Considering that the `OptionalQueryParamDecoderMatcher` class matches against optional parameters, it's straightforward that the `year` variable will have the type `Option[Year]`.

Moreover, every matcher uses an instance of the class `QueryParamDecoder[T]` to decode its query parameter. The DSL provides the decoders for basic types, such as `String`, numbers, etc. However, we want to interpret our `"year"` query parameters directly as an instance of the `java.time.Year` class.

To do so, starting from a `QueryParamDecoder[Int]`, we can map the decoded result in everything we need, i.e., an instance of the `java.time.Year` class:

```scala
implicit val yearQueryParamDecoder: QueryParamDecoder[Year] =
    QueryParamDecoder[Int].map(Year.of)
```

As the matcher types access decoders using the type classes pattern, our custom decoder must be in the scope of the matcher as an `implicit` value. Indeed, decoders define the companion object `QueryParamDecoder` that lets the matcher types summoning the proper instance of the type class:

```scala
object QueryParamDecoder {
  def apply[T](implicit ev: QueryParamDecoder[T]): QueryParamDecoder[T] = ev
}
```

The object `QueryParamDecoder` contains many other practical methods to create custom decoders.

### Matching Path Parameters

Another everyday use case is a route that contains some path parameters. Indeed, the API of our backend isn't an exception, exposing a route that retrieves the list of actors of a movie:

```
GET /movies/aa4f0f9c-c703-4f21-8c05-6a0c8f2052f0/actors
```

Using the above route, we refer to a particular movie using its identifier, which we represent as a UUID. Again, the `http4s` library defines a set of extractors that help capture the needed information. For a UUID, we can use the object `UUIDVar`:

```scala
case GET -> Root / "movies" / UUIDVar(movieId) / "actors" => ???
```

Hence, the variable `movieId` has the type `java.util.UUID`. Equally, the library defines extractors also other primitive types, such as `IntVar`, `LongVar`. Moreover, the default extractor binds to a variable of type `String`.

However, if we need, we can develop our custom path parameter extractor, providing an object that implements the method `def unapply(str: String): Option[T]`. For example, if we want to extract the first and the last name of a director directly from the path, we can develop our custom extractor:

```scala
object DirectorVar {
  def unapply(str: String): Option[Director] = {
    if (str.nonEmpty && str.matches(".* .*")) {
      Try {
        val splitStr = str.split(' ')
        Director(splitStr(0), splitStr(1))
      }.toOption
    } else None
  }
}

HttpRoutes.of[F] {
  case GET -> Root / "directors" / DirectorVar(director) => ???
}
```

### Composing Routes

The routes defined using the http4s DSL are composable. So, it means that we can define them in different modules and then compose them in a single `HttpRoutes[F]` object. As developers, we know how vital is compositionality of modules to obtain code that is easily maintainable and evolvable.

The trick is that the type `HttpRoutes[F]`, being an instance of the `Klesli` type, is also a `Semigroup`. In detail, the suitable type class is `SemigroupK`, as we have a semigroup of an effect (remember the `F[_]` type constructor).

Hence, [the main feature of semigroups](https://blog.rockthejvm.com/semigroups-and-monoids-in-scala/) is the definition of the `combine` function which, given two elements of the semigroup, returns a new element also belonging to the semigroup. For the `SemigroupK` type class, the function is called`combineK` or `<+>`.

```scala
def allRoutes[F[_]: Sync]: HttpRoutes[F] = {
  import cats.syntax.semigroupk._
  movieRoutes <+> directorRoutes
}
```

To access the `<+>`, we need the proper imports from Cats, which is at least `cats.syntax.semigroupk._`. Fortunately, we import the `cats.SemigroupK` type class for free using the `Klesli` type.

Last but not least, an incoming request might not match with any of the available routes. Usually, such requests should end up with 404 HTTP responses. As always, the http4s library gives us an easy way to code such behavior, using the `orNotFound` method from the package `org.http4s.implicits`:

```scala
def allRoutesComplete[F[_]: Sync]: HttpApp[F] = {
  allRoutes.orNotFound
}
```

As we can see, the method returns an instance of the type `HttpApp[F]`, which is a type alias for `Kleisli[F, Request[G], Response[G]]`. Hence, for what we said at the beginning of this article, this `Kleisli` is nothing more than a wrapper around the function `Request[G] => F[Response[G]]`. So, the difference with the `HttpRoutes[F]` type is that we removed the `OptionT` on the response.

## Generate Responses

As we said, generating responses to incoming requests is an operation that could involve some side effects. In fact, http4s handle responses with the type `F[Response]`. However, explicitly creating a response inside an effect `F` can be tedious.

So, the `http4s-dsl` module provides us some shortcuts to create responses associated with HTTP status codes. For example, the `Ok()` function creates a response with a 200 HTTP status:

```scala
HttpRoutes.of[F] {
  case GET -> Root / "directors" / DirectorVar(director) =>
    Ok(Director("Zack", "Snyder").asJson)
}
```

We will look at the meaning of the `asJson` method in the next section. However, it is crucial to understand the structure of the returned responses, which is:

```
F(
  Response(
    status=200, 
    headers=Headers(Content-Type: application/json, Content-Length: 40)
  )
)
```

As we may imagine, inside the response, there is a place for the status, the headers, etc. However, we don't find the response body because the effect was not yet evaluated.

In addition, inside the `org.http4s.Status` companion object, we can find the functions to build a response with every other HTTP status listed in the IANA specification. Suppose that we want to implement some type of validation on a query parameter.

For example, we want to return a _Bad Request_ HTTP Status if the query parameter `year` doesn't represent a positive number. First, we need to change the type of matcher we need to use, introducing the validation:

```scala
implicit val yearQueryParamDecoder: QueryParamDecoder[Year] =
  QueryParamDecoder[Int].emap { y =>
    Try(Year.of(y))
      .toEither
      .leftMap { tr =>
        ParseFailure(tr.getMessage, tr.getMessage)
      }
  }
object YearQueryParamMatcher extends OptionalValidatingQueryParamDecoderMatcher[Year]("year")
```

Then, we can introduce the code handling the failure with a `BadRequest` HTTP status:

```scala
case GET -> Root / "movies" :? DirectorQueryParamMatcher(director) +& YearQueryParamMatcher(maybeYear) =>
  maybeYear match {
    case Some(y) =>
      y.fold(
        _ => BadRequest("The given year is not valid"),
        year =>
          // Proceeding with the business logic
      )
    case None => NotFound(s"There are no movies for director $director")
  }
```

### Headers and Cookies

In the previous section, we saw that http4s inserts in the response some preconfigured headers. However, it's possible to add many other headers during the response creation:

```scala
Ok(Director("Zack", "Snyder").asJson, Header("My-Custom-Header", "value"))
```

In addition, the http4s library provides a lot of standard HTTP headers in the package `org.http4s.headers`:

```scala
import org.http4s.headers.`Content-Encoding`
Ok(`Content-Encoding`(ContentCoding.gzip))
```

Cookies are nothing more than a more complex form of HTTP Header, called `Set-Cookie`. Again, the http4s library gives us an easy way to deal with cookies in responses. However, unlike the headers, we set the cookies after the response creation:

```scala
Ok().map(_.addCookie(ResponseCookie("My-Cookie", "value")))
```

Obviously, we can instantiate a new `ResponseCookie`, giving the constructor all the additional information needed by a cookie, such as expiration, the secure flag, httpOnly, flag, etc.

## Encoding and Decoding Http Body

### Access to the Request Body

So, we should know everything we need to match routes. Now, it's time to understand how to decode and encode structured information associated with the body of a `Request` or of a `Response`.

As we initially said, we want to let our application expose an API to add a new `Director` in the system. Such an API will look something similar to the following:

```
POST /directors
{
  "firstName": "Zack",
  "lastName": "Snyder"
}
```

First thing first, to access its body, we need to access directly to the `Request[F]` through pattern matching:

```scala
case req @ POST -> Root / "directors" => ???
```

Then, the `Request[F]` object has a `body` attribute of type `EntityBody[F]`, which is a type alias for `Stream[F, Byte]`. As the HTTP protocol defines, the body of an HTTP request is a stream of bytes. The http4s library uses the [`fs2.io`](https://fs2.io/#/) library as stream implementation. Indeed, this library also uses the Typelevel stack to implement its functional vision of streams.

### Decoding the Request Body Using Circe

In detail, every `Request[F]` extends the `Media[F]` trait. This trait exposes many practical methods dealing with the body and some header of a request, and the most interesting is the following function:

```scala
final def as[A](implicit F: MonadThrow[F], decoder: EntityDecoder[F, A]): F[A] =
```

So, the `as` function lets us decode a request body as a type `A`. To do so, the function uses an `EntityDecoder[F, A]`, which we must provide in the context as an implicit object.

An `EntityDecoder` (and its counterpart `EntityEncoder`) allows us to deal with the streaming nature of the data in an HTTP body. In addition, decoders and encoders relate directly to the `Content-Type` declared as an HTTP Header in the request/response. Indeed, we need a different kinds of decoders and encoders for every type of content.

The http4s library ships with decoders and encoders for a limited type of contents, such as `String`, `File`, `InputStream`, and manages more complex contents using plugin libraries.

In our example, the request contains a new director in JSON format. To deal with JSON body, the most frequent choice is to use the Circe plugin:

```sbt
"org.http4s" %% "http4s-circe" % 0.21.23,
```

The primary type provided by the circe library to manipulate JSON information is the `io.circeJson` type. Hence, the `http4s-circe` module defines the type `EntityDecoder[Json]` and `EntityEncoder[Json]`, which is all we need to translate a request and a response body into an instance of `JSon`.

However, `Json` type translate directly into JSON literals, such as `json"""{"firstName": "Zack", "lastName": "Snyder"}"""`, and we don't really want to deal with them. Instead, we usually use case classes to represent information carried by a request or response body. In our example, the JSON representation of a director will translate in the following case class:

```scala
case class Director(firstName: String, lastName: String)
```

First, we decode the incoming request automatically into an instance of a `Json` object using the `EntityDecoder[Json]` type. However, we need to do a step beyond to obtain an object of type `Director`. In detail, Circe needs an instance of the type `io.circe.Decoder[Director]` to decode a `Json` object into a `Director` object.

Hence, we can provide the instance of the `io.circe.Decoder[Director]` simply adding the import to `io.circe.generic.auto._`, which lets Circe automatically derive for us encoders and decoders. The derivation uses field names of case classes as key names in JSON objects and vice-versa. As the last step, we need to connect the type `Decoder[Director]` to the type `EntityDecoder[Json]`, and this is precisely the work that the module `http4s-circe` does for us through the function `jsonOf`:

```scala
implicit val directorDecoder: EntityDecoder[F, Director] = jsonOf[F, Director]
```

Summing up, the final form of the API looks like the following:

```scala
def directorRoutes[F[_]: Sync]: HttpRoutes[F] = {
  val dsl = new Http4sDsl[F]{}
  import dsl._
  implicit val directorDecoder: EntityDecoder[F, Director] = jsonOf[F, Director]
  HttpRoutes.of[F] {
    case req @ POST -> Root / "directors" =>
      for {
        director <- req.as[Director]
        // ...
      }
  }
```

### Encoding the Response Body Using Circe

The encoding process of a response body is very similar to the one described for decoding a request body. For example, we go forward with the API getting all the films directed by a specific director during a year:

```
GET /movies?director=Zack%20Snyder&year=2021
```

Hence, we need to model the response of such API, which will be a list of movies:

```json
[
  {
    "title": "Zack Snyder's Justice League",
    "year": 2021,
    "actors": [
      "Henry Cavill",
      "Gal Godot",
      "Ezra Miller",
      "Ben Affleck",
      "Ray Fisher",
      "Jason Momoa"
    ],
    "director": "Zack Snyder"
  }
]
```

Hence, we can model the above information using a case class:

```scala
case class Movie(title: String, year: Int, actors: List[String], director: String)
```

Easy peasy lemon squeezy. As for decoders, we need first to transform our class `Movie` into an instance of the `Json` type, using an instance of `io.circe.Encoder[Movie]`. Again, the `io.circe.generic.auto._` import allows us to automagically define such an encoder. 

Instead, we can perform the actual conversion from the `Movie` type to `Json` using the extension method `asJson`, defined on all the types that have an instance of the `Encoder` type class that we have just created for our `Movie` type:

```scala
package object syntax {
  implicit final class EncoderOps[A](private val value: A) extends AnyVal {
    final def asJson(implicit encoder: Encoder[A]): Json = encoder(value)
  }
}
```

As we can see, the above method comes into the scope importing the `io.circe.syntax._` package. As 
we previously said for decoders, the library http4s-circe provides the type `EntityEncoder[Json]`,
with the import `org.http4s.circe._`. 

Now, we can complete our API definition, responding with the needed information:

```scala
val snjl: Movie = Movie(
  "Zack Snyder's Justice League",
  2021,
  List("Henry Cavill", "Gal Godot", "Ezra Miller", "Ben Affleck", "Ray Fisher", "Jason Momoa"),
  "Zack Snyder"
)

def movieRoutes[F[_]: Sync]: HttpRoutes[F] = {
  val dsl = new Http4sDsl[F]{}
  import dsl._
  HttpRoutes.of[F] {
    case GET -> Root / "movies" :? DirectorQueryParamMatcher(director) +& YearQueryParamMatcher(year) =>
      Ok(List(snjl).asJson)
    // ...
  }
}
```

## Wiring All Together

At this point, we learnt how to define a route, how to read path and parameter variable, how to read
and write HTTP bodies, and how to deal with headers and cookies. Now, it's time to wire all together,
defining and starting the server serving all our routes.

The http4s library support many types of servers. We can look at the integration matrix directly in
the [official documentation](https://http4s.org/v0.21/integrations/). However, the native supported
server is [blaze](https://github.com/http4s/blaze). 

We can instantiate a new blaze server using the dedicated builder, `BlazeServerBuilder`, which takes
as inputs the `HttpRoutes` (or the `HttpApp`), and the host and port serving the given routes:

```scala
val movieApp = MovieApp.allRoutesComplete[IO]
BlazeServerBuilder[IO](global)
  .bindHttp(8080,"localhost")
  .withHttpApp(movieApp)
  // ...it's not finished yet
```

In blaze jargon, we said that we mount the routes at the given path. The default path is `"/"`, but
if we need we can easily change the path using a `Router`:

```scala
val apis = Router(
  "/api/v1" -> MovieApp.movieRoutes[IO],
  "/api/v2" -> MovieApp.directorRoutes[IO]
).orNotFound
BlazeServerBuilder[IO](global)
  .bindHttp(8080, "localhost")
  .withHttpApp(apis)
```

As we can see, the `Router` accepts a list of mappings from a `String` root to an `HttpRoutes` type.
Moreover, we can omit the call to the `bindHttp`. If so, blaze will serve the APIs using port `8080`
on the `localhost` address.

Moreover, the builder needs an instance of an `scala.concurrent.ExecutionContext` that uses to handle
incoming requests concurrently. Finally. we need to bind the builder to an effect, because the 
execution of the service can lead to side effects itself. In our example, we are binding the effect
to the `IO` monad from the library Cats Effect.

### Executing the Server

Once we configured the basic properties of a `BlazeServerBuilder`, we need to run it. Since chose to
use the `IO` monad as an effect, the easier way to run an application using the blaze server is to 
run the service inside an `IOApp`, provided by the cats-effect library, too. 

The `IOApp` is a utility type that allows us to run applications that use the `IO` effect. Moreover,
inside the `IOApp`, we the Cats Effect library provides us for free two implicit instances needed by
the builder to work: An instance of the `ContextShift[IO]` type, and of the `Timer[IO]` type.

When it's up, the server listens to incoming requests on a port. If the server stops, it must 
release the port and close any other resource it's using. This is exactly the definition of the 
[`Resource`](https://typelevel.org/cats-effect/docs/2.x/datatypes/resource) type from the Cats 
Effect library. Hence, we can think about `Resource`s as the `AutoCloseable` type in Java type.

So, we can use the builder to create a `Resource`:

```scala
object Main extends IOApp {
  def run(args: List[String]): IO[ExitCode] = {
    // Omissis...
    BlazeServerBuilder[IO](global)
      .bindHttp(8080, "localhost")
      .withHttpApp(apis)
      .resource
      .use(_ => IO.never)
      .as(ExitCode.Success)
  }
}
```

Once we have a `Resource` to handle, we can `use` its content. In this example, we say that whatever
the resource is, we want to produce an effect that never terminates. In this way, the server can 
accept new requests over and over. Finally, the last statement maps the result of the `IO` into the
given value, `ExitCode.Success`.

### Try It Out!

Once the server is up and running, we can try our freshly new APIs with any HTTP client, such as 
`cURL` or something similar. Hence, the call `curl 'http://localhost:8080/api/v1/movies?director=Zack%20Snyder&year=2020' | json_pp`
will produce the following response, as expected:

```json
[
  {
    "id" : "6bcbca1e-efd3-411d-9f7c-14b872444fce",
    "director" : "Zack Snyder",
    "title" : "Zack Snyder's Justice League",
    "year" : 2021,
    "actors" : [
      "Henry Cavill",
      "Gal Godot",
      "Ezra Miller",
      "Ben Affleck",
      "Ray Fisher",
      "Jason Momoa"
    ]
  }
]
```

## Addendum: Why We Need to Abstract Over the Effect

One thing that we should have kept an eye on is the use of the abstract `F[_]` type constructor in 
the routes' definition, instead of the use of a concrete effect type, such as `IO`.

### A (Very) Fast Introduction to The Effect Pattern

First, why do we need to use an effect in routes definition? Well, we must remember that we are 
using a library that is intended to embrace the purely functional programming paradigm. Hence, our
code must adhere to the [substitution model](http://www.cs.cornell.edu/courses/cs312/2008sp/lectures/lec05.html),
and we know for sure that every code producing any possible side effect is not compliant to such a
model. Besides, we know that reading from a database, or a file-system, and in general interacting
with the outside world, potentially rises exceptions or produces different results every time it is
executed.

So, the functional programming paradigm overcomes this problem using the Effect Pattern. Instead of
executing directly the code that may produce any side effect, we can enclose it in a special context
that describes the possible side effect that code may produce, but doesn't effectively execute it, 
and the type the effect will produce:

```scala
val hw: IO[Unit] = IO(println("Hello world!"))
```

For example, the `IO[A]` effect from Cats Effect represents any kind of side effect, and the value
it might produce. The above code doesn't print anything to the console, but it defers the execution 
until it's possible. Besides, it says that it will produce a `Unit` value, once executed. The trick
is in the definition of the `delay` method, which is called inside the smart constructor:

```scala
def delay[A](a: => A): IO[A]
```

So, when will the application print the "Hello world" message to the standard output? The answer is
easy: the code will be executed once the method `unsafeRunSync` would be called:

```scala
hw.unsafeRunSync
// Prints "Hello world!" to the standard output
```

If we want that the substitution model applies to the whole part of the application, the only 
reasonable place to call the `unsafeRunSync` is in the `main` method, also called 
_the end of the world_. Indeed, it's exactly what the `IOApp.run` method does for us.

### Why Binding Route Definition to a Concrete Effect Is Awful

So, why don't we use directly a concrete effect, such as the `IO` effect, in the routes' 
definitions? Basically, there are two main reasons.

First, if we abstract over the effect, we can easily control what kind of execution model we want to
apply to our server. Will the server perform operation concurrently, or sequentially? In fact, there
are different kinds of effect in the Cats Effect that model the above execution models, the `Sync` 
type class for synchronous, blocking programming, and the `Async` type class for asynchronous, concurrent 
programming. Notice that the `Async` type extends the `Sync` type.

If we need to ensure at least some properties on the execution model applied to the effect, we can
use the context bound syntax. For example, we defined our routes using an abstract effect `F` that
has at least an associated `Sync` type class:

```scala
def movieRoutes[F[_] : Sync]: HttpRoutes[F] = ???
```

Moreover, the use of a type constructor instead of a concrete effect make the whole architecture 
easier to test. Binding to a concrete effect forces us to use it also in the tests, making test more
difficult to write.

For example, in test, we can bind `F[_]` to the `Reader` monad, instead, or any other type 
constructor eventually satisfying the constraints given within the context bound.

## Conclusion

Summing up, in this article we introduced the Http4s ecosystem that helps us in building servers 
serving API over HTTP. The library fully embraces the functional programming paradigm, using Cats 
and Cats Effects as building blocks. 

So, we saw how easy is the definition of routes, handling path and query parameters, and how to 
encoding and decoding JSON bodies. Finally, we wired all the things up, and we showed how to define 
and start a server instance.

Even though there many other features that this introductive tutorial doesn't address, such as
[Middlewares](https://http4s.org/v0.21/middleware/), [Streaming](https://http4s.org/v0.21/streaming/),
[Authentication](https://http4s.org/v0.21/auth/), we could now build many non-trivial use cases
concerning the development of HTTP API.

Hence, there is only one thing left: Try it on your own and enjoy pure functional programming 
HTTP APIs!
