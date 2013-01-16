---
title: Understanding Scala's Type Classes
tags: scala effectivescala typeclasses polymorphism
layout: post
---
Over the last year or so, I have found myself making more and more use of Scala's Type Class system to add flexibility to my code.  This is especially evident in the MongoDB Scala Driver, [http://github.com/mongodb/casbah](Casbah), where the most recent work has been to simplify many features by migrating them to type classes. 

During this work however, I've found during that many otherwise adroit Scala engineers seem befuddled or daunted by the Type Class. It does me no good to take advantage of clever features that my users don't understand, and many will benefit from introducing these concepts to their own code. So let's take a look at what type classes are, as well as how & why we can utilize them.

Wikipedia defines a Type Class as *"... a type system construct that supports ad-hoc polymorphism. This is achieved by adding constraints to type variables in parametrically polymorphic types"*. Admittedly, a bit of a mouthful -- and not very helpful to those of us who are self taught and lack the benefit of a comprehensive academic Computer Science education (myself included). Surely, there must be a way to simplify this concept?

In evaluating these ideas, I've found it easiest to think of a Type Class (in Scala, at least) as a special kind of *adapter*, which can impart additional capabilities upon a given type or set of types. In Scala the Type Class is communicated through *implicits*, and imparts one, or both, of two behaviors. First, a Type Class can be to utilized to *filter* what types are valid for a given method call (which I detailed in [this earlier post](/2011/07/13/User_Configgable_Type_Filtering_with_Type_Classes/)). Second, a Type Class can impart additional features and behaviors upon a type at method invocation time. This latter is much along the lines of an enhanced kind of composition, rather than the weaker inheritance which often plagues similar behaviors in a language like Java.

To better understand what I am describing, let's compare a few concepts around the creation and interaction of custom domain objects. I have several sets of tasks I have had to accomplish in Scala in the past -- and Scala solutions show some elegant Type Class oriented approaches which are rooted in the Standard Library. While this may seem a bit contrived, it is exactly the kind of problem through which *I* initially came to understand Type Classes –– and is thus an ideal lesson.

