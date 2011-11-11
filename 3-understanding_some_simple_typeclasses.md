# Understanding Some Simple Typeclasses

Now that we have a better understanding of what Scalaz gives us let's explore some of the simpler typeclasses it defines. We'll revist `Equal` and discuss `Show` and `Order`. For flavor, we will also discuss `Semigroup` which is one of those typeclasses that apply to types meeting a couple of mathematical properties. 

As you read through the details of these typeclasses it may seem like some of the behavior they define overlaps with existing features of Scala. It's true that some portions of Scalaz overlap with what is given to us in the standard library. Scala is a very functional language as-is and Scalaz is a purely functional library, so that overlap makes sense. It's important to remember that Scalaz and typeclasses represent a different way of thinking about your code. Also, we mentioned in the last section Scalaz gives us some functional data structures that don't exist in the standard library. The biggest idea that I hope will continue to become more evident is that Scalaz usually gives a more general version of a behavior that we can use on a wider range of types.

The topics we will discuss in this section are also foundational topics. They will give us a better understanding of how typeclasses work. Some of them are typeclasses that are used to build other typeclasses. For example, we will talk about `Order` of which all members are also members of `Equal`. The `Monoid` typeclass which we will talk about later is built on top of the `Semigroup` typeclass.

## Typing our Safe Equals

Before, we talked about `Equal[-A]` to better understand what typeclasses are. Let's revisit it one more time briefly and learn some finer details about it. To get our hands dirty we will also make our own class a member. We won't always do this when touring different typeclasses, but `Equal` is relatively straight-forward and we want typesafe equals for our types as well. 

Members of the `Equal` typeclass include all of the types you'd expect including `Int`, `Byte`, and `String` as well as ones you may or may not have expected like `xml.NodeSeq`. For a full list take a look at the [definition](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Equal.scala). As we know this typeclass gives us the `===` operator for all member types. 

	scala> 'a === 'b
	res4: Boolean = false

	scala> 'a === 'a
	res5: Boolean = true

	scala> 1 === 1
	res6: Boolean = true

	scala> 1 === "2"
	<console>:14: error: type mismatch;
	 found   : java.lang.String("2")
	 required: Int
				  1 === "2"

If we try to call `===` on one of our own types however we will get an error. We expect this because we know we don't have an implicit conversion from `MyClass` to `Equal[MyClass]`.

	scala> class MyClass(a: Int)
	defined class MyClass

	scala> (new MyClass(1)) === (new MyClass(1))
	<console>:15: error: could not find implicit value for parameter e: scalaz.Equal[MyClass]
              (new MyClass(1)) === (new MyClass(1))

So, if we want to make our class a member of the `Equal` typeclass we need to do two things. First we need a definition of `Equal[MyClass]` and second we need the implicit conversion. We can do this all at once thanks how Scalaz defines its own members of the `Equal` typeclass.

	scala> object MyEqualsMembers extends scalaz.Equals {
	     | implicit def MyClassEqual: Equal[MyClass] = equalA
	     | }
	defined module MyEqualsMembers

	scala> import MyEqualsMembers._
	import MyEqualsMembers._

	scala> (new MyClass(1)) === (new MyClass(1))
	res9: Boolean = false

We'll step through how we went from `error: could not find implicit value for parameter e: scalaz.Equal[MyClass]` to `res9: Boolean = false` in a moment, but first, you may be wondering why false was returned. Under the hood, Scalaz is using the `==` operator given to us by Scala. Since we have not overriden `equals` in our code, `==` is performing reference equality, which is false in this case. If we redefine `MyClass` using a case class, which defines `equals` for us as we'd expect, we get `true`.

	scala> case class MyClass(a: Int)
	scala> MyClass(1) === MyClass(1)
	res1: Boolean = true
 
It makes sense that Scalaz uses the built in Scala standard library where it can, but it also makes it hard to understand why Scalaz is useful. Here, Scalaz acts as a utility to go farther than Scala does to prevent unintended actions. In our case, we know all types that are a member of `Equal` by looking at its definition and then adding all types we've added to the typeclass ourself.

Let's get back to how we made `MyClass` a member of `Equal`. Scala & Scalaz did all of the heavy lifting for us. All we did was make an object that extends `scalaz.Equals` that contained this function:

	implicit def MyClassEqual: Equal[MyClass] = equalA

