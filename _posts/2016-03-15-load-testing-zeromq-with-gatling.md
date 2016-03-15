---
layout: post
title: Load testing ZeroMQ with a custom DSL for Gatling
excerpt: "Instructions for implementing a custom DSL for non-http load testing on top of the Gatling framework."
tags: [gatling, scala, dsl, load testing, zeromq, akka, akka-stream]
modified: 2016-03-15
comments: true
---

I've been working on a [pet project](https://github.com/sbilinski/reactive-zeromq) of my own lately, which is an [Akka Streams](http://doc.akka.io/docs/akka/current/scala.html) wrapper for [ZeroMQ](http://zeromq.org/) sockets. In order to test the library and the target application itself, I wanted to build some **load tests**, which could determine how well they performed under an arbitrary pressure.

There are several ways one could approach this, but if you want to stick to an existing framework and work on a *Scala* project, than [Gatling](http://gatling.io/) is probably a good choice. The framework provides an elegant [DSL](http://gatling.io/#/cheat-sheet/2.1.7) and can be integrated with an *sbt* project using the official [plugin](http://gatling.io/docs/2.1.7/extensions/sbt_plugin.html). You can find a brief summary of the core features on the project's [home page](http://gatling.io/#/why) and in [this](https://www.thoughtworks.com/insights/blog/gatling-take-your-performance-tests-next-level) blog post by *ThoughtWorks*.

*Gatling* is protocol-agnostic at its core, which allows to build some custom testing scenarios on top of it. However, most examples are focused on *HTTP* or *JMS* at the moment, which makes it somehow difficult to begin with.

This article is a summary of my own process with a custom *DSL* implementation for *ZeroMQ*. I hope you'll find it useful, if you're implementing a protocol of your own.

## Big picture

The initial test case I took was a simple **publisher-subscriber** fan-out, which would occur over relevant **ZeroMQ** sockets. My **Gatling** simulation would look something like this:

{% highlight scala %}
import com.mintbeans.rzmq.gatling.Predef._
import io.gatling.core.Predef._
import scala.concurrent.duration._

class MySimulation extends Simulation {
  val endpoint = "tcp://127.0.0.1:5555"
  val topic = "some-topic"

  val zmqConfig = ZMQ.endpoint(endpoint).receiveTimeout(100 millis)
  val subscribers = scenario("Consume Topic").exec(
    subscriber("simple-subscriber").topic(topic).messagesToRead(5)
  )

  setUp(
    subscribers.inject(atOnceUsers(10))
  ).protocols(zmqConfig).maxDuration(1 minutes)
}
{% endhighlight %}

As you've probably figured out, the idea is to create a group of simultaneous **subscribers**, that will listen to some fixed `topic` over **TCP/IP** and read an arbitrary number of messages.

The **publisher** is assumed to be working on the given address. It's implementation will be skipped in this article, but you can assume, that it will provide a constant stream of ascending integers.

## Preparing the DSL

I started by analyzing the [JMS](https://github.com/gatling/gatling/tree/master/gatling-jms) module, which is a fairly simple protocol implementation, comparing to [HTTP](https://github.com/gatling/gatling/tree/master/gatling-http). I've also read the [Gatling Protocol Breakdown](https://github.com/jagregory/gatling/blob/master/GatlingProtocolBreakdown.md) by James Gregory, which I highly recommend to begin with.

**NOTE:** The following implementation is based on Gatling `2.2.0-M3`. Several things had changed between `2.1.x` and `2.2.x` in the context of custom protocol implementation (in a good way). I would recommend checking the relevant GitHub branch, if you intend to work with `2.1.x`.

### Predef & DSL trait

The convention is to put all protocol-specific **DSL** into a `Predef` object for easy importing in your `Simulation` classes. Let's start by defining it:

{% highlight scala %}
object Predef extends ZmqDsl

trait ZmqDsl {
  ...
}
{% endhighlight %}

### Protocol definition

The protocol needs a **Gatling** representation, which will provide shared attributes for establishing a connection:

{% highlight scala %}
case class ZmqProtocol(endpoint: String, receiveTimeout: FiniteDuration) extends io.gatling.core.config.Protocol
{% endhighlight %}

We need to express the `ZmqProtocol` creation process in the **DSL**. The convention is to create a set of relevant builder classes, that will form a hierarchy:

{% highlight scala %}
object ZmqProtocolBuilderBase {
  def endpoint(address: String) = ZmqProtocolBuilder(address)
}

case class ZmqProtocolBuilder(endpoint: String, receiveTimeout: FiniteDuration = 500 millis) {

  def receiveTimeout(duration: FiniteDuration) = copy(receiveTimeout = duration)

  def build = ZmqProtocol(
    endpoint = endpoint,
    receiveTimeout = receiveTimeout
  )
}

trait ZmqDsl {
  val ZMQ = ZmqProtocolBuilderBase
  ...

  //By convention, a protocol instance is built using an implicit conversion
  import scala.language.implicitConversions

  implicit def zmqProtocolBuilder2zmqProtocol(builder: ZmqProtocolBuilder): ZmqProtocol = builder.build
}
{% endhighlight %}

Our simulation code could look like this at the moment:

{% highlight scala %}
import Predef._

val zmqConfig1 = ZMQ.endpoint(endpoint)
val zmqConfig2 = ZMQ.endpoint(endpoint).receiveTimeout(500 millis)

//A builder instance should be converted to a Gatling protocol implicitly
setUp(...).protocols(zmqConfig1)
{% endhighlight %}

You can add intermediary builders of course, if protocol creation should happen in several steps, that take some mandatory attributes on the way.

**Example**: the "base" builder could produce an "endpoint step" builder, which would take some mandatory parameters and create a valid `ZmqProtocolBuilder` in the end. The final builder is assumed to take **optional** configuration only. That's why we're creating a `copy` of it once an attribute is altered (both builders are immutable and can produce a valid protocol).

### Action definition

An `Action` is a top level abstraction for executing a concrete step along a scenario. It is implemented as an **Akka** actor, that receives `Session` messages for triggering the action.

In this example we'll implement a **subscriber action**, that will:

1. Connect to a `ZMQ_PUB` socket
2. Consume some integers from the stream
3. Terminate

#### ActionBuilder DSL

Instead of producing an `Action` directly, we need to create a hierarchy of **DSL** builders, that will be converted to an `ActionBuilder` instance when evaluated in the simulation class. This "action builder" will be used by **Gatling** to create concrete `Action` instances during scenario execution later.

Let's define the **DSL** and the `ActionBuilder` for the **subscriber** process:

{% highlight scala %}
case class SubscriberProcessBuilderBase(processName: String) {
  def topic(topic: String) = SubscriberProcessBuilder(processName, topic)
}

case class SubscriberProcessBuilder(processName: String, topic: String, messagesToRead: Int = 1) {
  def messagesToRead(numberOfMessages: Int) = copy(messagesToRead = numberOfMessages)

  def build(): ActionBuilder = SubscriberActionBuilder(processName, topic, messagesToRead)
}

case class SubscriberActionBuilder(processName: String, topic: String, messagesToRead: Int) extends ActionBuilder {
  //NOTE: This is 2.2.0-M3 specific
  def build(system: ActorSystem, next: ActorRef, ctx: ScenarioContext) = {
    val protocol = ctx.protocols.protocol[ZmqProtocol]
    val statsEngine = ctx.statsEngine

    system.actorOf(SubscriberAction.props(processName, protocol, topic, messagesToRead, statsEngine, next), actorName("subscriber"))
  }
}

trait ZmqDsl {
  ...

  def subscriber(processName: String) = SubscriberProcessBuilderBase(processName)

  //By convention, an ActionBuilder instance is created using an implicit conversion from a parent builder
  import scala.language.implicitConversions

  implicit def subscriberProcessBuilder2ActionBuilder(builder: SubscriberProcessBuilder): ActionBuilder = builder.build()
}
{% endhighlight %}

We can use this in the **simulation** as follows:

{% highlight scala %}
import Predef._

val subscribers = scenario("Consume Topic").exec(
  //Our SubscriberProcessBuilder should be implicitly converted to an ActionBuilder instance
  subscriber("simple-subscriber").topic("some-topic").messagesToRead(5)
)

setUp(
  subscribers.inject(atOnceUsers(10))
).protocols(zmqConfig)
{% endhighlight %}

#### Action implementation

One missing piece is the `SubscriberAction` implementation, which is created by the `SubscriberActionBuilder` in the previous step. We'll implement it next:

{% highlight scala %}
class SubscriberAction(processName: String, protocol: ZmqProtocol, topic: String, messagesToRead: Int, val statsEngine: StatsEngine, val next: ActorRef) extends Interruptable with Failable {

  val expectedTopicBytes = ByteString(topic)
  val mainCtx = new ZContext(1)

  override def postStop() = {
    mainCtx.close()
  }

  override def executeOrFail(session: Session): Validation[String] = {
    val ctx = ZContext.shadow(mainCtx)
    try {
      statsEngine.logRequest(session, processName)

      val requestStartDate = now()
      val socket = ctx.createSocket(ZMQ.SUB)
      socket.setReceiveTimeOut(protocol.receiveTimeout.toMillis.toInt)
      socket.subscribe(topic.getBytes(ZMQ.CHARSET))
      socket.connect(protocol.endpoint)
      val requestEndDate = now()

      val messageStream = {
        val serverStream = for (i <- Stream.range(0, messagesToRead)) yield readMessage(socket)
        serverStream.takeWhile(validContent _).map(extractPayload _)
      }

      val responseStartDate = now()
      val messages = messageStream.toList
      val responseEndDate = now()

      val timings = ResponseTimings(requestStartDate, requestEndDate, responseStartDate, responseEndDate)

      messages match {
        case list if list.size == messagesToRead =>
          statsEngine.logResponse(session, processName, timings, Status("OK"), None, None)
          Success("OK")
        case list =>
          statsEngine.logResponse(session, processName, timings, Status("KO"), None, None)
          Failure(s"Invalid list size ${list.size}")
      }
    } finally {
      ctx.close()
      session.terminate()
    }
  }

  @inline
  private def validContent(response: Option[Either[String, ByteString]]): Boolean = response match {
    case Some(Right(_)) => true
    case _ => false
  }

  @inline
  private def extractPayload(response: Option[Either[String, ByteString]]): ByteString = response match {
    case Some(Right(bytes)) => bytes
    case _ => throw new IllegalStateException("Extracting payload from an invalid response.")
  }

  @inline
  private def readMessage(socket: ZMQ.Socket) = {
    Option(ZMsg.recvMsg(socket)).map { zMsg =>
      zMsg.asScala.toList.map(zFrame => ByteString(zFrame.getData))
    }.map {
      case topic :: payload :: Nil if topic == expectedTopicBytes => Right(payload)
      case topic :: payload :: Nil => Left(s"Invalid topic: ${new String(topic.toArray, ZMQ.CHARSET)}")
      case _ => Left("Invalid message format")
    }
  }

  @inline
  private def now() = System.currentTimeMillis()
}
{% endhighlight %}

The most important part are the `statsEngine` calls, which are responsible for storing request and response stats. If everything was setup correctly, then you should be able to get a valid report after all tests were executed.

<figure>
    <a href="/images/posts/gatling_zeromq_test_results.png" title="Gatling test results."><img src="/images/posts/gatling_zeromq_test_results.png" alt="Gatling screenshot" ></a>
</figure>

## Conclusions

I hope you enjoyed the article and didn't get discouraged by `implicit` conversions and longish code examples.

You can find a complete version of the **DSL** in the [reactive-zeromq](https://github.com/sbilinski/reactive-zeromq) project on **GitHub**. Please take it with a grain of salt - it's a small experiment of mine, not a full blown communication library.
