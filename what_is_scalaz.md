# What is Scalaz?

Scalaz's Google Code page says it is a library of "Type Classes and Pure Functional Data Structures." You may have heard it also uses the Pimp My Library pattern or seen references to its pimpz. What does this mean? In this section we will talk about all of that. Knowing what they are isn't the only thing that is interesting, however. Its how we can use them and why they make our code better that we really want to know. We will only begin to scratch the surface of the "hows" & the "whys" in this section. First, we must go over the "whats". If you are familiar with the "introductory" functional programming & data structures and typeclasses gibber gabber you may find this section too far up in the clouds. In the next sections we will get our hands dirty and cover things in detail. Check out the section on pimping your Scala if your not familiar with the pattern. We will also discuss the core pimpz of Scalaz.

## Typeclasses == Classes of Types

Typeclasses are classes of types. I can hear my 1st english teacher telling me, "you can't use the word your defining in the definition, young man." Correct he is -- and was, although I can't quite remember the word.

So let's get a better defintion. 

	Typeclasses define a finite set of behaviors on a possibly ininite set of types. 

Let's pick that apart. We have two sets of things in play here. The first is a finite set of behaviors (read functions, or if you prefer methods). The second is a possibly infinite set of types. A `possibly infinite set of types`, that needs some more explanation. Infinite doesn't always mean all of the possible Scala types we could ever write. We might be talking about them, but we could also be talking about all types we could write that are parameterized by one, or even two types. In functional programming and Scalaz we are often talking about types that meet some mathematical property, often from a branch of Mathematics called `Category Theory`<super>1</super>.  This is why `possibly infinite` is important. We may know the number of types that meet a certain mathematical property right this moment but we human beings are a smart species and it is possible there  are more types, maybe onese we haven't even written yet, that we have yet to prove meet the property. A simple example, and a not too mathematical one, would be all of the types for which we could compare two instances with one another. You may be thinking, "but wait `==` lets me compare any two things." This isn't Scala's `==` operator (behaviour) though. This is comparing two instances *of the same type and only the same type*. In Scalaz this is the `===` operator and support for one of your own classes would need to be defined, but we'll talk about that more in a minute.


At first, its easy to think that typeclasses sound just like classes in warm-and-fuzzy OOP-land. `Any` defines methods on every Scala type, you might say. But how would you define a superclass of all types parameterized by one type? You can't with classes. Thanks to the features of Scala we have another solution.

### Typeclasses in Scala

Typeclasses are like interfaces on steroids. Hmm, That sounds familiar…quick grab your nearest Scala book, its probably in the chapter about traits! Traits, in the purely OOP sense allow for multiple inheritance. For our purposes they let us define not only abstract behavior but also concrete behavior, typically built on-top of the abstract stuff (or maybe not). Scala, also, has a really powerful type system that you can use to control how traits are used as well as their behaviors. Combine these two together and you can start writing Scala typeclasses. But wait, I said abstract behavior. That sounds like inheritance. Functional programming doesn't mean we have to throw the object-oriented baby out with the bath water. We need to implement that abstract behavior somewhere, its how we get those finite sets of types for our typeclasses, and in Scala we use inheritance and concrete classes or objects to do it. 

Traits can be mixed into any class hierarchy and are a great way to help you think about typeclasses. Typeclasses describe functionality that can be performed on a bunch of seemingly unrelated types as long as those types satisfy certain conditions.

### Why Typeclasses?

We now have a better understanding of what these typeclass things are and how they differ from classes in OOP but it may or may not be clear as to why they are useful. Why would we want to use typeclasses in our code? How do they make writing and maintaing our code better? These are not easy questions to answer but we will begin too. The rest, I hope, will become aparent as you read on and learn how to use the ones Scalaz provides for you and how to make your own classes (types) members of some typeclasses as well.

In classic Object Oriented Programming we use a class hierarchy to organize related domain objects and functionality shared between them. `Any` gives us all of the functionality we can use on any Scala object. For the most part it makes sense, but in many cases it doesn't. A common pitfall of OOP is having to "fudge" the definition of a method so that we can move it up the class hierarchy. I call this "dirtying the class hierarchy." You begin to litter superclasses with methods that must be shared across subclasses but as a result some subclasses that "dont really" fit with the behavior get it anyway<super>3</super>. This leads to more edge case handling in the method and even causes clients of the method to do more edge case handling as well. I mentioned the `==` operator before because it is a great example of this. Especially in Scala. At first, it seems obvious that we can compare any two things for equality. Why shouldn't I be able to ask if `"1" == 1`? You can in Scala if you want, but more often that not you will ask it accidentally and you will be surprised when your code doesn't run the expected branch of the `if` expression because comparing two instances of different types in Scala will always return false. 

	scala> 1 == "1"
	res1: Boolean = false

