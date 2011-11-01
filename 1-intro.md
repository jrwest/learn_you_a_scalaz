#  All Aboard the Train to Scalaz

Welcome to the world of Scalaz! If you've arrived here maybe you've heard of it? It's a beatiful place full of typeclasses and functional programming paradigms. What are those you ask? Read on and we'll find out together. 

I began writing this after re-reading a similar introduction in the great Haskell book of a similar (and original) name, "Learn You a Haskell for Great Good". It is intended to be an introduction to Scalaz for intermediate and above level Scala developers and as a means to improve my own knowledge of the library as well. 

There are several versions of Scalaz out in the wild. We will be discussing Scalaz6 which is the most recent stable release. Scalaz7 has several forks and is still a work in progress. It also a [rewrite](http://code.google.com/p/scalaz/wiki/Scalaz7) of portions of the library so it is safe to assume code used here will not be the same in Scalaz7. 

# What you'll need to get started

In order to follow along you will need Scala installed on your machine, as well as [SBT](https://github.com/harrah/xsbt/wiki). All examples will be tested against Scala 2.9.1. The SBT version I am using is 0.11.0. 

If you clone the Scalaz6 [repository](https://github.com/scalaz/scalaz) you can use SBT and the Scala REPL as a convenient way to play with the library. After you clone it, change into the project's base directory in your terminal and boot up SBT.

	$ cd path/to/scalaz
	$ sbt
	…
	>

We can compile the code and run a Scala console with the everything we need on the classpath.

	> compile
	…
	> scalaz-core/console
	[info] Starting scala interpreter...
	[info] 
	Welcome to Scala version 2.9.1.final (Java HotSpot(TM) 64-Bit Server VM, Java 1.6.0_26).
	Type in expressions to have them evaluated.
	Type :help for more information.

	scala> 
 
   
# The Main Imports

Throughout this tutorial when playing the Scalaz we will assume you have imported the following two things:

	scala> import scalaz._; import Scalaz._
	import scalaz._
	import Scalaz._

Although I have found it easy to use only certain parts of the library where I need them, I do suggest you always use these two imports (this is one of the things being discussed as part of the work on Scalaz7). If not, you may try to use some functionality you would expect to be in scope but wont be. If you use an IDE that imports things for you automatically or on-demand you may need to take some extra precaution to make sure you change the imports to the ones above.

# Before we learn, lets just do some cool stuff

Before we dive into things like typeclasses and purely functional data structures lets just have some fun with Scalaz and see how can use some basic things in our code. 

## A Useful Way to Work With Options

Your probably familiar with how to apply a function to an `Option[A]` if it is not a `None` and if not return a default value. You use `map` and `getOrElse`. When applied to an Option `map` applies the function if the value exists. 

	scala> some(3) map { _ + 1 }
	res0: Option[Int] = Some(4)

	scala> (none : Option[Int]) map { _ + 1 }
	res3: Option[Int] = None

You may have noticed I'm using lowercase versions of `Some` and `None`. This is something Scalaz gives us. The problem with using `Some` and `None` is they return instances of that type. Sometimes this can mess with the Scala compiler and implicits when it expects an `Option[A]`. Instead `some` and `none`, the lower-case alternatives are defined with the more general result type `Option[A]`.

	def some[A](a: A): Option[A] = Some(a)
	def none[A]: Option[A] = None

You can also turn an instance of `A` into an `Option[A]` by calling `some` on it. This is sometimes convenient when defining a method that returns option as well.

	scala> 3.some
	res11: Option[Int] = Some(3)

The `getOrElse` method returns an instance of `A` on an `Option[A]` and is given to us by the standard library. Put together with map we are able to write expressions like,

	scala> 3.some map { _ + 1 } getOrElse 2
	res12: Int = 4

	scala> (none : Option[Int]) map { _ + 1 } getOrElse 11
	res14: Int = 11

Scalaz gives us a more expressive way of writing the same expression:

	scala> some(3) some { _ + 1 } none { 2 }
	res15: Int = 4

	scala> none[Int] some { _ + 1 } none { 11 }
	res16: Int = 11

This statement isn't just more expressive however. It also can help us from doing something we didn't intend. 

	scala> none[Int] some { _ + 1 } none { "11" }
	<console>:14: error: type mismatch;
	 found   : java.lang.String("11")
	 required: Int
           none[Int] some { _ + 1 } none { "11" }

Where as `getOrElse` may lead to us accidentally returning an unintended type,

	scala> none[Int] getOrElse { "2" }
	res27: Any = 2

## Checking Two Optional Values at Once

Scalaz also gives us a useful way to check the existence and values of two optional values at once. In the imperative style we would probably use a set of boolean functions combined by boolean operators in an if expression or nested if expressions. That's not to say we can't and don't do this in the functional programming world but in the imperative style we are often talking about `null` handling. In Scala we have the benefit of using the `Option` type instead of `null` and it is verbose and cumbersome to use `option.isDefined` as well as extracting the value in if expressions. A useful way to work with `Option` values is to pattern match on them using `match`. We are familiar with this. But how do we check both exist at the same time. We could use a `(Option[A], Option[B])` but then there are four cases to handle two of which we dont care about. Instead we can use the `<|*|>` operator (I don't know what to call it, spaceship, bar star?) to take an `Option[A]` and an `Option[B]` and get back an `Option[(A, B)])`. If either of the two operands are `None` the resulting value with be `None`. Otherwise, they will be `Some[(A, B)])`. Now, when we pattern match we only have to handle the two cases we care about: when they both exist and otherwise. 

	scala> 3.some <|*|> 2.some match {
		| case Some((a, b)) if a == b => println("equal")
		| case Some((_, _)) => println("not equal")
		| case None => println("at least one doesnt exist")
		| }
	not equal

	scala> 3.some <|*|> none[Int] match {
		| case Some((a, b)) if a == b => println("equal")
		| case Some((_, _)) => println("not equal")
		| case None => println("at least one doesnt exist")
		| }
	at least one doesnt exist
	

Especially when we have many cases to handle when both exist, this is a much cleaner and clearer approach than using `(Option[A], Option[B])` despite the funky operator. You can also do some other cool things mixing this operator with for comprehensions or some of the things above.

	scala> for ((a, b) <- 3.some <|*|> 2.some) yield a + b
	res22: Option[Int] = Some(5)

	scala> (3.some <|*|> none[Int]) some { t2 => t2 } none { (0, 0) }
	res23: (Int, Int) = (0,0)

Also, as we will learn at a later time you can use this operator with more than just options. 

# What's Next, Now & Beyond

We've seen some cool things Scalaz gives us but we haven't really covered why. Also, it's not exactly aparent how, unless you are familiar with Pimp My Library pattern which Scalaz uses to extend the types defined by the Scala standard library, its own types and even yours. In the next section we will cover what Scalaz really gives us as well as this pattern it uses to make it convenient and concise to use in our code. 

In later sections we will cover different parts of the Scalaz library and how we can use them in our code. We will talk about some of the theory behind why but only enough to understand why it makes our code better. 