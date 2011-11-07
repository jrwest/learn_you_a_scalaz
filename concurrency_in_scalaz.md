# Concurrency in Scalaz (WIP)

*todo: transition from previous chapter*

In this section we'll explore how Scalaz applies functional programming ideas to make your concurrent code easier to write and reason about, and less prone to bugs.

## Let's talk about the `Future`

Before we start, let's briefly cover `Future`s to get some background on what we're going to talk about in this section. Simply put, `Future[T]` is a class that represents work that will give you a `T`, but isn't done yet. Futures are in the Java standard library, and Java makes is relatively easy to start computations in the background and return `Future`s to represent those computations. Here's how you'd do that in Scala (you didn't think we'd go back to Java land, did you?)

	scala> import java.util.concurrent.{Executors, Callable}
	import java.util.concurrent.{Executors, Callable}

	scala> val svc = Executors.newCachedThreadPool
	svc: java.util.concurrent.ExecutorService = java.util.concurrent.ThreadPoolExecutor@1bd60b6a

	scala> val future = svc.submit(new Callable[String] { override def call = {Thread.sleep(20); "Done!" }})
	f: java.util.concurrent.Future[String] = java.util.concurrent.FutureTask@6668a2cc
	
	scala> f.get
	res0: String = Done!

Notice the call to `svc.submit(new Callable…)` - that's what turns your code into a `Future` that executes it concurrently. When you call `get` on that future, you wait until the future is done and has a result.

`Futures` give a pretty high level abstraction that helps you write safe & easy-to-reason-about concurrent code. That's what we were going for, right? Well, almost. The one problem with futures arises when you need to do implement a function that applies a function to a bunch of futures like this:

	def applyToAllResults[F, G](futures:List[Future[F]], fn:F => G):List[G]

A naïve implementation would look like this:

	def applyToAllResults[F, G](futures:List[Future[F]], fn:F => G):List[G] = {
		var l = List[G]()
		for(f <- futures) {
			l = l :+ fn(f.get)
		}
		l
	}

But there's a problem here! You have to call f.get on every future in the list to get each result. If you have a future somewhere in the list that takes a long time to finish, then you hold up the loop until the future is done!  This property is called **composability**, and it turns out to be important if you need to deal with many `Future`s at once.

## I `Promise` it gets better!

Scalaz fixes that composability problem with `Promise`s (insert additional play-on-words for promise here). Remember, `Futures` had a problem in our previous example when we wanted to get the results of many futures in a list and pass each result to a function. Promises solve that problem by letting you give them a function, and automatically executing that function for you when the result is ready.

First, let's check out how to create a promise. Make sure you `import scalaz.concurrent._` for these examples:

	scala> lazy val e = {Thread.sleep(200); "Done!"}
	e: java.lang.String = <lazy>

	scala> val p = promise(e)
	p: scalaz.concurrent.Promise[java.lang.String] = <promise>

	scala> p.get
	res0: java.lang.String = Done!

Now that we can easily create promises, let's go back to our `applyToAllResults` function. The first step in our implementation is to tell each promise to execute our function when it's done:

	def getPromisesForAllResults[F, G](promises:List[Promise[F]], fn:F => G) = for (p <- promises) yield p.map(fn)

We're almost there - we now have a function that returns a list of promises, which will contain the results of applying `fn` on the result of each promise. Let's finish it off:

	def applyToAllResults[F, G](promises:List[Promise[F]], fn:F => G) = {
		val promisesForAllResults = getPromisesForAllResults(promises, fn)
		val mappedPromise = promisesForAllResults.parMap((p:Promise[G]) => p.get)
		mappedPromise.get
	}

The key to this code is `parMap`. `parMap` took our function `(p:Promise) => p.get` and returned a `Promise`, which we called `mappedPromise`. `mappedPromise`'s contains a `List[G]`, which is exactly the type we need to return! When we call `get` on `mappedPromise`, we'll get our `List[G]` back. Notice that we now have all of the individual `get` calls on each `Promise[F]` happening in the background, and now we can call one `get` call on a **new** promise, and get back the result <sup>1</sup>.

I hope you can see the benefit to this approach with `Promise`s over the previous approach with `Future`s. There are a lot of problems that need to use multiple concurrent operations, and `Promise`s give a hopefully easy to reason about joining those concurrent operations together.

