---
layout: post
title: Writing a Scala and Akka-HTTP based client for REST API (Part I)
comments: true
---

And so I decided to write another HTTP Client using Scala. Sounds easy? Sure, if you know where to start. As with so many things in the Scala world, picking your libraries can be the hardest part. In this post I'll try to make the experience of writing a HTTP client library in Scala a little less painful so you don't have to start from scratch. Ok, enough introduction, let's start!

In my particular use case I wanted to develop a type-safe HTTP client library for consuming Oanda REST API v20. You can have a look at the [API spec](http://developer.oanda.com/rest-live-v20/introduction) if you want to learn more about specific protocol but here we want to focus on getting the right tools to get the job done.

First, we need an HTTP client library that we can use with Scala. There is no shortage of good options for Java, like the de-facto standard [Apache HTTP Client](http://hc.apache.org/httpcomponents-client-5.0.x/index.html) or its async [sibling](https://hc.apache.org/httpcomponents-asyncclient-4.1.x/index.html), [Async HTTP Client](https://github.com/AsyncHttpClient/async-http-client), [HTTP Client from Google](https://github.com/google/google-http-java-client) and many other solid choices. And certainly, Scala - being a JVM language - can make use of any of those. But as a Scala developer, one is always trying to find a good 'idiomatic' Scala library first before resorting to other options. However, as I've learned from my almost 4 years experience coding in Scala, this is not always easy. Unfortunately, my conclusion after spending quite some time researching for alternatives in this space is that most libraries are immature, or are not maintained any more, or both.

I used [spray](http://spray.io) in the past and was more or less happy with it. However, it is not maintained now and users are directed to migrate to [Akka HTTP](http://doc.akka.io/docs/akka-http/current/scala.html) which is an official rewrite of spray. Initially, I was a bit hesitant to bring in Akka HTTP into my project. After all, I didn't really need any of the server-side functionality, which seems to be the main focus of Akka HTTP, with client capabilities being treated more or less as a stepchild. After a quick 'google' I found a few alternatives and decided to first give [dispatch](https://github.com/dispatch/reboot) a try. It seemed like a solid choice, was being praised as one of the most mature HTTP client libraries written entirely in Scala, is both easy to get started with and has a beautiful idiomatic Scala API. So far so good! It did indeed work fine for basic GET/PUT/PATCH/POST requests, but as soon as I had to do some more or less advanced stuff with it like processing responses with chunked transfer encoding I hit a road block. It's not that I didn't try to make it work. But it turned out to be overly complicated and I didn't want to spend a lot of time 'hacking in' basic HTTP functionality when I had a real task at hand. This is, unfortunately, quite often a common theme among Scala libraries that either don't come from Lightbend itself or have a quality label of [Typelevel](http://typelevel.org) attached to it. 'Hello World' use cases are working fine out of the box, however as soon as you start to require more advanced functionality be prepared to roll up your sleeves and spend countless hours trying to fix basic stuff using limited resources available on StackOverflow and others.

Long story short, eventually I had to abandon dispatch and went back to using Akka HTTP, which was at least mature enough, but more importantly, has the largest community behind it and is being actively maintained. My only piece of advice here for anyone who is starting a new project that requires HTTP client capabilities, don't waste your time experimenting with different Scala libraries and just pick the one that works, for me it is undoubtfully Akka HTTP.

We don't want to bother users of our client library with low-level details of the HTTP communication, right? This means we need to be able to abstract away the JSON wire format into a more robust Scala [ADT](https://en.wikipedia.org/wiki/Algebraic_data_type) model based on `case class`es and `case object`s. So here is our next challenge: picking a JSON library. Again, plenty of alternatives out there and I have to say I haven't tried most of them. I used [spray-json](https://github.com/spray/spray-json) in the past, however the amount of boilerplate required to make it work was not ideal. If you want to go with spray-json, the good news is that it comes with out-of-the-box support by Akka HTTP. However, in order to reduce the amount of boilerplate required for working with JSON I decided to look for alternatives. That's when I discovered [circe](https://github.com/circe/circe), which seemed to have a very active community behind it, has numerous Q&As on StackOverflow and even comes with its own gitter chat channel. Circe is boasting fully automatic derivation for encoding/decoding your domain model (`case class`es and `case object`s) into/from JSON. In the simplest scenario all you need to do is provide one import where you want to use en-/decoding:

```scala
import io.circe.generic.auto._
```

The rest is magically taken care of by Scala macros generating implicit instances of `Encoder` and `Decoder` objects. Looks good on paper, BUT: macros are not the most stable part of the Scala compiler (as we'll see in a second) and it will slow down your compilation by A LOT (at one point my [project](https://github.com/msilb/scalanda-v20) took 12 minutes to compile). Now, one particular issue I encountered when using fully automatic derivation is that circe is using [shapeless](https://github.com/milessabin/shapeless) under the hood to derive `Encoder` and `Decoder` instances, which in turn relies on Scala macros. This foundation seems to be not very rock solid though. Things like [this](https://issues.scala-lang.org/browse/SI-7046), [this](https://issues.scala-lang.org/browse/SI-7567) and [this](https://github.com/lloydmeta/enumeratum/issues/90) do not fill one with confidence. In fact, I encountered the same error as the author of the last linked github issue: `knownDirectSubclasses observed before subclass registered` and it just seemed completely random depending on in which file my case classes were defined and similar things. Only switching to a more reliable semi-automatic derivation using `Codec` annotation finally stabilized the build. However, the build still takes about 3-4 minutes to complete. I am thinking about abandoning `shapeless` backed derivation altogether and using more manual ways to define my `Encoder`s and `Decoder`s in order to speed up the compilation. This however, will bring back the boilerplate. Tradeoffs, it's always about tradeoffs.

Ok, enough talk, let's see some code. First, let's start with the basic model design. We make a remote HTTP call to the API and can receive a `200` status code response with a JSON payload, or we get one of many error status codes `4xx`, again with optional JSON payload detailing the error message. Here is how we want to model it:

```scala
import io.circe.generic.semiauto.deriveDecoder
import io.circe.Decoder

sealed trait Response
object Response {
  ...
  private def decodeSuccessOrFailure[T, S <: T : Decoder, F <: T : Decoder]: Decoder[T] = Decoder.instance { c =>
    c.downField("errorMessage").as[String]
      .flatMap(_ => c.as[F])
      .left
      .flatMap(_ => c.as[S])
  }
  sealed trait ConfigureAccountResponse extends Response
  object ConfigureAccountResponse {
    case class ConfigureAccountSuccessResponse(clientConfigureTransaction: ClientConfigureTransaction,
                                               lastTransactionID: TransactionID) extends ConfigureAccountResponse
    case class ConfigureAccountFailureResponse(clientConfigureRejectTransaction: Option[ClientConfigureRejectTransaction],
                                               lastTransactionID: Option[TransactionID],
                                               errorCode: Option[String],
                                               errorMessage: String) extends ConfigureAccountResponse
    implicit val decodeConfigureAccountSuccessResponse: Decoder[ConfigureAccountSuccessResponse] = deriveDecoder
    implicit val decodeConfigureAccountFailureResponse: Decoder[ConfigureAccountFailureResponse] = deriveDecoder
    implicit val decodeConfigureAccountResponse: Decoder[ConfigureAccountResponse] =
      decodeSuccessOrFailure[ConfigureAccountResponse, ConfigureAccountSuccessResponse, ConfigureAccountFailureResponse]
  }
  ...
}
```

Now let's have a close look at the signature of the `decodeSuccessOrFailure` function. It takes 3 type parameters: `T` is our base trait for a particular type of API response, `S` is mapping a successful response, and `F` a failure, both must be subtypes of `T` as indicated by the `<:` symbol. `: Decoder` is simply a Scala syntactic sugar for a so-called _implicit evidence_ parameter `implicit d: Decoder[T]`. This is required to let the compiler know that we can only accept type parameter `T` for which we have an implicit `Decoder` instance defined somewhere in scope. Ok, let's move on. So, when we receive our JSON object from the remote call, the compiler doesn't know whether it's a success or failure and which case class should the response be decoded into. We need to help it a little. For instance, we might infer from the structure of the JSON that the response is an error response. In this case we know that all error responses _must_ contain the field `errorMessage`, hence we attempt to decode the appropriate subtype of `T` depending on whether `errorMessage` is present in our JSON object or not.

Once we sorted out decoding of our response we can turn our eyes on the HTTP request/response workflow. Akka HTTP provides some capabilities for [unmarshalling](http://doc.akka.io/docs/akka-http/current/scala/http/common/unmarshalling.html) HTTP response entities. It can unmarshall HTTP response entities into `String`, `Int`, `List` and other basic types and in the simplest case all that's required to fetch a String response from our HTTP endpoint is this line of code `Http().singleRequest(req).flatMap(r => Unmarshal(r.entity).to[String])`. But what about our custom case classes, what if we want to unmarshal HTTP response not into `String` but directly into our subtype of `Response` trait? Well, we need to put in some leg work to be able to do that. In order to make it work we need to connect our circe `Decoder`s defined in the previous step with the unmarshalling infrastructure of `Akka HTTP`. Fortunately, this can be achieved fairly quickly with minimal overhead. All we need to do is provide an implicit instance of `FromEntityUnmarshaller[T]` with `T` being our response type we're trying to decode. Here's how this can be done with a few lines of code:

```scala
import akka.http.scaladsl.model.MediaTypes.`application/json`
import akka.http.scaladsl.unmarshalling.{FromEntityUnmarshaller, Unmarshaller}
import akka.util.ByteString
import io.circe.{Decoder, jawn}

object AkkaHttpCirceSupport extends AkkaHttpCirceSupport
trait AkkaHttpCirceSupport {
  private val jsonStringUnmarshaller = Unmarshaller.byteStringUnmarshaller
    .forContentTypes(`application/json`)
    .mapWithCharset {
      case (ByteString.empty, _) => throw Unmarshaller.NoContentException
      case (data, charset) => data.decodeString(charset.nioCharset.name)
    }
  implicit def circeUnmarshaller[A](implicit decoder: Decoder[A]): FromEntityUnmarshaller[A] =
    jsonStringUnmarshaller.map(jawn.decode(_).fold(throw _, identity))
}
```

Here we build on existing Akka HTTP functionality provided by `Unmarshaller.byteStringUnmarshaller` and decoding done by JSON facade `jawn` used internally by `circe` to generate our custom Unmarshaller instance. Phew, nuff boilerplate already, you may ask? We're almost there.

We are now ready to make our HTTP call. We can use our implicitly provided `FromEntityUnmarshaller` in combination with the also implicitly defined `Decoder` instance for our response type `T` to decode the received HTTP response into the correct case class:

```scala
private def sendReceive[T <: Response : Decoder](req: HttpRequest): Future[T] =
    Http().singleRequest(req).flatMap(r => Unmarshal(r.entity).to[T])

def getAccountsList: Future[AccountsListResponse] = {
  val req = // build request...
  sendReceive[AccountsListResponse](req)
}
```

As you can see here the entire communication chain is asynchronous and we're only operating with futures throughout. This is one of the advantages of Akka HTTP as it allows us to build our communication in a completely async manner without unnecessary blocking/waiting on the response to be received or unmarshalling to complete. This will especially come in handy when I show you later how to process responses with chunked transfer encoding and map them as streams of data. Stay tuned for Part II.
