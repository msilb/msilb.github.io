---
layout: post
title: Scala Wrapper for Oanda REST API
comments: true
---

[Oanda](http://oanda.com) is in my opinion one of the best online brokers.
Originally only offering forex trading, Oanda has more recently expanded its offering to allow for trading of stock and commodity indices through the so-called CDFs (contract for difference).
You can read up on that stuff on their website.
In this post I would like to offer you some guidance on how to use Oanda as a broker for executing algorithmic / automated trading strategies.

Their new REST-based [API](http://developer.oanda.com/rest-live/introduction) can be used with any language in order to connect to Oanda's servers and fetch market data / place orders.
There main downsides to using the REST API directly is that it is not type-safe.
For instance, if you look at some of their examples using `curl` you will notice that all input parameters and returned values are essentially strings, even if the actual value is a number or a boolean or an enumeration.
This may be ok for simple testing but in a large trading application this will most certainly become a source for all kinds of bugs that are hard to track down.
That's why I created a Scala wrapper for Oanda's API called [Scalanda](https://github.com/msilb/scalanda), which provides a type-safe domain model for all input and output values.
It is based on [Akka](http://akka.io) to provide the non-blocking API, which means that all interaction has to be done in an actor-conform fashion.

Simply drop this line into your `build.sbt` to use it from your Scala/Akka-based trading application:

{% highlight scala %}
libraryDependencies += "com.msilb" %% "scalanda" % "{{site.version.scalanda}}"
{% endhighlight %}

Scalanda is internally utilizing the [Spray](http://spray.io) library as a non-blocking HTTP client to send requests to and receive responses from the Oanda server.
The core of the HTTP communication is the following piece of code:

{% highlight scala %}
def pipelineFuture[T <: Response](implicit unmarshaller: FromResponseUnmarshaller[T]): Future[HttpRequest => Future[T]] =
    for {
      Http.HostConnectorInfo(connector, _) <- IO(Http) ? Http.HostConnectorSetup(
        host = env.restApiUrl(),
        port = if (env.authenticationRequired()) 443 else 80,
        sslEncryption = env.authenticationRequired(),
        defaultHeaders = authTokenOpt
            .map(authToken => List(HttpHeaders.Authorization(OAuth2BearerToken(authToken))))
            .getOrElse(Nil)
      )
    } yield authTokenOpt match {
      case Some(authToken) => addCredentials(OAuth2BearerToken(authToken)) ~> sendReceive(connector) ~> unmarshal[T]
      case None => sendReceive(connector) ~> unmarshal[T]
    }
{% endhighlight %}

This probably requires some explanation.
As you can see here I am defining the so-called pipeline `Future`, which will be the base routing for processing all HTTP requests and receiving responses.
Essentially, what happens here is that first of all a new HTTP connection is established to Oanda's server using the URL specific for the selected environment (Sandbox, Practice or Production).
Also depending on the environment we need to send the authentication token.
Once the connection is successfully established, we return the `Future` of type `HttpRequest => Future[T]` denoting a function from `HttpRequest` to a `Future` of type `T`, where `T` is a sub-class of `Response`.

In a concrete example we can then use the base pipeline `Future` to request a list of tradeable instruments for a specific account:


{% highlight scala %}
case req: GetInstrumentsRequest =>
      log.info("Getting instruments: {}", req)
      val uri = Uri("/v1/instruments").withQuery(
        Query.asBodyData(
          Seq(
            Some(("accountId", accountId.toString)),
            req.fields.map(fields => ("fields", fields.mkString(","))),
            req.instruments.map(instruments => ("instruments", instruments.mkString(",")))
          ).flatten
        )
      )
      handleRequest(pipelineFuture[GetInstrumentsResponse].flatMap(_(Get(uri))))
{% endhighlight %}

Have a look at the [source code](https://github.com/msilb/scalanda) and feel free to ask me any questions related to usage of the API wrapper.
I tried to cover all functionality offered by Oanda's API but if you feel that something is missing or you would like to enhance the functionality, simply fork the repo and send a pull request.

Enjoy and good luck testing your trading strategies!

PS: I will cover a sample trading strategy I have implemented in Scala in one of the future posts. Stay tuned.