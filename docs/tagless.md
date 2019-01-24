# Tagless-final, to the point.

I have been using tagless final a lot lately. I am very satisfied with the approach, but have not find an article that expresses its value in a "down to earth" kind of way. This is my attempt at providing one.

## Preambule

Though it may be required to understand the contents, this article will **not** give an explanation of what the following notions are, and expects from the reader some basic understanding.

* higher-kinded types
* typeclasses
* monad transformers (OptionT, EitherT)

This article was originally written in Markdown, and the snippets are verified for both compile and run using the awesome [mdoc](https://scalameta.org/mdoc/docs/installation.html).

The only dependencies required to compile the snippets are the following (in SBT syntax)

```scala  
"org.typelevel" %% "cats-effect" % "1.1.0",
"org.typelevel" %% "cats-mtl-core" % "0.4.0"
```

To reduce the amount of code in each snippet, we're making the following imports available to all the snippets in the article .

```scala mdoc
// typeclass instances and syntactic sugar
import cats._
import cats.data._
import cats.implicits._
```

Note : these imports are quite helpful to discover features from cats. When your knowledge of the library improves, you should strive to import only the necessary rather than wildcards.

## Intro

### What problem(s) does the tagless-final approach solve ?

It maximises decoupling between various parts of application/program. By doing
so, it minimises the amount of refactoring needed when new concerns arise / new features require implementing. Therefore it saves you time and effort, which in turn saves your employer money.

It also allows you to build a mental framework for problem solving that is clear, flexible, consistent, and simple to defend in a debate of ideas.

### What does it cost ?

Performance-wise, next to nothing. Training-wise, it depends on your ability to reason in abstractions. However, there are countless people willing to help in the community for the sheer satisfaction of knowledge transfer. The developers I know who actually took the time to understand the concepts involved have
expressed a fair amount of satisfaction doing so.

### What is it ?

**tagless final** is essentially a way of writing interfaces (and implementations of these interfaces), which involves abstracting over the **effect context**.

```scala mdoc
// F[_] is an abstract effect context
trait KVStore[F[_]]{
  def put(key : String, value : String) : F[Unit]
  def get(key : String) : F[Option[String]]
  def delete(key : String) : F[Unit]
}
```

### Wait ? context ?

When you create an interface, if you're not familiar with the tagless technique,you're most likely deciding (knowingly or not) on a context its methods will work against. For instance, you could have written our `KVStore` interface sporting a synchronous context as such :

```scala mdoc
trait KVStoreSync {
  def put(key : String, value : String) : Unit
  def get(key : String) : Option[String]
  def delete(key : String) : Unit
}

// which is completely equivalent to

trait KVStoreId {
  // cats.Id[A] = A
  def put(key : String, value : String) : Id[Unit]
  def get(key : String) : Id[Option[String]]
  def delete(key : String) : Id[Unit]
}
```

Or you could decide for it to work against an asynchronous context, as such :

```scala mdoc
import scala.concurrent.Future

trait KVStoreFuture {
  def put(key : String, value : String) : Future[Unit]
  def get(key : String) : Future[Option[String]]
  def delete(key : String) : Future[Unit]
}
```

The tagless interface does not reference `Future`, nor `Id`, nor any other **effect type**, but rather declares that there is one, called `F`, and that what `F` is does not matter at this point.

### You mentioned "effects" ?

The definition of effect is "a change which is a result or consequence of an action or other cause." When you write a program, you likely have to address some concerns that might have an impact on the overall computation, such as validation, optionality, error handling ... these are effects. In functional programming, effects are usually associated to **datatypes** that have the **capability** to support/handle them, such as `Either`, `Option`, `Try`.

### What if different methods of my interfaces work with different effects ?

The should not : if methods are grouped in an interface, it's an anti-pattern to have one return `Future` and another return `Try`. You should always have some reason for grouping methods together in an interface, such as some rule binding them :

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

The rule is that each piece of logic should declare the minimal set of **capabilities** the effect type `F` needs to support for it to work. In this case, we need **sequentiality** as the calls need to be composed in a sequential manner (ie the for-comprehension).

That capability of sequentiality is provided by what is often called the **Monad** typeclass (interface). It lets you compose calls sequentially, which means that the result of one function can be passed as an input to the next.

In the scala ecosystem, the two main libraries that provide that typeclass/interface are [cats](https://github.com/typelevel/cats) and [scalaz](https://github.com/scalaz/scalaz)

With cats, the code at call site looks like this :

```scala mdoc
def verify[F[_] : Monad] // Declaring the capability the code needs
    (kvStore : KVStore[F]) : F[Boolean] = {
  import kvStore._
  for {
    _     <- put("key", "value")
    value <- get("key")
  } yield value === Some("value")
}
```

### How do I call this generic function ?

It needs an implementation of `KVStore`. Let's go ahead and implement one.

## Version 1

For the sake of giving a simple example, we'll start one with a dirty implementation that uses mutability :

### The implementation

```scala mdoc
import scala.collection.mutable.{ Map => MMap }

class MutableKVStore(map : MMap[String, String])
extends KVStore[Id] {

  def put(key : String, value : String) : Id[Unit] =
    map += (key -> value)

  def get(key : String) : Id[Option[String]] =
    map.get(key)

  def delete(key : String) : Id[Unit] =
    map -= key

}
```

### The call site

```scala mdoc
{
  val kvStore : KVStore[Id] =
    new MutableKVStore(MMap.empty)

  verify(kvStore) // Boolean
}
```

We've implemented a dirty/dumb KVStore, and asserted that our generic `test` function works against it.

If you're wondering, it works because a `Monad` instance is provided for `cats.Id`. I recommend implementing one yourself as an exercise, it is trivial and can help you build an instinct for the `Monad` abstraction.

## Version 2

As a rule of thumbs, you should avoid relying on mutability (unless you're implementing a piece of logic where performance is critical and low-level operations are required). Let's provide an implementation that manipulates state in a referentially transparent manner. For that, we'll use the `State` datatype. Eugene Yokota wrote a decent [blog entry](http://eed3si9n.com/herding-cats/State.html) about it that I recommend to read if you are not familiar with it.

### The implementation

```scala mdoc
type Data = Map[String, String]
type StateEffect[A] = State[Data, A]

object StateKVStore extends KVStore[StateEffect]{

  def put(key : String, value : String) : StateEffect[Unit] =
    State.modify(_ + (key -> value))

  def get(key : String) : StateEffect[Option[String]] =
    State.inspect(map => map.get(key))

  def delete(key : String) : StateEffect[Unit] =
    State.modify(_ - key)
}
```

### The call site

```scala mdoc
{
  val kvStore : KVStore[StateEffect] =
    StateKVStore

                     // Peels :
  verify(kvStore)    // StateEffect[Boolean]
    .runA(Map.empty) // Eval[Boolean]
    .value           // Boolean
}
```

As you can see, even though the **effect type** changed from `Id` to `State`, calling the `verify` method did not require any glue code, because both datatypes support `sequentiality` by virtue of having an associated lawful `Monad` instance.

In the `State` iteration, the only difference is the calls to "run" the state and peel the layers to access the boolean value, but the body of the `verify` function has remained the same.

### Is that it ... ?

Ideally, the implementation also needs to abstract over the **effect context**, and refer the **capabilities** rather than **datatypes**. The reason is that **capabilities** can be supported by several datatypes, and reversely a single **datatype** potentially supports more capabilities than a piece of logic actually needs.

Expressing the signature of a construct in terms of **capabilities** rather than **datatypes** is an approach to inversion of control that abides by the least power principle and increases separation of concerns.

The **cats** and **scalaz** ecosystem provides **typeclasses** encoding of
a fair number of **capabilities**, which are usually enough to express most of your logic.

## Capabilities, typeclasses and datatypes

Before we continue, I'd like to point out the great [infographic](https://github.com/tpolecat/cats-infographic) by Rob Norris that describes types/typeclasses provided by the **cats** library, and their hierachy.

TODO : insert table

## Version 3

This time, we'll be manipulating state through the `MonadState` typeclass.
It is provided by the [cats-mtl](https://github.com/typelevel/cats-mtl) library. We'll start by adding some imports :

```scala mdoc
import cats.mtl._
import cats.mtl.implicits._
```

### The implementation

```scala mdoc
type StateCapa[F[_]] = MonadState[F, Data]

class KVStoreImpl[F[_] : StateCapa]
extends KVStore[F]{

  val state = implicitly[StateCapa[F]]

  def put(key : String, value : String) : F[Unit] =
    state.modify(_ + (key -> value))

  def get(key : String) : F[Option[String]] =
    state.inspect(map => map.get(key))

  def delete(key : String) : F[Unit] =
    state.modify(_ - key)
}
```

### The call site

```scala mdoc
{
  // StateEffect supports the capability StateCapa
  // needed by the implementation, because cats-mtl
  // provides an instance of the MonadState typeclass
  // for the State datatype
  val kvStore : KVStore[StateEffect] =
    new KVStoreImpl

                     // Peels :
  verify(kvStore)    // State[Data, Boolean]
    .runA(Map.empty) // Eval[Boolean]
    .value           // Boolean
}
```

As you can see, nothing changes at call site.

## But still, what did we gain ?

If you place yourself at a particular instant in the early life of a codebase, there is not much point gain. You know all the concerns your application needs to handle and you can elect one effect type that will work. In the latest iteration for instance, the effect type elected was `cats.data.State`.

However, the point of the approach is that it facilitates pivoting. Let's add some requirements, and decide that deleting a key that does not exist from the store should result in an error.

### Proving the compatibility

If we are to add erroring logic, the `StateEffect` is not gonna be enough. We need to add a **layer** to our datatype that will act as a channel for errors.

```scala mdoc
{
  type Error = String
  type StateLayer[A] = State[Data, A]
  type ErrorLayer[A] = EitherT[StateLayer, Error, A]
  type EffectStack[A] = ErrorLayer[A]

  val kvStore : KVStore[EffectStack] =
    new KVStoreImpl

                     // Peels :
  verify(kvStore)    // ErrorLayer[Boolean]
    .value           // StateLayer[Either[Error, Boolean]]
    .runA(Map.empty) // Eval[Either[Error, Boolean]]
    .value           // Either[Error, Boolean]
}
```

This essentially means that with zero additional effort, the current code was able to run against a datatype that supports more capabilities.

### How does it work though ?

All we did was writing logic by declaring using **capabilities** (typeclasses) rather than concreter **effect datatypes** . By doing so, we've abided by the least power principle : a piece of logic does not have access to more information than what it actually needs, and we've made it more difficult to add code in a place it does not belong.

Capabilities can often (not always) transit through monad transformers :

Therefore :
* `ErrorLayer` supports `Monad` because the `StateLayer` it wraps supports it.
* `ErrorLayer` supports `MonadState` because the `StateLayer` it wraps supports it.

The glue-code already exists and and is thoroughly tested. We gain access to it through the imports of `cats.implicits._` and `cats.mtl.implicits._`.

### About the ordering of the layers

The order in which the layers are stacked can also change :

```scala mdoc
{
  type Error = String
  // changing order of effects
  type ErrorLayer[A] = Either[Error, A]
  type StateLayer[A] = StateT[ErrorLayer, Data, A]
  type EffectStack[A] = StateLayer[A]

  val kvStore : KVStore[EffectStack] =
    new KVStoreImpl

                     // Peels :
  verify(kvStore)    // StateLayer[Boolean]
    .runA(Map.empty) // Either[Error, Boolean]
}
```

Since we reason in terms of **capabilities** rather than concrete **effect datatypess**, the business logic does not have to worry about the ordering of the effects. It is however worth noting that when you choose the **effect type**, the order can have an impact. For instance, wrapping the state effect in an erroring layer could mean that you wouldn't be able to inspect the state in the event of an error, so it might be better to do the opposite.


## Version 4

Now that we've established the compatibility of the logic, let's re-implement the KVStore to cater to the new requirement (erroring when removing an abstent key) :

### The implementation

```scala mdoc
type Error = String
type ErrorCapa[F[_]] = MonadError[F, String]

class KVStoreImpl2[F[_] : StateCapa : ErrorCapa]
extends KVStore[F]{

  val state = implicitly[StateCapa[F]]
  val effect = implicitly[ErrorCapa[F]]

  def put(key : String, value : String) : F[Unit] =
    state.modify(_ + (key -> value))

  def get(key : String) : F[Option[String]] =
    state.inspect(map => map.get(key))

  def delete(key : String) : F[Unit] =
    for {
      exists <- state.inspect(_.contains(key))
      _      <- if (exists) state.modify(_ - key)
                else effect.raiseError(s"not found : $key")
    } yield {}
}
```

### The call site

```scala mdoc
{
  type Error = String
  // changing order of effects
  type ErrorLayer[A] = Either[Error, A]
  type StateLayer[A] = StateT[ErrorLayer, Data, A]
  type EffectStack[A] = StateLayer[A]

  val kvStore : KVStore[EffectStack] =
    new KVStoreImpl2

                     // Peels :
  verify(kvStore)    // StateLayer[Boolean]
    .runA(Map.empty) // Either[Error, Boolean]
}
```

Note that we still haven't changed our `verify` function : it does not need to handle errors and therefore its set of capabilities has not changed.

Coming from a Java background, this is really liberating for me : the main business logic (`verify`) only needs to express a composition of calls, and it is exempt from the concern of error handling.

## The case of side-effects

So far, the code we have written has respected the boundaries of the JVM and has lived solely in memory. But most programs need to call upon functionalities that live outside of the JVM and handle calls that are classified as **impure**,
which means they either rely on the outside world (to know the time, to create a random value), or have an effect on the outside world (printing a value in a terminal, writing to a database)

There is a set of **capabilities** that are specifically designed to reason about these particular concerns and manipulate **impure** computations in a referentially-transparent way. The [cats-effect](https://typelevel.org/cats-effect/) library provides typeclass encoding of these capabilities, as well as **effect datatypes** that support them.

In addition, **cats-effect** provides **capabilities** for capturing callback-based asynchronous calls into the **effect type**, for performing computations concurrently, for scheduling calls, ... as well as low-boilerplate,performant, simple concurrency primitives for non-blocking state manipulation, clean resource-handling, atomicity ...

## The downsides

The approach of **tagless final** is not completely free of downsides :

* It does take some learning effort to get acquainted with the various typeclasses and build an instinct for which ones you need to solve a particular problem.
* I personally like using [context bounds](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html) when declaring **capabilities**, which involves using type aliases, and sometimes invoking the implicit instance via `implicitly`. This is a little bit of boilerplate.

The interfaces/typeclasses involved are very much worth exploring :

* The interfaces are minimal. Both **cats** and **scalaz** provide a fair amount of functions, but each interface/typeclass only holds a very small number of abstract core functions that are completely orthogonal to each other (ie none can be expressed in terms of the others), the rest being derived from the core functions.
* Each method of the interface/typeclass has a reason to exist with relation to its neighbour methods, which is described in (mathematical) **laws**. The rules might have more or less familiar names depending on your education in mathematics (such as **associativy**), but they allow you to reason consistently about the interface, and prevent you from worrying about inexistent edge cases.
* Although the Scala compiler is not powerful enough to statically check the laws, **cats** and **scalaz** have implemented them as [reusable tests](https://typelevel.org/cats/typeclasses/lawtesting.html) that run against generated input sets.
* There are more and more libraries that use these interfaces and thus maximise the amount of possible integration. Amongst which : [fs2](https://fs2.io/) and [https://http4s.org/]

