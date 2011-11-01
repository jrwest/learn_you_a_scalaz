# Concurrency in Scalaz (WIP)

*todo: transition from previous chapter*

In this section we'll explore how Scalaz applies functional programming ideas to make your concurrent code easier to write and reason about, and less prone to bugs.

## Let's talk about the `Future`

Before we start, let's briefly cover `Future`s to get some background on what we're going to talk about in this section. Simply put, `Future[T]` is a class that represents work that will evaluate to a `T` that's happening in the background which might not be finished yet. Of course, Java also makes it easy to start computations and return `Future`s to represent those computations. Here's how you'd do that in Scala (you didn't think we'd go back to Java land, did you?)

	scala> import java.util.concurrent.{Executors, Callable}
	import java.util.concurrent.{Executors, Callable}

	scala> val svc = Executors.newCachedThreadPool
	svc: java.util.concurrent.ExecutorService = java.util.concurrent.ThreadPoolExecutor@1bd60b6a

	scala> val future = svc.submit(new Callable[String] { override def call = {Thread.sleep(20); "Done!" }})
	f: java.util.concurrent.Future[String] = java.util.concurrent.FutureTask@6668a2cc
	
	scala> f.get
	res0: String = Done!

Notice the call to `svc.submit(new Callable…)` - that's what turns your code into a `Future` that executes it concurrently. When you call `get` on that future, you wait until the future is done and has a result.

`Futures` give a fairly high level abstraction that helps you write safe & easy-to-reason-about concurrent code. That's what we were going for, right? Well, almost. The one problem with futures arises when you need to do implement a function that applies a function to a bunch of futures like this:

	def applyToAllResults[F, G](futures:List[Future[F]], fn:F => G):List[G]

A naïve implementation would look like this:

	def applyToAllResults[F, G](futures:List[Future[F]], fn:F => G):List[G] = {
		var l = List[G]()
		for(f <- futures) {
			l = l :+ fn(f.get)
		}
		l
	}

But there's a problem here! You have to call f.get on every future in the list to get each result. If you have a future somewhere in the list that takes a long time to finish, then you hold up the loop until the future is done! That is, `Future`s are not easily **composable**.

## I `Promise` it gets better!

Scalaz fixes the composability problem with `Promise`s (insert additional play-on-words for promise here). Remember, `Futures` posed a problem in our previous example when we wanted to call a function on the results of many futures in a list. Promises let you tell them the function to call when the result is ready.

First, let's check out how to create a promise. It's less code to create promises using Scalaz. Here's the canonical example. Make sure you `import scalaz.concurrent._` for these examples:

	scala> lazy val e = {Thread.sleep(200); "Done!"}
	e: java.lang.String = <lazy>

	scala> val p = promise(e)
	p: scalaz.concurrent.Promise[java.lang.String] = <promise>

	scala> p.get
	res0: java.lang.String = Done!

Now that we can easily create promises, let's go back to our `applyToAllResults` function. The first step in implementing that function is to get our function called on each promise when the promise is complete, without calling get:

	def getPromisesForAllResults[F, G](promises:List[Promise[F]], fn:F => G):List[Promise[G]] = {
		var l = List[Promise[G]]()
		for(p <- promises) {
			l = l :+ p.map(fn)
		}
		l
	}

We're almost there - we now have a function that returns a list of promises, which will contain the results of applying `fn` on the result of each promise. Let's finish it off

	def applyToAllResults[F, G](promises:List[Promise[F]], fn:F => G):List[G] = {
		val promises = getPromisesForAllResults(promises, fn)
		val mappedPromise = promises.parMap(promise => promise.get)
		mappedPromise.get
	}

The key to this code is the parMap function. That function took our function `promise => promise.get` and returned a `Promise`, which we called mappedPromise, that contains a `List[G]` - exactly the type we need to return! After all that, we can finally call `get` on mappedPromise, and we'll get our `List[G]` back. The key, though, is that all the `get` calls on each `Promise` in the list will be called in parallel, and `mappedPromise.get` will return when all of the get calls have returned.

I hope you can see the benefit to this approach with `Promise`s over the approach with `Future`s. Many problems require the use of multiple concurrent operations, and `Promise`s give a straightforward and hopefully easy to reason about (and functional!) mechanism for joining those concurrent operations together in interesting ways.

Promises are a great tool for executing one-off computations and getting the result when they're done, but once a promise is complete, that's it, you're done. Do not pass go. Do not collect $200. But what if you want to do the same concurrent computation repeatedly? Keep on truckin' to find out how.

## `Actor` like you're impressed

You might already be familiar with [Scala Actors](http://www.scala-lang.org/api/current/scala/actors/Actor.html) or [Akka Actors](http://akka.io/docs/akka/1.2/scala/actors.html). If you aren't, not to worry! Scalaz's actors are similar to the other forms, but are actually somewhat simpler. Let's start with a short introduction to Scalaz Actors. You can think of actors as a way to execute functions that look like `T => Unit` concurrently. Another way to think about an actor is a promise on which you never call `get`. Let's look at a simple example that builds an actor that does some rather inefficient logging for us:

	val logger = actor(s:String => {
		for(c in s) {
			println(c)
			Thread.sleep(500)
		}
	})
	logger("Log message 1 for great good")()
	logger("Log message 2 for great good")()
	logger("Log message 3 for great good")()

As we can see, doing logging takes a loooooooing time (approximately (s.length / 2) seconds per log string)! But we can call logger as much as we want without waiting that long because the code inside of `actor(…)` is executed concurrently to the code that calls `logger(…)`. Another thing to note is that our call to `logger(…)` returns a `Function0` that returns Unit. That is, we get back a function that looks like: `() => Unit`, so we have to actually execute that function by adding the `()` after the initial call to logger.

This code shows the simplest form of a Scalaz actor. In fact, Scalaz actors are extremely simple, but remember that you can do anything you want in the function that you pass to the actor. Let's look at a more complex example that acts as a proxy and forwards data to the appropriate place:
	
	val proxyActor = actor(e:(Option[(String, Actor[String])], Option[(Int, Actor[Int])]) => {
		def doMatch[T](opt: Option[T, Actor[T]]) {
			opt match {
				case Some(tup) => tup._2(tup._1)()
				case None => //do nothing
			}
		}
		doMatch(e._1)
		doMatch(e._2)
	})

**TODO: figure out some more novel uses for Scalaz actors**