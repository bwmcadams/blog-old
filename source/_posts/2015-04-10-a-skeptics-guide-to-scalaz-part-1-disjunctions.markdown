---
layout: post
title: "A Skeptic's Guide to scalaz' Gateway Drugs: Part 1 - Disjunctions"
date: 2015-04-10 18:41:37 -0400
comments: true
categories: 
  - scalaz
  - scala
  - functionalprogramming
---
*(This is Part 1 of a series of distillations of a presentation I've been giving for the last year, ["A Skeptic's Guide to scalaz' Gateway Drugs"](http://slides.com/bwmcadams/scalaz-gateway-drugs). It is meant to provide an introduction to the core functionality of scalaz that a developer might find most useful, without going off the deep end.)*

What is `scalaz` exactly? Well, at its core, `scalaz` is a functional programming library for Scala. It is intended to bring more functional programming concepts from languages like Haskell into Scala.

Until recently, I've looked at [scalaz](https://github.com/scalaz/scalaz) rather skeptically. As a self taught developer without any college or advanced mathematics training, I felt intimidated by what I saw. Long before I exited the ranks of rookie Scala programmer, I dabbled in Haskell a bit but left more confused than I started at. 

![](/images/clockwork-eyes.gif)

So, tools like `scalaz` were a bit scary to me. Plus, "what's wrong with the Scala standard library?". As it turns out, a lot. Let's look first at Scala's Either, and where we can improve upon it.
<!--more-->
Scala's builtin [Either](http://www.scala-lang.org/api/current/#scala.util.Either) is a common construct used to indicate one of two conditions: Success or Error. These are, by convention, `Left` for Errors, and `Right` for Success. The concept is good, but there are some limits to interaction. By default, we cannot use it in a for comprehension:

```scala
scala> val success = Right("Success!")
success: scala.util.Right[Nothing,String] = Right(Success!)

scala> success.isRight
res2: Boolean = true

scala> success.isLeft
res3: Boolean = false

scala> for {
       |   x <- success
       | } yield x
       <console>:10: error: value map is not a member of scala.util.Right[Nothing,String]
                      x <- success
                           ^
``` 

*(Note, there is a function called `rightProjection` which converts an `Either` into something comprehendable)*

In the `scalaz` world, there is a similar construct to `Either`, known as `\/`. If `\/` reads like a mouthful, you can call it Disjunction - which we'll do in this post as well. A Disjunction can, like in `Either` have either a Left or a Right side - typically representative of a Success (right) or an Error (left). These are represented by symbols: `-\/` for Left, `\/-` for Right. 

*(I find, in general, that the easiest memory trick is to look at which side of the Disjunction the `-` appears on.)*

With Disjunctions, there is an assumption that we prefer Success (the right, or `\/-`) - this is also known as Right Bias. With Right Bias, for comprehensions, `map`, and `flatMap` unpack for us where "success" (`\/-`) continues and "failure" (`-\/`) aborts.

When declaring a return type of a Disjunction, you should generally prefer to use the Infix notation for clarity sake. That is to say:

```scala
def query(arg: String): Error \/ Success
```

with the Infix notation for the return type, is preferable to this:

```scala
def query(arg: String): \/[Error, Success]
```

the standard notation, which may not read as clearly.

As for declaring instances of Left or Right, there are a few options in `scalaz`. The first is postfix operators, `.left` and `.right`, which wrap an existing value in a Disjunction instance:

```scala
import scalaz._
import Scalaz._

scala> "Success!".right
res7: scalaz.\/[Nothing,String] = \/-(Success!)

scala> "Failure!".left
res8: scalaz.\/[String,Nothing] = -\/(Failure!)
```

Alternately, we can use the Disjunction singleton instance instead:

```scala
import scalaz._
import Scalaz._

scala> \/.left("Failure!")
res10: scalaz.\/[String,Nothing] = -\/(Failure!)


scala> \/.right("Success!")
res12: scalaz.\/[Nothing,String] = \/-(Success!)
```

which, by virtue of being more explicit, may be clearer to the reader.

Finally, we can construct instances of Left and Right directly:

```scala
import scalaz._
import Scalaz._

scala> -\/("Failure!")
res9: scalaz.-\/[String] = -\/(Failure!)

scala> \/-("Success!")
res11: scalaz.\/-[String] = \/-(Success!)
```

Remember I talked about how we could comprehend over Disjunctions, and success continued while failure aborted?

![Here's Johnny?](/images/shining-grinning.gif)

Let's look at what that looks like with some sample data:

```scala

import scalaz._
import Scalaz._

val success1 = \/.right("This succeeded")

val success2 = \/.right("This succeeded also!")

val fail1 = \/.left("This failed miserably...")

val fail2 = \/.left("Oops :(")

```

We now have a couple of Disjunction instances we can use. If we comprehend over only Right, we get back an instance of Right.

```scala
for {
  one <- success1
  two <- success2 
} yield (one, two)
/* res0: scalaz.\/[Nothing,(String, String)] = 
      \/-((This succeeded,This succeeded also!)) */
```

Since both instances we used were Right, we get back an instance of Right ( `\/-` ).

What if we include a Left in there?

```scala
for {
  one <- success1
  two <- fail1  
  three <- success2
} yield (one, three)
/* res1: scalaz.\/[String,(String, String)] = 
      -\/(This failed miserably...) */
``` 
The behavior here is much like with an `Option` instance of `None` (Which we'll talk about in the next post). When Scala encounters a "failure" - in this case a Left ( `-\/` ) - it aborts the iteration and returns an instance of the failure. This case means the `yield` never gets run. This can be valuable as an easy way to work with multiple Disjunctions all of which needed to be Right.

Now you've had a (hopefully) gentle introduction to the world of `\/`. In [the next part](http://bytes.codes/2015/04/13/a-skeptics-guide-to-scalaz-gateway-drugs-part-2-options-with-disjunction/), we'll talk about using Scala's `Option` in conjunction with Disjunctions to better handle failure conditions with existing code.
