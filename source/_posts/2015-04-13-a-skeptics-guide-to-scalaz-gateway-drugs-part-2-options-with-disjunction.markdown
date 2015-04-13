---
layout: post
title: "A Skeptic's Guide to Scalaz' Gateway Drugs: Part 2 - Options with Disjunction"
date: 2015-04-13 12:14:44 -0700
comments: true
categories: 
  - scalaz
  - scala
  - functionalprogramming
---
*(This is Part 2 of a series of distillations of a presentation I've been giving for the last year, ["A Skeptic's Guide to scalaz' Gateway Drugs"](http://slides.com/bwmcadams/scalaz-gateway-drugs). It is meant to provide an introduction to the core functionality of scalaz that a developer might find most useful, without going off the deep end. Previous entries include [Part 1 -- Disjunctions](/2015/04/10/a-skeptics-guide-to-scalaz-part-1-disjunctions/))*

Welcome back to the Skeptic's Guide to `scalaz`. In [the last part](/2015/04/10/a-skeptics-guide-to-scalaz-part-1-disjunctions/) of this series, we introduced you to the power of `scalaz` Disjunctions---also known as `\/`---and how we can use them to indicate a return value of *either* an Error or a Success. As a reminder, convention dictates that Left---`-\/`---is an error, while Right---`\/-`---is success. 

![Hello? scalaz?](/images/buck-phone.gif)

In this part, we'll discuss interactions with Scala's `Option`. Specifically, I want to discuss how to manage "stacks" of `Option` in for comprehensions, and how to use Disjunction to manage them.

<!--more-->

In Scala, `Option` is a container commonly used to indicate a return type that can have no value. `Option` has two subtypes: `Some[T]`---which contains a value of type `T`---and `None`, which contains no value. We use these in the Scala world to avoid the sins of `null`; Because `None` is a valid object, invoking functions on it doesn't cause the dreaded `NullPointerException`.

Similar to `scalaz` Disjunctions, there is a "Right" bias on `Option`. Specifically, it is biased towards `Some[T]`, and when we comprehend over `Some[T]` the loop continues:

```scala 
val some1 = Some("This is a value.")

val some2 = Some("This is also a value.")

val some3 = Some("You guessed it. A value")


for {
  one <- some1
  two <- some2
  three <- some3 
} yield (one, two, three)
/* res2: Option[(String, String, String)] = 
    Some((This is a value.,This is also a value.,You guessed it. A value)) */

```

As I said, `Option` has a bias towards `Some`. Each step of the comprehension here unpacks a value from `Some`. But what if there's a `None` thrown in there?

```scala

for {
  one <- some1
  two <- None
  three <- some3
} yield (one, two, three)
/* res3: Option[(String, Nothing, String)] = None */

```

What went wrong? In short, the same behavior as we saw when we threw a Left Disjunction into a comprehension. When we encounter a `None`, the loop aborts and returns the failure value. For a deeper look at what I mean---and how to fix it---let's construct some more concrete sample data. 
 
```scala
case class Address(city: String)

case class User(first: String, 
                last: String, 
                address: Option[Address])

case class DBObject(id: Long, 
                    user: Option[User])

val brendan = 
  Some(DBObject(1, Some(User("Brendan", "McAdams", None))))

val someOtherGuy = 
  Some(DBObject(2, None))
```

Here is a set of constructs that will let us represent a user & address in our database. Note that both `User` and `Address` are optional on their respective containers. I've seen a lot of code that works this way: "If the database failed to return a row, let's return `None`". Here's what it looks like in practice when one of those row retrievals fails...

```scala
for {
  dao <- brendan
  user <- dao.user
} yield user

/* res4: Option[User] = Some(User(Brendan,McAdams,None)) */

```

In our first example, `brendan` is a `DBObject` with a valid `User`. When we comprehend over just the `DBObject` and `User`, we get back a valid `Some`. But what if we try to extract both the `User` and `Address` from `someOtherGuy`?

```scala
for {
  dao <- someOtherGuy
  user <- dao.user
  address <- user.address
} yield address
/* res5: Option[Address] = None */

```

Now, if we were retrieving the data from the database we've run up against a very interesting question. Was there no `User`? Or was there no `Address`? This is the problem I ran into a *lot* with returning `Option` from the database.

![Boom!](/images/majorKong.gif)

Fundamentally, comprehending over groups of `Option` leads to "silent failure". Luckily, `scalaz` includes some implicits to convert an `Option` to a Disjunction. Since Disjunction's right bias makes it easy to comprehend, we can do the conversion in place without rewriting a lot of code. For a Left, we'll still get useful information in place of `None`.

```scala
None.toRightDisjunction("No object found")
/* res6: scalaz.\/[String,Nothing] = -\/(No object found) */
```

Here, we call the implicit function `toRightDisjunction` upon an instance of `Option` (`None`, in this case). Specifically, `toRightDisjunction` says "Convert an `Option` to a disjunction where `Some[T]` becomes `\/-(T)`---Right---and `None` becomes `-\/(<argument>)`", or Left. That last bit is important: the argument to `toRightDisjunction` is used to create a value for a Left Disjunction.

For those who prefer 'concise' over 'explicit', there is also a symbolic version of `toRightDisjunction`, which is functionally identical:

```scala
None \/> "No object found"
/* res7: scalaz.\/[String,Nothing] = -\/(No object found) */
```

So, when there's a `None` we use the argument to `toRightDisjunction` to create a `-\/`, but if there's a `Some` we convert the value from `Some[T]` to `\/-[T]`. Here's what it looks like with `Some` values:


```scala
Some("My Hovercraft Is Full of Eels") \/> "No object found"
/* res8: scalaz.\/[String, String] = \/-(My Hovercraft Is Full of Eels) */

Some("I Will Not Buy This Record It Is Scratched")
  .toRightDisjunction("No object found")
/* res9: scalaz.\/[String, String] = 
  \/-(I Will Not Buy This Record, It Is Scratched") */
```

Given these new tools, let's look at that user/address extraction again.

```scala
for {
  dao <- brendan \/> "No user by that ID"
  user <- dao.user \/> "Join failed: no user object"
} yield user
/* res10: scalaz.\/[String,User] = \/-(User(Brendan,McAdams,None)) */

for {
  dao <- someOtherGuy \/> "No user by that ID"
  user <- dao.user \/> "Join failed: no user object"
  address <- user.address \/> "Join failed: No address on user"
} yield address
/* res11: scalaz.\/[String,Address] = -\/(Join failed: no user object) */
```

Hey, look at that! On our second comprehension, we got some useful information back about what went wrong. Now we can log that, return it to the frontend, or whatever else it is you do with failure data.

What if we want to do something beyond comprehensions? Stay tuned for our next episode, where we'll talk about `Validation`, and how to use it to check multiple error conditions.