The function is named `MyClassEqual`. This is a style choice and you should never have to call it directly. `equalA` is the reason we extend `scalaz.Equals`, a convenience trait that has functions for defining members of `Equal[A]`. `equalA` takes a type `A` and defines an `Equal[A]` using `==`. In our case `A` is `MyClass`. This was the first thing we needed to make `MyClass` a member. The second was an implicit. That was easy, we just marked our function with `implicit`, put it in scope and voila! We can now use `===` with `MyClass`. As a final bonus, we get `/==`, typesafe not equals.

	scala> MyClass(1) /== MyClass(2)
	res29: Boolean = true

## Showing Off

Another simple typeclass to explore while we solidify our understanding is `Show`. A type that is a member of `Show` can be represented as a string (in Scalaz it is a `List[Char]`). Wait, but don't we have `toString` in Java and Scala that lets us do this? Yes, but like many of the others, `Show` is a foundational typeclass inspired from Haskell. In Haskell, `Show` is the only equivalent of the familiar `toString`.

`Show` is probably one of, if not the single least interesting of the Scalaz typeclasses, but that fact also makes it pretty easy to understand. Like `===` and `Equal` we can call `show` on any type for which there is an implicit `Show[A]` defined in scope. If there isn't one we get a compiler error.

	scala> 3.show
	res0: List[Char] = List(3)

	scala> MyClass(1).show
	<console>:16: error: could not find implicit value for parameter s: scalaz.Show[MyClass]
              MyClass(1).show

Like we did previously with `Equal`, we can easily make `MyClass` part of the `Show` typeclass. I'll leave that as an exercise for you.

Even though it's simple, `Show` is still cooler than `toString` because we can override its definition from outside the class we're calling it on. Just like `===` was defined in `Identity` in the last section, `show` is defined similarly:

	def show(implicit s: Show[A]): List[Char] = s.show(value)

`show` takes an implicit `Show[A]`. When we called `3.show`, since we have the Scalaz imports, an implicit `Show[Int]` is in scope and its used as the one argument to the function. We could also *explicitly* pass our own `Show[Int]` to override the behavior. 

	scala> object CustomShow extends Show[Int] {
	     | def show(a: Int): List[Char] = ("My Show: %d" format a).toList
	     | }
	defined module CustomShow

	scala> 3.show(CustomShow)
	res4: List[Char] = List(M, y,  , S, h, o, w, :,  , 3)

If we make `CustomShow` implicit since its defined in a nearer scope we can use `show` like before and still modify its behavior.

	scala> implicit def implicitCustomShow = CustomShow
	implicitCustomShow: CustomShow.type

	scala> 3.show
	res5: List[Char] = List(M, y,  , S, h, o, w, :,  , 3)

Now that's pretty cool. We modified the behavior of a function without touching its defintion.

## The Order of Things

Some typeclasses are built on-top of other typeclasses. When we say this we mean a typeclass that adds behavior to a subset of types in another typeclass. `Order` is one such typeclass. All members of `Order` are members of the `Equal` typeclass. This is pretty easy to see from part of its defintion: `trait Order[-A] extends Equal[A]`.

That's cool and all but what is `Order`. Members of the `Order` typeclass support a single behavior: `(A, A) => scalaz.Ordering`. Given two instances of `A`, we return an instance of `scalaz.Ordering`. `scalaz.Ordering` has three pre-defined "constants", `LT`, `EQ`, `GT`. Basically, members of `Order` can be sorted. Just like before, although `scalaz.Ordering` sounds a lot like `scala.Ordering`, `Order` is a typeclass and `scala.Ordering` is just a polymorphic trait. In Scalaz, we use `?|?` to get the ordering of two instances of the same type. 

	scala> 1 ?|? 2
	res9: scalaz.Ordering = LT

	scala> 1 ?|? 1
	res10: scalaz.Ordering = EQ

	scala> 1 ?|? 0
	res11: scalaz.Ordering = GT

We can see how typeclasses generalize behavior by trying to order `LT`, `EQ`, `GT` themselves.

	scala> (LT : scalaz.Ordering) ?|? (GT: scalaz.Ordering)
	res14: scalaz.Ordering = LT

	scala> (EQ : scalaz.Ordering) ?|? (GT: scalaz.Ordering)
	res16: scalaz.Ordering = LT

	scala> (GT : scalaz.Ordering) ?|? (GT: scalaz.Ordering)
	res17: scalaz.Ordering = EQ

