# Understanding Some Simple Typeclasses

Now that we have a better understanding of what Scalaz gives us let's explore some of the simpler typeclasses it defines. We'll revist `Equal` and discuss `Show` and `Order`. For flavor, we will also discuss `Semigroup` which is one of those typeclasses that apply to types meeting a couple of mathematical properties. 

As you read through the details of these typeclasses it may seem like some of the behavior they define overlaps with existing features of Scala. It's true that some portions of Scalaz overlap with what is given to us in the standard library. Scala is a very functional language as-is and Scalaz is a purely functional library, so that overlap makes sense. It's important to remember that Scalaz and typeclasses represent a different way of thinking about your code. Also, we mentioned in the last section Scalaz gives us some functional data structures that don't exist in the standard library. The biggest idea that I hope will continue to become more evident is that Scalaz usually gives a more general version of a behavior that we can use on a wider range of types.

The topics we will discuss in this section are also foundational topics. They will give us a better understanding of how typeclasses work. Some of them are typeclasses that are used to build other typeclasses. For example, we will talk about `Order` of which all members are also members of `Equal`. The `Monoid` typeclass which we will talk about later is built on top of the `Semigroup` typeclass.

## Typing our Safe Equals

Before, we talked about `Equal[-A]` to better understand what typeclasses are. Let's revist it one more time briefly and learn some finer details about it. To get our hands dirty we will also make our own class a member. We won't always do this when touring different typeclasses, but `Equal` is relatively straight-forward and we want typesafe equals for our types as well. 

Members of the `Equal` typeclass includes all of the types you'd expect including `Int`, `Byte`, and `String` as well as ones you may or may not have expected like `xml.NodeSeq`. For a full list take a look at the [definition](https://github.com/scalaz/scalaz/blob/master/core/src/main/scala/scalaz/Equal.scala). As we know this typeclass gives us the `===` operator for all member types. 

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

Ok so we've gotten our hands a little dirty -- or if your still pimping with Xzhibit, a little greasy -- but now it's time to really dive in and get muddy. We'll talk about a cool typeclass, `Semigroup`. Although the heading above is hyperbole and Semigroups don't really let us add the world, they do let us add a ton of stuff. <sup>1</sup>

Members of `Semigroup` meet two mathematical properties.

### Rule 1: closure

The first rule you need to follow to be part of the semigroup typeclass is that you must be able to *append* two instances of your type and get a third instance of the same type back. This brings us to the first rule of semigroups:

	For all instances a, b of type A, append(a, b) gives us another A.

So if I append an `Int` and an `Int` or a `String` and `String` I get an `Int` or `String` repsepectively. This is called *closure*. Can I append two `Option`s? Well, we will talk about that in a moment. Lets just try 'em all first! 

In Scalaz, you can call append on an instance of a type that is a member of `Semigroup` giving it another instance using `|+|`. 

	scala> Option(1) |+| Option(2)
	res22: Option[Int] = Some(3)

	scala> Option(1) |+| none
	res23: Option[Int] = Some(1)

	scala> 1 |+| 1
	res30: Int = 2

	scala> 2.0 |+| 3
	res31: Double = 5.0

	scala> "hello, " |+| "world"
	res32: java.lang.String = hello, world


### Rule 2: associativity

The second rule you need to be a member of `Semigroup` is called *associativity*. It adds onto the closure rule by saying that if you append a and b together, and then append the result of that to c, then the result has to be the same as if you append a to the result of appending b and c. This is the exact same associativity rule as you might remember from addition & subtraction in math. Formally, the property looks like this:

	Let a, b, c be any three instances of type A. append is associative on A if append(a, append(b, c)) === append(append(a, b), c)

Let's see if that works for `Option`s.

	scala> ((some(1) |+| some(2)) |+| some(3)) === (some(1) |+| (some(2) |+| some(3)))
	res35: Boolean = true

// TODO: that's only one case prove with scalacheck/specs2?

Ok, so we've appened two `Option`s, got back an `Option` and proven that *associativity* holds as well. This is why `Option` is a member of `Semigroup` and we can use `|+|` with any instance of that type.

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

We'll talk about tuples soon, but first let's try appending two `Either`s. When we append two `Either`s, we actually are appeneding either its left projection or its right projection. Scala gives use the `left` and `right` methods on `Either` to get these projections. Scalaz also gives us the convience `left` and `right` methods on all types via `Identity`. When we append two `Either.LeftProjection`s if the first operand is `Left` we return it, if not we return the second operand regardless of whether its a `Left` or `Right`. The same holds for `Either.RightProjection` and `Right`.

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

// return to booleans, || is default operation, using |^| (logical and in pipes) to do &&
// return to lists, why we can |+| on a List[MyClass]
// implement MyClass in Semigroup



<sup>1</sup> In mathematics, Semigroups are actually a much more general concept. We are talking about their use in Scalaz so we will not cover it that in depth but you may find it interesting to start with the ever so general defintion on [Wikipedia](http://en.wikipedia.org/wiki/Semigroup).