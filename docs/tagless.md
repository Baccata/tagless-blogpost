# Tagless-final, to the point.

I've been using the tagless final a lot lately, and I haven't found a satisfying / down to earth explanation of why it is an extremely useful
way of structuring programs.

## Preambule

Though it may be required to understand the contents, this article will **not** give an explanation of what the following things are, and expects from the reader some very basic understanding.

* higher-kinded types
* typeclasses
* monad transformers (OptionT, EitherT)

This article was originally written in Markdown, and the snippets are verified for both compile and run using the awesome [mdoc](https://scalameta.org/mdoc/docs/installation.html).

The only dependencies required to compile the snippets are the following (in SBT syntax)

```scala  
  "org.typelevel" %% "cats-effect" % "1.1.0",
  "com.typesafe.akka" %% "akka-actor" % "2.5.19"
```

## Intro

### What problem(s) does the tagless-final approach solve ?

It maximises decoupling between various parts of application/program. By doing
so, it minimises the amount of refactoring needed when new concerns arise / new features require implementing. Therefore it saves you time and effort, which in turn saves your employer money.

It also allows you to build a mental framework for problem solving that is clear, flexible, consistent, and simple to defend in a debate of ideas.

### What does it cost ?

Performance-wise, next to nothing. Training-wise, it depends on your ability to reason in abstract concepts, but there are countless people willing to help in the community for the sheer satisfaction of knowledge transfer. The developers I know who actually took the time to understand the concepts involved have
expressed a fair amount of satisfaction doing so.

### What is it ?

**tagless final** is essentially a way of writing interfaces (and implementations of these interfaces), which involves removing the knowledge of which concrete **effect context** is involved in it.

```scala mdoc
// F[_] is an abstract effect context
trait KeyValueStore[F[_]]{
  def put(key : String, value : String) : F[Unit]
  def get(key : String) : F[Option[String]]
  def delete(key : String) : F[Unit]
}
```

### Wait ? context ?

When you create an interface, if you're not familiar with the tagless technique,you're most likely deciding (knowingly or not) on a context its methods will work against. For instance, you could have written our `KeyValueStore` interface sporting a synchronous context as such :

```scala mdoc
trait KeyValueStoreSync {
  def put(key : String, value : String) : Unit
  def get(key : String) : Option[String]
  def delete(key : String) : Unit
}
```

Or you could decide for it to work against an asynchronous context, as such :

```scala mdoc
import scala.concurrent.Future

trait KeyValueStoreFuture {
  def put(key : String, value : String) : Future[Unit]
  def get(key : String) : Future[Option[String]]
  def delete(key : String) : Future[Unit]
}
```

### You mentioned "effects" ?

The definition of effect is "a change which is a result or consequence of an action or other cause." When you write a program, you likely have to address some concerns that might have an impact on the overall computation, such as validation, optionality, error handling ... these are effects. In functional programming, effects are usually associated to datatypes that have the **capability** to support/handle them, such as `Either`, `Option`, `Try`.

### So what's the "effect" exposed by the synchronous interface you wrote?

It's the effect of no-effect. You can reify it as such :

```scala mdoc
type Id[A] = A
trait KeyValueStoreId {
  def put(key : String, value : String) : Id[Unit]
  def get(key : String) : Id[Option[String]]
  def delete(key : String) : Id[Unit]
}
```

### What if the methods of my interfaces work with heterogenous effects ?

The should not. You should always have some reason for grouping
methods together in an interface, such as some rule binding them :

``` 
put(K,V) then get(K) should give me V
```

which you can formalise as such :

```scala
for {
  _     <- put("key", "value")
  value <- get("key")
} yield value === Some("value")
```

As for the typed returned by this expression ? It does not matter when you write this code. Whether it's `Future`, `Try`, `Either`, the code is going to be the same, and the rule should hold, given all methods return data in the same **effect context**. That is the whole point of **tagless final** : what does not matter to a piece of logic should be available to it.

### How is that code even going to compile ?

The rule is that each piece of logic should declare the minimal set of **capabilities** your effect type needs to support for it to work. In this case, we need **sequentiality** as the calls need to be composed in a sequential manner (ie the for-comprehension).

That capability of sequentiality is often provided by the **Monad** interface,
which is a minimal construct that lets you compose calls sequentially

In the scala ecosystem, the two main libraries that provide that interface (amongst many others) are [cats](https://github.com/typelevel/cats) and [scalaz](https://github.com/scalaz/scalaz)

With cats, the code at call site looks like this :

```scala mdoc
import cats.Monad
import cats.implicits._

def test[F[_] : Monad] // Declaring the capability the code needs
    (kvStore : KeyValueStore[F]) : F[Boolean] = {
  import kvStore._
  for {
    _     <- put("key", "value")
    value <- get("key")
  } yield value === Some("value")
}
```

### How is that generic function called ?

It needs an implementation of `KeyValueStore`. Let's go ahead and implement one.

## Version 1

For the sake of the article, we'll start one based on an akka actor in order to isolate the in-memory state and manipulate it a thread-safe way. We'll use the [ask pattern](https://doc.akka.io/docs/akka/2.5/actors.html#ask-send-and-receive-future) to communicate with the Actor holding the state, which involves `Future`

### The protocol

Let's start by defining an ADT that will mirror the interface and serve as a protocol for communicating with the Actor.

```scala mdoc
sealed trait KeyValueProtocol
case class Put(key : String, value : String) extends KeyValueProtocol
case class Get(key : String) extends KeyValueProtocol
case class Delete(key : String) extends KeyValueProtocol
```

### The actor

Then, the actor itself

```scala mdoc
import akka.actor._

class KeyValueActor() extends Actor {
  def withState(map : Map[String, String]) : Receive = {
    case Put(key, value) =>
      sender ! context.become(withState(map + (key -> value)))
    case Get(key) =>
      sender ! map.get(key)
    case Delete(key)     =>
      sender ! context.become(withState(map - key))
  }
  def receive = withState(Map.empty)
}

object KeyValueActor {
  def props() = Props(new KeyValueActor())
}
```

We're also creating a factory method to reduce the boilerplate in following snippets.

```scala mdoc
def withActor[A](f : ActorRef => A) : A = {
  val system = ActorSystem()
  val kvActor : ActorRef = system.actorOf(KeyValueActor.props())
  val res = f(kvActor)
  system.terminate()
  res
}
```

### The KeyValueStore implementation

```scala mdoc
import akka.actor._
import akka.pattern.ask
import akka.util._
import scala.concurrent.duration._

class ActorKeyValueStoreFuture(ref : ActorRef, timeout : FiniteDuration)
extends KeyValueStore[Future]{

  private implicit val askTimeout = Timeout(timeout)

  def put(key : String, value : String) =
    (ref ?  Put(key, value)).mapTo[Unit]

  def get(key : String) =
    (ref ?  Get(key)).mapTo[Option[String]]

  def delete(key : String) =
    (ref ?  Delete(key)).mapTo[Unit]

}
```

### The call site

```scala mdoc
withActor { kvActor =>

  // Bootstrapping
  val kvStore : KeyValueStore[Future] =
    new ActorKeyValueStoreFuture(kvActor, 500.millis)

  // Necessary to get an instance Monad[Future]
  import scala.concurrent.ExecutionContext.Implicits.global
  import cats.instances.future.catsStdInstancesForFuture

  // Calling logic
  val resultFuture : Future[Boolean] = test(kvStore)

  // Waiting for the result
  scala.concurrent.Await.result(resultFuture, 1.second)
}
```

### Is that it ... ?

No. Ideally you also want the implementation to abstract over the **effect context**, but similarly to the example, you often have to call upon external constructs and libraries that do not (such as the `ask` pattern from akka).

The approach is to **lift** the effect type/mechanism these libraries elected to use into your abstract effect type `F`.

## Version 2

Let's change our implementation to further defer the selection of the effect type. The first version had it target Future directly. In this iteration, we
will lift the `Future` into an abstract `F`. In order to do that, we will use capabilities provided from [cats-effect](https://github.com/typelevel/cats-effect). That library provides datatypes and typeclasses (capabilities) to generically deal with synchronous/asynchronous/concurrent computations in a convenient and boilerplate free manner.

### So how do I turn a Future to an F ?

There is no typeclass that expresses the ability to lift a `Future` into an abstract `F`. However, there is one that provides the ability to lift `cats.effect.IO` into an abstract `F`. It is called `LiftIO`. Conveniently, `IO` supports a `fromFuture` method. With it, we can modify our `KeyValueStore` implementation as such :

```scala mdoc
import akka.actor._
import akka.pattern.ask
import akka.util._

import cats.effect._

import scala.concurrent._
import duration._

class ActorKeyValueStore[F[_] : LiftIO]
    (ref : ActorRef, timeout : FiniteDuration)
  extends KeyValueStore[F]{

  // The lifting function
  def liftFuture[A](futureA : => Future[A]) : F[A] =
    LiftIO[F].liftIO(IO.fromFuture(IO(futureA)))

  private implicit val askTimeout = Timeout(timeout)

  def put(key : String, value : String) =
    liftFuture((ref ?  Put(key, value)).mapTo[Unit])

  def get(key : String) =
    liftFuture((ref ?  Get(key)).mapTo[Option[String]])

  def delete(key : String) =
    liftFuture((ref ?  Delete(key)).mapTo[Unit])

}
```

### What about call site ?

What changes is that we've added a requirement on `F` to support the capability of lifting an `IO` into it. This capability is not supported by `Future`, but `IO` can be trivially lifted into an `IO`, therefore it does support the `LiftIO` capability.

```scala mdoc
withActor { kvActor =>
  // Bootstrapping
  val kvStore : KeyValueStore[IO] =
    new ActorKeyValueStore(kvActor, 500.millis)

  // Calling the logic
  val resultInEffect : IO[Boolean] = test(kvStore)

  // Getting the result
  resultInEffect.unsafeRunSync()
}
```

## But still, what did we gain ?

If you place yourself at a particular instant in the early life of a codebase, there is not much point gain. You know all the concerns your application needs to handle and you can elect one effect type that will work. In the latest iteration for instance, the effect type elected was `cats.effect.IO`.

However, the point of the approach is to prevent glue-code from being needed when you amend parts of it.

For instance, say you need to address two new concerns, **optionality** and **erroring** . At the **end of the world** of the application (typically the main method), the effect-type will be impacted as you will need to add **layers** to it, each layer reflecting a particular concern.

```scala mdoc 
import cats.data._
import cats.effect._

type Error = String

type AsyncLayer[A] = IO[A]
type OptionLayer[A] = OptionT[AsyncLayer, A]
type ErrorLayer[A] = EitherT[OptionLayer, Error, A]

type ComplexEffect[A] = ErrorLayer[A]
```

The magic of the approach then unravels, as absolutely no change is required for our existing logic to run in a more complex datatype.

```scala mdoc
withActor { kvActor =>

  // no glue code needed to instantiate the KeyValueStore
  val kvStore : KeyValueStore[ComplexEffect] =
    new ActorKeyValueStore(kvActor, 500.millis)

  // no glue code needed to run the function
  val resultInEffect : ComplexEffect[Boolean] = test(kvStore)

  // only glue code needed : peeling the various layers
  resultInEffect
    .value // peeling error layer
    .value // peeling option layer
    .unsafeRunSync() // running async computation
}
```

This essentially means that with zero additional effort, the current logic is compatible with any new logic that will be added to tackle the new concerns.

### How does it work though ?

All we did was write both our `KeyValueStore` implementation and our business logic by decalring the minimal set of **capabilities** (typeclasses) they need need to operate. One had declared a `Monad` capability, the other a `LiftIO` one.

By doing so, we've abided by the least power principle : a piece of logic does not have access to more information than what it actually needs, and we've made it more difficult to add code in a place it does not belong.

To understand the hierachy of power between capabilities, you can refer yourself to the great [infographic] (https://github.com/tpolecat/cats-infographic) by Rob Norris that describes types provided by the **cats** library, .

Capabilities can often (not always) transit through monad transformers : the glue-code [already exists](https://github.com/typelevel/cats-effect/blob/master/core/shared/src/main/scala/cats/effect/LiftIO.scala#L39) and is thoroughly tested, you don't have to write it yourself.

Therefore :
* `OptionLayer` supports `LiftIO` because `AsyncLayer` supports it.
* `OptionLayer` supports `Monad` because `AsyncLayer` supports it.
* `ErrorLayer` supports `LiftIO` because `OptionLayer` supports it.
* `ErrorLayer` supports `Monad` because `OptionLayer` supports it.

We could also change the order in which `Option` and `Error` layers are stacked together, and the code would still hold.

# Capabilities and typeclasses

It does take a fair amount of effort to get acquainted with the various capabilities and build an instinct for which ones you need to solve a particular problem.

The reasons why you should learn them are the following :

* They are minimal. Both **cats** and **scalaz** provide a fair amount of functions, but each interface/typeclass only holds a small number of functions that are completely orthogonal to each other (ie none can be expressed in terms of the others).
* They are lawful : each method of the interface/typeclass has a reason to exist with relation to its neighbour methods, which is described in (mathematical) **laws**. The rules might have unfamiliar names such as **associativy**, but they allow you to reason consistently about the interface and apply the same patterns no matter the actual concrete datatype you end up using.
* Although the Scala compiler is not powerful enough to statically check the laws, but **cats** and **scalaz** have implemented them as **reusable tests** that run against generated input sets.