The truth is its not Scala's fault that `==` works this way. Its a product of the JVM it runs on. Scala does its best to tell you at compile time that your most likely doing something you didnt mean too. 

	warning: comparing values of types Int and java.lang.String using `==' will always yield false
              1 == "1"
                ^
The warning is a nice helper but typesafe compilers are even more helpful when they give us compiler errors when we are about to do something wrong. The problem is we can't make `==` typesafe because it is defined on `Any`<super>2</super>. Many times in our own classes we handle this with something like

	class A {
		…
		def equals(o: Any) = o match {
			case oa: A => // real checking here
		   case _ => false
		}
		…
	}

This is where typeclasses make our code better. We can avoid this repetition and code smell. Instead we can define a typeclass for `all types that can be compared to others of the same type for equality`. Let's not talk about the typeclass right away but just think about how we could define a utility method to do typesafe equals. We might do something simple like,

	scala> object MyEquals { 
			| def tEqual[A](a: A, b: A) = a == b
			| }
	defined module MyEquals

	scala> import MyEquals._
	import MyEquals._

And then use it to check equality between some instances of the same type,
  
	scala> tEqual(1, 1)
	res2: Boolean = true

	scala> tEqual(1, 2)
	res3: Boolean = false

What about different types:

	scala> tEqual(1, "3")
	res4: Boolean = false

Hmm, we didn't get a compiler error (well a repl error in this case) like we should for typesafe equals. You may have spotted why. If not, look back at our definition of `tEqual` it takes a type parameter `A`. Scala's type inferencer will look for a common supertype of the two arguments passed in if you do not specify the type of `A` when calling `tEqual`. When we can `tEqual(1, "3")` we are actually calling the eqiuvalent of `tEqual[Any](1, "3")`. Specifying `A` we get the expected error.

	scala> tEqual[Int](1, "3")
	<console>:12: error: type mismatch;
	 found   : java.lang.String("3")
	 required: Int
	              tEqual[Int](1, "3")


Ok, we have a typesafe equals function now, but its burdensome to use, verbose and we cant use it as an operator. Also, its not defined as part of a typeclass its just a utility method contained in an object. But wait, I said this stuff was supposed to clean up our code. It will, and we'll talk about how Scalaz does that for us, at least in part, by discussing the `Equal` typeclass.

### A Scalaz Example - The Equal Typeclass

So the `Equal` typeclass encompasses all types for which any two instances of that type can be compared for equality. In Scalaz, it is defined as

	// https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Equal.scala
	trait Equal[-A] {
		def equal(a1: A, a2: A): Boolean
	}

Note: I typically do not want to cover how Scalaz is implemented. I'm more interested in discussing how to use it for the betterment of mankind! Rarely will I include all or even parts of the definitions in this tutorial except for when I think it might benefit our understanding of things. When included, there will also be a link to where the code can be found. Scalaz can be a bit hard to navigate at times even as well laid out as it is. In this case I find it useful to see how `Equal` is defined and to discuss it a bit. This is also not the full story of `Equal` (some of which you will see in a moment). 

Like we said above, Scala trait's help us to define typeclasses. Before we talk about the `equal` function lets look at the definition of the trait: `trait Equal[-A]`. The definition leverages more than traits but also the Scala type system, like we said it would. `Equal` is parameterized by a single type. We use the type variable `A` to represent this type. In type classes it is always the case that our traits will be parameterized by some number of types. The trait lets us group similar behavior (the finite set) and the type parameters let us talk about the set of types the behaviors apply too. Let's use this trait to define a better utility method than the `tEqual` we defined before. This is not how Scalaz makes the equal behavior to you, we are still working towards that, but it will give us an understanding of this typeclass. In this case, we are going to show how `Int` is a member of `Equal`.

	scala> object IntEqual extends Equal[Int] {
		| def equal(a1: Int, a2: Int): Boolean = a1 == a2
		| }
	defined module IntEqual

	scala> IntEqual.equal(1, 2)
	res0: Boolean = false

	scala> IntEqual.equal(1, 1)
	res1: Boolean = true

	scala> IntEqual.equal(1, "3")
	<console>:10: error: type mismatch;
	 found   : java.lang.String("3")
	 required: Int
              IntEqual.equal(1, "3")

This time around there is no ambiguity to the type inferencer, we can't apply `IntEqual.equal` to an `Int` and a `String`; we get the compiler error we expect. `Equal`'s type paramter `A` is interesting and helps us do this. It is prefixed by a minus (`-`). In Scala this means that `A` is *contravariant*. This means if we have `A1` which is a subtype of `A2` then `Equal[A1]` is a supertype of `Equal[A2]`.

At first, at least to me, contravariance is never as clear as its counterpart covariance. Why is `Equal[A2]` a subtype of `Equal[A1]` when `A1` is a sub-type of `A2`? The answer lies in the one behavior defined on the `Equal` typeclass, `equal(a1: A, a2: A): Boolean`. The function takes two instances of `A` and returns a `Boolean`. This is what we expect from typesafe equals, so let's use that knowledge to explore why `A` is contravariant in this case. Imagine we define a subclass of `Int` called `RangedInt` that represents a subset of set of integers represented by `Int`. It makes sense that we could pass one of these into `IntEqual.equal`, since `RangedInt` is a subset of the domain of `Int` we know that they can be compared for equality. However, we can't pass an `Any` to `IntEqual.equal` because it is a supertype of `Int`. By the way, that's exactly what we are trying to prevent! So `Equal[Int]` is a supertype of `Equal[Any]` even though `Any` is a supertype of `Int`.  Now this makes more sense. `Equal[Any]` provides a broader set of functionality (the ability to compare any two Scala objects using our typesafe equals) than `Equal[Int]` does, the same way `Any` defines broader functionality than `Int`. 

Hint: `Equal[Any]` is not given to you by Scalaz because it isn't very useful and kind of defeats the purpose of the `Equal` typeclass. Scala's blend of OOP and FP is extremely powerful but it also let's you define useless things like `Equal[Any]` which you should not do. 

So the `Equal` typeclass sounds pretty good. It gives us a very powerful typesafe `equal` function. Still, it is not exactly clear how we get instances of `Equal`. Scalaz includes `Int` and many other types in the `Equal` typeclass but how do we use this stuff. Also, `equal` still can't be used like an operator. Well, before we get to see the whole picture we need to learn how to Pimp our Scala. 

## Pimp my Ride

Oh damn, Xzhibit just showed up on the scene and he wants to pimp your code! Drive it over to the shop and we'll get started. Unlike the TV show though you don't have to wait until your ride is ready, please pimp along-side (fire up your scala shell and follow along). 

The *Pimp my Library* pattern allows you to extend existing classes with new functionality without changing their original definition. We call these extensions *pimps*. If you've written Ruby this may sound similar to Monkey Patching and it is. The difference is its typesafe and you get all that compile time goodness! Scala itself uses this pattern everywhere, its how your Java strings have super powers and how you can give them to your Java collections as well (by importing `scala.collection.JavaConversions._`). 

The key to the pattern is another feature of Scala: implicits. Using implicits you can convert one type to another without having to explicitly say so in your code. In our case we want to convert our type (or class) to a wrapper that provides the aditional functionality. Maybe you want to add the method `fortyTwo` to any instance of `List`. Lets do that. The first step is to define a class that is paramterized by a `List` (you can also use a trait, remember the key here is implicits) and define the `fortyTwo` method.

	scala> class ListFortyTwoW[T](xs: List[T]) {
		   | def fortyTwo: T = xs(41)
		   | }
	defined class ListFortyTwoW

`ListFortyTwoW` could be instantiated directly and then we can call `fortyTwo` on the list we passed into it,

	scala> (new ListFortyTwoW(Nil)).fortyTwo
	java.lang.IndexOutOfBoundsException: 41

but thats verbose, nasty and most importantly not very interesting. What would be interesting is being able to call `List(1, 2, 3).fortyTwo`. That's not something given to us by the Scala Standard Library. When you call a method that is not defined for a type Scala will look in scope for an implicit function that converts the instance to a type that does. We need one of those, and as you might have guessed its pretty easy to define.

	scala> implicit def listToFortyTwoW[T](xs: List[T]) = new ListFortyTwoW[T](xs)
	listToFortyTwoW: [T](xs: List[T])ListFortyTwoW[T]

With the implicit in scope we can now call our method on a list. 

	scala> List(1, 2, 3).fortyTwo
	java.lang.IndexOutOfBoundsException: 41

We have just extended Scala lists! Scalaz uses this pattern to provide a succinct way of using typeclasses.

### Identity, MA, MAB

The core Pimps of Scalaz are [Identity](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Identity.scala), [MA](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/MA.scala) and [MAB](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/MAB.scala). At first these names  may seem somewhat non-sensical but once you know what they do it makes a bit more sense. `Identity` applies to all Scala types, `MA` to all types `M` parameterized by one type `A` and `MAB` to all types `M` parameterized by types `A` and `B` (remember these are not concrete types but type variables). It may also sound like these pimps are typeclasses but this is not the case. It is true that each of them do define some behavior common to all types but they are just highly general pimps (unlike the one we defined above that only works on lists). Remember, the pimps give us a convenient way to do this. In fact they include behavior from several typeclasses and are able to do so using implicits. Earlier I said typeclasses are defined using traits and the type system. This does not mean all traits that are polymorphic are typeclasses. Some of what is in these pimps are also convenience methods or notation. For example, `|>` is defined in `Identity` as, 

	def |>[B](f: A => B): B = f(value)

This function takes a function from `A => B` and calls it on the instance of `A` that `Identity` wraps, returning the resulting `B`.
### Back to the Equal Typeclass

Let's jump back to the `Equal` typeclass and see how Scalaz gives us the convenient `===` operator via `Identity`.

   // https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Identity.scala
  	def ===(a: A)(implicit e: Equal[A]): Boolean = e equal (value, a)

Note: Once again, the intention is not to discuss the implementation details of Scalaz at length. How `Equal` and the implicits are in scope is a path I encourage you to explore, but it wont be covered here.

The defintion of `===` has two argument lists, the second of which is implicit. Scala allows us to do this so that we can define more convenient to use functions. In the case of this defintion we can call `===` on any type `A` (`A` will first be implicitly converted to an `Identity[A]`) for any `A` which has a corresponding `Equal[A]` defined. This only works if both the implicit conversion from `A` to `Identity[A]` and an implicit `Equal[A]` are in scope. This is why the methods defined in `Identity`, `MA` and `MAB` don't always apply to all of the types the pimps encompass. If we don't have a `Equal[A]` for our `A` then it is not a member of the `Equal` typeclass and we can't use the `===` behavior on it.

With all of that in mind we have a pretty clear picture of what Scala typeclasses look like. They are representing using polymorphic traits and the scala type system. We provide convenient access to the behaviors they define on certain types using pimps and implicits.

## Purely Functional Data Structures

Scalaz also says it gives us purely functional datastructes. How do purely functional datastructures differ from the imperitave ones? We can derive the definition by thinking about the how programs written to a functional programming contract are pure along with our knowledge of Scala. 

Functional programming requires us to program without side-effects. We won't dive into the depths of discussing why functional programming but lets talk about side-effects since its pertinent to the discussion of functional data structures. We're all familiar with side-effects like reading and writing to a database, or printing text to the console. Mututating objects (changing the value's of their fields) are side-effects too. Since we can't use side-effects in programs written to the functional contract we can't mutate objects in our functions. This is the first pillar of functional data structures. We cannot perform *destructive updates* (cause side-effects) on them. 

We are familiar with some data structures like this already in Scala. `scala.collection.immutable.List` is a data structure that we cannot mutate. Once we have an instance of `List` that's it. We know from using Scala however that we can get new lists from our lists, however. And thats just it! We get NEW instances of our data structures instead of mutating them. Let's start with a list of 3 `Int`s

	scala> val a = List(1, 2, 3)
	a: List[Int] = List(1, 2, 3)

When he add an element to the head of the list and assign it to a new variable we do not change the original list as we know. We can also add a different element to the head of our list that `a` points to and not affect the first.

	scala> val b = 1 :: a
	b: List[Int] = List(1, 1, 2, 3)

	scala> a
	res0: List[Int] = List(1, 2, 3)

	scala> b
	res1: List[Int] = List(1, 1, 2, 3)

	scala> val c = 2 :: a
	c: List[Int] = List(2, 1, 2, 3)

	scala> b
	res4: List[Int] = List(1, 1, 2, 3)

	scala> c
	res5: List[Int] = List(2, 1, 2, 3)

We can also return how we built `b` and `c` from `a` using the `tail` method and show that their tails are equal.

	scala> b.tail
	res6: List[Int] = List(1, 2, 3)

	scala> c.tail
	res7: List[Int] = List(1, 2, 3)

	scala> b.tail == c.tail
	res9: Boolean = true

This brings us to the second pillar of functional data structures. They are *persistent*. This means that when we get a new instance of a functional data structure that has a transformation applied to it (like adding a number to a `List[Int]`) the old one is still around for us to use. With imperative data structures this is normally not the case. With a mutable map, if we change the value of a key that's it, the old value is gone.

Scalaz data structures obide by these rules. Many of them are very useful and we will cover them as we move through this tutorial. 

## Scalaz and the Unicodes

One important thing to note about Scalaz is it does make heavy use of unicode. For example there is an alias for `===`, `≟`. If you go perusing the code yourself you will noticed many more. I have found that for the most part each unicode operator has an ascii equivalent. If you follow the definitions you will find the ascii version and the non-operator function name as well. I will use the ascii operators mainly because they are still very nice and I don't know how to type many of the unicode equivalents. Also, I find some of the unicode operators like `≟` a bit hard to read (its an equals sign if a question mark on top if you are having a hard time too). 


<super>1</super> Category theory will be covered in this tutorial, however the primary focus is code not mathematics.
<super>2</super> How `==` is defined is actually a bit more complicated than just that one method but thats a topic that will be covered in detail in your Scala book. 
<super>3</super> Of course, the GOF give us lots of patterns to make our OOP code better but they don't help us clean up everything.

functional datastructures reference: http://www.cs.cmu.edu/~rwh/theses/okasaki.pdf

 