<!--more-->
First, let's take a look at *sorting* and custom objects to best understand how one accomplishes this. It is not an uncommon task in development for us to create our own objects and need to integrate them into Standard Library behaviors, such as sorting. Let's work with a few sample objects in the form of "Bank Accounts" to look at how this all work (and I'm aware of the poor concurrency control, etc. around balance -- this is a contrived example). Here's our Bank Account object: 

{% highlight scala %}
class BankAccount(val accountNumber: Long, val holderFirst: String,
                  val holderMiddle: Option[String], val holderLast: String,
                  var balance: Double) {

  def holderName = 
    "%s, %s %s".format(holderLast, holderFirst, holderMiddle.getOrElse(""))

  override def toString: String = 
    "{ Acct # %s, Held by %s with a balance of $%8.2f}".format(
        accountNumber, holderName, balance
    )
                              
}
{% endhighlight %}

We can easily populate collections with instances of these accounts as well.


{% highlight scala %}

val accounts = List(new BankAccount(1000893, "Brendan", Some("W."), "McAdams",
                                    1234.56),
                    new BankAccount(1000256, "John", None, "Smith", 
                                    10000291.83),
                    new BankAccount(1000012, "Jane", None, "Doe", 
                                    45.28),
                    new BankAccount(4002158, "Alan", None, "Smithee", 
                                    834567.00))

println("Bank Accounts: " + accounts)

{% endhighlight %}

Given collections of instances of these bank accounts in each language, we'd like to easily sort them –– given an arbitrary set of requirements.  Now, neither Scala or Java can "automatically" figure out how to sort these, instead requiring assistance from us (the developer).

{% highlight scala %}

val sortedAccounts = accounts.sorted

println("* Sorted Accounts: " + sortedAccounts)
{% endhighlight %}

It is unfortunately not *quite* that easy, as the above code will fail to compile asking for a missing argument:

{% highlight scala %}
BankAccount.scala:39: error: No implicit Ordering defined for this.BankAccount.
val sortedAccounts = accounts.sorted
                                    ^
one error found
{% endhighlight %}

Like in Java, we need to provide Scala information about how to sort a class of type `BankAccount`.  In Java however, we would need to use inheritance and actually change the structure of `BankAccount` by implementing the `Comparable` interface.  Personally, I've never been a fan of that approach -- changing a class directly can lead to behavioral oddities. It also has two major limitations that I've run into in the past. 

First we get locked into only *one* way to sort a `BankAccount`. If initially we want to sort by `accountNumber`, and code that in we are restricted should another part of our application need to sort by `balance`. We either work around the builtin sort methods or subclass, introducing more complications.

Second, we are severely restricted in our ability to handle this with a third party class. What if `BankAccount` is a vendor supplied class and is `final` so we cannot even create an extended version which implements comparable? Suddenly we are restricted from taking advantage of the sort routines built into the standard library and have to reinvent our own. Not ideal.


Instead, with Scala, the implementation of our `Comparable` equivalent is done externally in a Type Class of type `scala.math.Ordering`. When implemented, our instance of `Ordering` will both control what classes can be sorted as well as providing information about how to sort. But because it is implemented externally and provided as an implicit we can provide multiple versions should we need different sorting behaviors down the line\*. 


It is important to note that a Type Class in Scala is typically *stateless*. It is provided to callers as a single static instance based on Type, and only infers necessary state information from *instances of the referenced type* passed to its methods. The Type Class is controlling how instances of a given type should behave generically and should be side effect free. 

The normal way of providing a typeclass is to create a static implicit object of a trait implementation of `TypeClass[A]` in scope. I prefer to declare the base trait separately from the implicit object to encourage easier reusage.

For providing [Ordering\[BankAccount\]](http://www.scala-lang.org/archives/downloads/distrib/files/nightly/docs/library/scala/math/Ordering.html), we need to implement an abstract method `def compare(x: T, y: T): Int` which compares two instances of `T` (Where, in this case, `T` represents `BankAccount`) and returns an `Int` signifying their order against one another. Negative represents that `x < y`, positive that `x > y` and zero if `x == y`.

Let's take a look at how our `Ordering` instance for sorting a `BankAccount` by `accountNumber` might look.

{% highlight scala %}
trait BankAccountNumberOrder extends scala.math.Ordering[BankAccount] {
  def compare(x: BankAccount, y: BankAccount): Int = 
    if (x.accountNumber < y.accountNumber) 
      -1
    else if (x.accountNumber > y.accountNumber) 
      1
    else
      0
  
}

implicit object BankAccountNumberSort extends BankAccountNumberOrder
{% endhighlight %}

Now with an implicit instance of `Ordering[BankAccount]` in scope, our sort can succeed. Running our code should produce expected results:

{% highlight scala %}
Bank Accounts: List({ Acct # 1000893, Held by McAdams, Brendan W. with a balance of $ 1234.56}, { Acct # 1000256, Held by Smith, John  with a balance of $10000291.83}, { Acct # 1000012, Held by Doe, Jane  with a balance of $   45.28}, { Acct # 4002158, Held by Smithee, Alan  with a balance of $834567.00})

Sorted Accounts: List({ Acct # 1000012, Held by Doe, Jane  with a balance of $   45.28}, { Acct # 1000256, Held by Smith, John  with a balance of $10000291.83}, { Acct # 1000893, Held by McAdams, Brendan W. with a balance of $ 1234.56}, { Acct # 4002158, Held by Smithee, Alan  with a balance of $834567.00})
{% endhighlight %}

The big benefit here (in my eyes) is that we didn't need to modify our `BankAccount` class at all to provide this behavior. *Even if `BankAccount` was a sealed third party class* we can provide sorting information for it. This is far superior to an inheritance based solution such as Java's. And if we wanted later to sort by `balance` instead of `accountNumber` we can explicitly pass a different instance to sort:

{% highlight scala %}
object BankAccountBalanceOrder extends scala.math.Ordering[BankAccount] {
  def compare(x: BankAccount, y: BankAccount): Int = 
    if (x.balance < y.balance) 
      -1
    else if (x.balance > y.balance) 
      1
    else
      0
}

val sortedByBalance = accounts.sorted(BankAccountBalanceOrder)

println("$ Sorted By Balance: " + sortedByBalance)
{% endhighlight %}

Complete control is passed to us from an externally controlled system. I'll save the details for a future post, but we can even use a type class to define what it means if I say `brendansAccount - johnsAccount` using an instance of `scala.math.Numeric[BankAccount]`.

Now go forth and Type with Class. 

\* [Daniel Spiewak](http://twitter.com/djspiewak) points out that Sun realized this complication as well a few Java releases back and introduced [Comparator](http://docs.oracle.com/javase/6/docs/api/java/util/Comparator.html), which is very similar to this Type Class approach.