Low and behold `scalaz.Ordering` is a member of `Order`. Woah! `scala.Ordering` is also part of `Order`. Since we said earlier that members of `Order` are also members of `Equal` we should be able to compare `scalaz.Ordering`s for equality too.

	scala> (GT : scalaz.Ordering) === (LT : scalaz.Ordering)
	res18: Boolean = false

	scala> (GT : scalaz.Ordering) === (GT : scalaz.Ordering)
	res19: Boolean = true

	scala> (EQ : scalaz.Ordering) === (EQ : scalaz.Ordering)
	res20: Boolean = true

Step back and think about why this simple example makes our code better. We just reasoned about what's possible based on some simple properties. The `scalaz.Ordering` type is a member of `Order`, so we know (or can easily reason) that it must be a member of `Equal`. It has a small impact in this case but as we continue to use the library that impact will grow. 

## Adding the World (with Semigroups)

Ok so we've gotten our hands a little dirty -- or if you're still pimping with Xzhibit, a little greasy -- but now it's time to really dive in and get muddy. We'll talk about a cool typeclass, `Semigroup`. Although the heading above is hyperbole and Semigroups don't really let us add the world, they do let us add a ton of stuff. <sup>1</sup>

Members of `Semigroup` meet two mathematical properties.

### Rule 1: closure

The first rule you need to follow to be part of the semigroup typeclass is that you must be able to *append* two instances of your type and get a third instance of the same type back. This brings us to the first rule of semigroups:

	For all instances a, b of type A, append(a, b) gives us another A.

So if I append an `Int` and an `Int` or a `String` and `String` I get an `Int` or `String` respectively. This is called *closure*. Can I append two `Option`s? Well, we will talk about that in a moment. Lets just try 'em all first! 

In Scalaz, you can call append on an instance of a type that is a member of `Semigroup` giving it another instance using `|+|`. 

	scala> 1.some |+| 2.some
	res22: Option[Int] = Some(3)

	scala> 1.some |+| none
	res23: Option[Int] = Some(1)

	scala> 1 |+| 1
	res30: Int = 2

	scala> 2.0 |+| 3
	res31: Double = 5.0

	scala> "hello, " |+| "world"
	res32: java.lang.String = hello, world


### Rule 2: associativity

The second rule you need to be a member of `Semigroup` is called *associativity*. It adds onto the closure rule by saying that if you append a and b together, and then append the result of that to c, then the result has to be the same as if you append a to the result of appending b and c. This is the exact same associativity rule as you might remember from addition & multiplication in math. Formally, the property looks like this:

	Let a, b, c be any three instances of type A. append is associative on A if append(a, append(b, c)) === append(append(a, b), c)

Let's see if that works for `Option`s.

	scala> ((1.some |+| 2.some) |+| 3.some) === (1.some |+| (2.some |+| 3.some))
	res35: Boolean = true

// TODO: that's only one case prove with scalacheck/specs2?

Ok, so we've appended two `Option`s, got back an `Option` and proven that *associativity* holds as well. This is why `Option` is a member of `Semigroup` and we can use `|+|` with any instance of that type.

But wait, there's more! `Option` is not always a member of `Semigroup`. I know, I know, I just said it was a member and it is, *as long as its type parameter is a member of `Semigroup`*. Here's the code that enforces that rule:

	implicit def OptionSemigroup[A : Semigroup]: Semigroup[Option[A]] = ...

`OptionSemigroup` takes one type parameter `A` which is a `Semigroup`. This allows the compiler to tell us when we try to append to optional things that we actually can't append.

Also, If we try to use `|+|` on two instances of `case class MyClass(a: Int)`, defined earlier, we get an error.

	scala> MyClass(1) |+| MyClass(2)
	<console>:18: error: could not find implicit value for parameter s: scalaz.Semigroup[MyClass]
              MyClass(1) |+| MyClass(2)

This type of error is becoming pretty familiar. There is no definition of `Semigroup[MyClass]` and certainly not an implicit one in scope. We've just seen that `MyClass` is not a member of `Semigroup`. Based on that we know the same holds true for using `|+|` on instances of `Option[MyClass]`. Don't worry we will make `MyClass` a member soon enough.

	scala> some(MyClass(1)) |+| none
	<console>:22: error: could not find implicit value for parameter s: scalaz.Semigroup[Option[MyClass]]
              some(MyClass(1)) |+| none