But `Promise`s have a caveat that makes them a poor choice for some problems: once a `Promise` is complete, that's it, you're done. Do not pass go. Do not collect $200. What if you want to do the same concurrent computation repeatedly? Go on if you'd like to find out how!

## `Actor` like you're impressed

You might already be familiar with [Scala Actors](http://www.scala-lang.org/api/current/scala/actors/Actor.html) or [Akka Actors](http://akka.io/docs/akka/1.2/scala/actors.html). If you aren't, not to worry! Scalaz's actors are similar to the other forms, but simpler & purer.

Actors are similar to Promises because they execute a function concurrently. The difference lies in the interface they give you. A promise starts its function in the background & then lets you get the result when it's done, but a Scalaz actor can execute its function in the background as many times as is necessary, and anyone can send an asynchronous message to the actor in order to execute that function. Also, the actor's function always returns Unit, so `Actor` doesn't give you a direct way to get a return value like `Promise` does.

Let's start with a simple example. Remember that actors do their work in the background. You can always quickly send a message to and it'll be put on the actor's mailbox queue. So you can do whatever you want in an actor without worrying about blocking your thread. Let's look at a simple example that does logging very inefficiently:

	val logger = actor(s:String => {
		for(c in s) {
			println(c)
			Thread.sleep(500)
		}
	})
	logger("Log message 1 for great good")()
	logger("Log message 2 for great good")()
	logger("Log message 3 for great good")()

As we can see, doing logging takes a loooooooing time (approximately (s.length / 2) seconds per log string)! But we can call logger as much as we want without waiting. The log messages will be queued up, and will be executed in order in the actor sometime in the future.

Also, notice that `()` after the logger(…) call. `logger(…)` actually returns a `() => Unit`, so we have to actually execute that returned function by adding the `()` after the initial call to logger.

### Going big

Now that we have the basics, let's consider a bigger example. Imagine we have a very popular iPhone game called Furious Pigeons. There are a variety of different pigeons in the game, each of which is identified by a unique string. Somewhere in your code, there's a case class that looks like this: `case class Pigeon(kind:String)`.

There are billions of pigeons in the game, and you'd like to count the number of pigeons of each kind. You have a `List[Pigeon]` called `l` that has all sorts of different pigeon types, and you'd like to count the number of each pigeon. With the help of a Scalaz actors and a ConcurrentHashMap to hold the counts, we can do that quickly:
	
	import java.util.concurrent.ConcurrentHashMap
	import java.util.concurrent.atomic.AtomicInteger
	import util.Random
	import scalaz._
	import Scalaz._
	
	val counts = new ConcurrentHashMap[String, AtomicInteger]()
	
	//a simple function to calculate the counts of Pigeons in a List
	def calculateCounts(pigeons:List[Pigeon]) = {
		for(p <- pigeons) {
			Option(counts.putIfAbsent(p.kind, new AtomicInteger(1))) match {
				case Some(atomicInt:AtomicInteger) => atomicInt.incrementAndGet()
				case None => //atomic integer already set
			}
		}
	}

	//let's create a bunch of actors that can count pieces of the pigeons list
	val numCountActors = 100
	val countActors = for(i <- 0 until numCountActors) yield actor(( calculateCounts _ ))

	//let's create an actor that can take in the monstrous list and distribute it across all of our countActors
	val distributorActor = actor((l:List[Pigeon]) => {
		val rand = new Random()
		//the size of the chunks of Pigeons we want to count in each countActor
		val chunkSize = 20
		def distribute(startInclusive:Int, endExclusive:Int) {
			if(startInclusive > l.size) {
				return
			}

			val actor = countActors(rand.nextInt(countActors.size))
			actor(l.slice(startInclusive, endExclusive))()
			distribute(endExclusive, endExclusive + chunkSize)
		}
		distribute(0, chunkSize)
	})

	val tonsOfPigeons = getAMonstrousAmountOfPigeons()
	distributorActor(tonsOfPigeons)()

### Divide and Conquer

You might recognize what we did here as a divide and conquer pattern. We split a large list into smaller pieces, and did a calculation on each. Since we're doing addition, which is associative, we can do the calculations in any order and come up with the correct result.

<sup>1</sup> We could have done the map and parMap in the same step, but we split them here for clarity.