You can append many other types inlcuding `List`, `Boolean`, `Tuple2`and `Either`. You can see the full list in the [definition of `Semigroup`](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Semigroup.scala). Here are some examples of the first two. 

	scala> true |+| false
	res0: Boolean = true

	scala> false |+| false
	res1: Boolean = false

	scala> true |+| true
	res2: Boolean = true

	scala> List(1, 2, 3) |+| List(4, 5, 6)
	res3: List[Int] = List(1, 2, 3, 4, 5, 6)

	scala> List(MyClass(1), MyClass(2)) |+| List(MyClass(3))
	res5: List[MyClass] = List(MyClass(1), MyClass(2), MyClass(3))

We'll talk about tuples soon, but first let's try appending two `Either`s. When we append two `Either`s, we actually are appeneding either its left projection or its right projection. Scala gives us the `left` and `right` methods on `Either` to get these projections. Scalaz also gives us the convience `left` and `right` methods on all types via `Identity`. When we append two `Either.LeftProjection`s if the first operand is `Left` we return it, if not we return the second operand regardless of whether its a `Left` or `Right`. The same holds for `Either.RightProjection` and `Right`.

	scala> 1.left.left |+| 2.left.left
	res1: Either.LeftProjection[Int,Nothing] = LeftProjection(Left(1))

	scala> 1.right.left |+| 2.left.left
	res2: Either.LeftProjection[Int,Int] = LeftProjection(Left(2))

	scala> 1.right.left |+| 2.right.left
	res3: Either.LeftProjection[Nothing,Int] = LeftProjection(Right(2))

	scala> 1.right.right |+| 2.right.right
	res4: Either.RightProjection[Nothing,Int] = RightProjection(Right(1))
	
	scala> 1.left.right |+| 2.right.right
	res5: Either.RightProjection[Int,Int] = RightProjection(Right(2))

	scala> 1.left.right |+| 2.left.right
	res6: Either.RightProjection[Int,Nothing] = RightProjection(Left(2))

This is very useful if you have a default `Either` you want to use when another `Either` you have may be of the incorrect projection. 

Let's get back to the `List` and `Boolean` examples from above, now. You have have noticed a couple things. For one, when we added two instances of `List[MyClass]` we didn't get an error about a missing implicit. This is because `append` for `Semigroup[List]` is not defined in the same fashion as `append` in `Semigroup[Option]`.<sup>2</sup> When you append two lists it doesn't matter their type, `|+|` acts like `:::` from the Scala standard library. 

The second thing you may have noticed is that `|+|` on `Boolean` is the same as `||` and that is the truth. But only a half-one. I've been lying to you a bit up until now (unless you've been checking the footnotes). `Semigroup` is actually a bit more general then I've lead to believe. Actually, its the `append` operation thats more general. In fact, I haven't really defined what `append` is -- except for somewhat letting you believe it meant "to add two things". If you checkout the [Wikipedia on Semigroup](http://en.wikipedia.org/wiki/Semigroup) it tells you straight away,

	"a semigroup is an algebraic structure consisting of a set and a binary operation"

The set is all instances of `A`, where `A` is the type parameter given to `Semigroup`. The binary operation is `append`. That's a very general definition. An operation that takes two operands (and must meet those properties from above of course). So this doesn't necessarily mean that `append` means "add". It's only rules are *taking two operands of the same type*, *returning an instance of the same type*, *being a closed operation* and *being associative*. It doesn't have to be addition! The careful reader may have begun to pick up on this in the `Either` example. There was no adding! Actually, `|+|` on `Either` worked more like it did for `Boolean` than it did on `List` or `Option`. 

This leads us to another possible conclusion. Is it possible that a type `A` can be a `Semigroup` under more than one such binary operation? Why, yes it is and `Boolean` is one of them! What other operation might there be? We know `||` is the binary operation `|+|` on `Boolean` uses by default. `&&` is similar. `Boolean` is closed under conjunction, `Boolean && Boolean` always returns `Boolean`, and conjunction is also associative, `((Boolean && Boolean) && Boolean) === (Boolean && (Boolean && Boolean))`. How do we use `&&` with `|+|` instead of `||`? Well, you kind of need to tell it to. Scalaz, defines a type for us called `BooleanConjunction` that is primed and ready for us to use with `|+|`. Actually, all `BooleanConjuction` does is wrap a boolean for conjunctions sake. When we call `|+|` passing it two `BooleanConjunction`s we are actually using the `appened` defined in `Semigroup[BooleanConjunction]` which in turn uses conjunction (of course!). We get a `BooleanConjunction` from a `Boolean` by calling `|∧|` on it. (the `∧` between the pipes is the unicode character `\u2227`, logical-and, not a caret). 

	scala> (true |∧|) |+| (false |∧|)
	res5: scalaz.BooleanConjunction = false

	// just for reminder's sake
	scala> true |+| false
	res6: Boolean = true

We've now seen `Semigroup` and its append do some pretty cool things. It's let us put a lot of things together with like things, we've changed the meaning of "put together" for those things and sometimes they are kind of recursive: when we put together two `Option`s it only worked if the underlying type was a member of `Semigroup` as well. It's not a far reach, especially since I mentioned it earlier, that we can append tuples as well. Tuples, like `Option` require the underlying types to be members of `Semigroup`. Also, since we can only append members of the same set we can append two `(String, Int)`s but not a `(String, Int)` and an `(Int, String)`. 

	scala> (1, 2) |+| (3, 4)
	res10: (Int, Int) = (4,6)

	scala> (1.some, 2) |+| (3.some, 4)
	res13: (Option[Int], Int) = (Some(4),6)

	scala> (1.some, 2) |+| (3.some, none)
			<console>:14: error: too many arguments for method |+|: (a: => (Option[Int], Int))(implicit s: 			scalaz.Semigroup[(Option[Int], Int)])(Option[Int], Int)
              (1.some, 2) |+| (3.some, none)


Scalaz defines tuple members of `Semigroup` for sizes two to four. You can define more if you really want.

I think its much more interesting to define our own member of Semigroup. Before we do, we need a way to prove that our member meets the rules to be a member. Technically, we can make anything a member of Scalaz's `Semigroup` typeclass, but that doesn't actually make it a semigroup. To check the closure condition we will cheat a bit and rely on the type system and the defintion of append. To check associativity we're going to use two awesome Scala testing libraries that work even better together, [Specs2](http://etorreborre.github.com/specs2/) & [ScalaCheck](http://code.google.com/p/scalacheck/). We'll start with a simple shell of the specification.

	class SemigroupSpec extends Specification with ScalaCheck { def is = 
	
		"A semigroup is associative" ! checkAssoc

		def checkAssoc = //…

	}

If you are not familiar with Specs2 or ScalaCheck I highly recommend checking out the documentation. Quickly, what we've done here is outlined a single Specs2 example that will check if our soon to be member of semigroup is associative. We haven't actually said how we will check that property, but we will do so with ScalaCheck. Let's assume we first want to make `case class MyClass(i: Int)` from earlier our new member. We can define our associativity check as:

	def checkAssoc = check {
		(a: Int, b: Int,  c: Int) => 
			((MyClass(a) |+| MyClass(b)) |+| MyClass(c)) must beEqualTo((MyClass(a) |+| (MyClass(b) |+| MyClass(c))))
	}

This will generate three instances of Int several times over. Each time, we use Specs2 matchers to show that appending instances of `MyClass` is associative. If this holds for all runs the test passes. 

To make `MyClass` a member of `Semigroup` we'll do something very similar to what we did for `Equal`. Once again, Scalaz gives us a convenient builder function, this time via the trait `Semigroups` (noticing a pattern?).

	object MySemigroup extends scalaz.Semigroups {
  		implicit def MyClassSemigroup: Semigroup[MyClass] = semigroup((a, b) => MyClass(a.i + b.i))
	}

The `semigroup` builder takes a function of two parameters of type `A`, returning an `A`. This is your `append` function. If you add `import MySemigroup._` to the spec body and run it, it will pass.<sup>3</sup> We know this because addition, which is all we are really doing is associative as well. 

How might we incorrectly make `MyClass` a member of `Semigroup`. Well we know subtraction is not associative. Does your spec pass if your define `MyClassSemigroup` like below?

	semigroup((a, b) => MyClass(a.i - b.i))

Mine doesn't. Can you find one or two other ways we could properly define `MyClass` as a member of `Semigroup`?

Ok, that was pretty straight-forward. Let's move to a slightly more complex example. What if we instead define `MyClass` with a type parameter like:

	case class MyClass[T](a: Option[T])

We can quickly reason that we can make this a semigroup as long as `T` is a member of `Semigroup`. Let's rewrite our check function first:

	def checkAssoc = check {
		(a: Option[Int], b: Option[Int],  c: Option[Int]) => 
		((MyClass(a) |+| MyClass(b)) |+| MyClass(c)) must beEqualTo((MyClass(a) |+| (MyClass(b) |+| MyClass(c))))
	}

Our test is a bit limited, we only choose one type we know to be a member of the typeclass, but it will suffice for our purposes. We can then define our new implicit conversion:

	implicit def MyClassSemigroup[T : Semigroup]: Semigroup[MyClass[T]] = semigroup((a, b) => MyClass(a.a |+| b.a))

The defintion limits the conversion to types `T` which are themselves a `Semigroup`. If we don't have a semigroup for a `T` then we cannot append two `MyClass[T]` instances. We see this because if we run our specification it fails. However, if we try to append two instances of `MyClass[(Int, Int) => Int]` we get a familiar error:

	scala> MyClass(((a: Int, b: Int) => a).some) |+| MyClass(((a: Int, b: Int) => b).some)
	<console>:20: error: could not find implicit value for parameter s: \
						 scalaz.Semigroup[MyClass[(Int, Int) => Int]]
              MyClass(((a: Int, b: Int) => a).some) |+| MyClass(((a: Int, b: Int) => b).some)


## Putting it Together

When we use type classes together their power becomes even more evident. In [this gist](https://gist.github.com/1342196) Jason Zaugg, one of the Scalaz committers, shows how we can make `Order[A]` a member of `Semigroup`. By doing so, we make it easy to build compound orderings from two `Order[A]`s. In the example, the `Person` class (shown below) is what we want to order.

	case class Person(name: String, age: Int)

We can easily define a way to order two `Person` instances by name or age using the `orderBy` convenience method given to us by Scalaz.

	val byName = orderBy((_: Person).name)
	val byAge = orderBy((_: Person).age)

Using a `Semigroup[Order[A]]` we could append the two orderings so that we order first by age then by name. 

	val byAgeThenName = byAge |+| byName
	byAgeThenName.order(Person("bob", 12), Person("bob", 13)) // scalaz.Ordering = LT

Let's take a look at how `Semigroup[Order[A]]` is defined:

	implicit def OrderSemigroup[A]: Semigroup[Order[A]] = 
		semigroup((o1, o2) => order((a1: A, a2: A) => o1.order(a1, a2) |+| o2.order(a1, a2)))

The definition uses the fact that `Ordering` is also a semigroup. Because we are appending two `Order`s we should get another `Order` back, which is exactly what using the `order` convenience method does. `order` takes a function that returns an `Ordering` given two `A`s and uses the function to build an `Order[A]`. In this case the function we pass to `order` uses the first `Order[A]` operand (`o1`) passed to `|+|` to get the `Ordering` for the two `A`s, then appends it to the `Ordering` returned using the second `Order[A]` (`o2`).
 
## So That's The Basics

We've now seen several of the simpler typeclasses and I hope some of their advantages are becoming clear. One thing that I hope is abundantly clear is that typeclasses are highly general. This is what makes them a bit hard to reason about at first, but its also why they are such a powerful construct. One example of this is how typeclasses improve Java-Scala interop. When working with Java collections, for example, we cannot use operators like `:::` without first importing `scala.collections.JavaConversions._`. Sometimes this boxing between Java and Scala types can be expensive and unnecessary -- often causing conversion when using the underlying Java type would really suffice if it were only possible to easily append two such objects. Java types can be members of type classes as well as long as they meet the type class' properties. `LinkedList`, `PriorityQueue` and `CopyOnWriteArrayList` are just a few of the Java types that are members of `Semigroup`. We can append these members using `|+|` without the conversion, which in some cases may not even exist. In later sections we will learn about other type classes that will allow us to do things like map and fold over types including Java ones. 


<sup>1</sup> In mathematics, Semigroups are actually a much more general concept. We are talking about their use in Scalaz so we will not cover it that in depth but you may find it interesting to start with the ever so general defintion on [Wikipedia](http://en.wikipedia.org/wiki/Semigroup).

<sup>2</sup> In reality, Scalaz does not define a `Semigroup[List]` for us, but a more general one on things that are traversable. You can get the nitty gritty by reading the [implementation](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Semigroup.scala).

<sup>3</sup> If you are using SBT to follow along you can run the spec inside the SBT console by calling `test`. Please refer to the [Specs2 Runners Guide](http://etorreborre.github.com/specs2/guide/org.specs2.guide.Runners.html#Runners+guide) for more info on how to run tests in your environment, otherwise. 
