---
title: User Configurable Type Filtering with Scala Type Classes
tags: tech mongodb scala mongodb typeclasses stupidpettricks
layout: post
---
When I woke up this morning and looked through my twitter mentions, I found this gem sitting there from the middle of the night:

![@rit when using "_id" $lt new ObjectId(timestamp) it throws ValidDateOrNumericType, but we might want to select records after id timestamp (from @justthor)](/images/oid_lt_casbah.png)

The user in question is complaining that when using [Casbah's DSL](http://github.com/mongodb/casbah), it doesn't allow a MongoDB ``ObjectId`` as a valid type to the ``$lt`` operator.  But as [@justthor](http://twitter.com/justthor) points out, it is entirely possible to use ``ObjectId`` with the ``$lt`` operator since it contains timestamp information (See the [documentation for ObjectId](http://www.mongodb.org/display/DOCS/Object+IDs) if you want nitty gritty detail).   When I wrote the code for ``$lt`` however, I needed to decide what types were valid and weren't valid; I can't exactly guarantee type safety wih a DSL like Casbah's, but I can enforce type *sanity*.  Whether I forgot that you can use ``ObjectId`` in ``$lt`` or just decided that most people wouldn't need to is irrelevant --- I had in this case blocked a user from accomplishing something valid that they needed to.

It is a more than reasonable problem, and my initial reaction was "oh crap, I guess I need to patch that".  But what I forgot is that a few releases back, I rearchitected Casbah to obviate this kind of problem.  Casbah now allows for a user definable (or, if you prefer, "adjustable") type filter on any of its DSL operators.  This is accomplished through a very simple application of Scala Type Classes, a term which gets batted around a lot in the Scala community, but few seem able to understand or articulate its meaning to us lesser mortals.  Over the last few months I've come to understand Type Classes much more deeply than I think I ever expected, and applied these lessons to the design of my code.  As I failed to document the power and usage of these features at the time, I am going to be writing some additional detailed articles about *my* understanding of Type Classes in the next few weeks, and this is the first of such explanations.  

So the question at hand is, how exactly does Casbah allow us to do this magical type filtering that I just mentioned, without patching the driver or creating a new release?  First, let's look at how Casbah used to do things before the introduction of the as-yet unexplained Type Class introduction.

{% highlight scala %}
/**
 * Trait to provide the $lt (Less Than) method on appropriate callers.
 *
 * Targets (takes a right-hand value of) String, AnyVal (see Scala docs but basically Int, Long, Char, Byte, etc)
 * DBObject and Map[String, Any].
 *
 *
 * @author Brendan W. McAdams <brendan@10gen.com>
 * @see http://www.mongodb.org/display/DOCS/Advanced+Queries#AdvancedQueries-%3C%2C%3C%3D%2C%3E%2C%3E%3D
 */
trait LessThanOp extends QueryOperator {
  private val oper = "$lt" 

  def $lt(target: String) = op(oper, target)
  def $lt(target: java.util.Date) = op(oper, target)
  def $lt(target: AnyVal) = op(oper, target)
  def $lt(target: DBObject) = op(oper, target)
  def $lt(target: Map[String, Any]) = op(oper, target.asDBObject)
}
{% endhighlight %}

The ``op`` method is just a helper function which assists the DSL in constructing a valid MongoDB query under the covers; it's not particularly relevant to this discussion so let's leave it aside for now.  What *is* important is to note that this is a very naive approach to the DSL game --- it is neither type safe or type sane.  It has some very basic support for handling "special" types like ``String``, ``java.util.Date``, ``DBObject`` and ``Map[String, Any]`` but blindly allows anything which is ``AnyVal`` through.  There are, in my opinion, three problems here:

    * Is blindly passing ``AnyVal`` a good idea? Typically it consists of things like ``Int``, ``Char``, ``Boolean``, etc but can we be certain this is a safe assumption?
    * What if a user defines a custom type which they serialize to MongoDB as something which supports $lt ---  How do they use that type with the DSL without patching Casbah itself? ``BigDecimal`` is an example of this; the current code does not allow it to be used at all with ``$lt``.
    * What if a user has a domain specific reason to *restrict* certain of these types? I shouldn't even need to manufacture an example here: Users should be able to restrict types without patching code, if possible

These questions and the fact that I kept patching bugs, adding types, etc on user request led me to seek a new solution while working on Casbah 2.0. While searching for this solution I started to gain a rudamentary understanding of Type Classes and their power, and embarked upon a quest to give the users of Casbah flexible control over the types the DSL will and will not allow through.  Of course, I neglected to adequately document those changes, until now.

After the conversion to Type Classes, here's what the ``$lt`` operator's code looks like today::

{% highlight scala %}
/**
 * Trait to provide the $lt (Less Than) method on appropriate callers.
 *
 * Targets (takes a right-hand value of) String, Numeric, JDK And Joda Dates, 
 * Array, DBObject (and DBList), Iterable[_] and Tuple1->22.
 *
 *
 * @author Brendan W. McAdams <brendan@10gen.com>
 * @see http://www.mongodb.org/display/DOCS/Advanced+Queries#AdvancedQueries-%3C%2C%3C%3D%2C%3E%2C%3E%3D
 */
trait LessThanOp extends QueryOperator {
  private val oper = "$lt"

  def $lt(target: String) = op(oper, target)
  def $lt(target: DBObject) = op(oper, target)
  def $lt(target: Array[_]) = op(oper, target.toList)
  def $lt(target: Tuple1[_]) = op(oper, target.productIterator.toList)
  def $lt(target: Tuple2[_, _]) = op(oper, target.productIterator.toList)
  /** SNIP a bunch of individual tuple handling */
  def $lt(target: Iterable[_]) = op(oper, target.toList)
  def $lt[T: ValidDateOrNumericType](target: T) = op(oper, target)
}
{% endhighlight %}

You should, at this point, notice a fairly stark contrast between the old code and the new.  While we still have specific allowances for ``DBObject``, and have included allowances for things that are ``Iterable`` or ``Array``-like, the explicit support for Dates, Numbers and Booleans (Primitive Numbers and Booleans being part of ``AnyVal``) is gone.  Or are they?

The clever reader may have noticed something new lurking at the bottom of this code::

{% highlight scala %}
  def $lt[T: ValidDateOrNumericType](target: T) = op(oper, target)
{% endhighlight %}

This specific method is the pivot point of our new filter system: it is what allows users to independently define what types, beyond the hardcoded ones above, are and are not allowed to pass into the ``$lt`` operator.

But what does it all mean?

Of Type Classes and Context Bounds
----------------------------------

The method above defines a type parameter of ``[T: ValidDateOrNumericType]``; you're probably used to seeing covariance (``T <: ValidDateOrNumericType``) or contravariance (``T >: ValidDateOrNumericType``), but may not have encountered this notation before now.  The single colon type boundary is known as a *Context Bound*  It does not, as first guess might tell you, say that ``T`` must be *exactly* a ``ValidDateOrNumericType``.

Context Bounds are a fairly new syntax, recently introduced to the type parameter system in Scala 2.8.  What they give us is a shortcut for a type-dependent implicit argument (they also work with ``Manifests``, which I'll explain in a future post).  When the Scala Compiler parses our code it will actually produce the following statement, which is how we would have said this in versions prior to 2.8::

{% highlight scala %}
    def $lt[T](target: T)(implicit evidence$1: ValidDateOrNumericType[T]) = op(oper, target)
{% endhighlight %}

What the Context Boundary syntax says is that we want to accept a generic type ``T``, as long as there is an *implicit* instance of ``ValidDateOrNumericType[T]`` available.  With this simple statement, what we say is quite literally as long as an implicit instance of ``ValidDateOrNumericType`` is available for type ``T``, it is a valid type for this method.  This means that you, as a user, can quickly change what type are and aren't allowed into this ``$lt`` method merely by adjusting the implicit scope.  Powerful, no?

In this case, ``ValidDateOrNumericType`` itself is a completely empty trait (but, as we'll see in my next post can actually become quite powerful with the introduction of some methods!) which is used as a simple filter. The default imports for Casbah's DSL automatically gives you several predefined filter types::

{% highlight scala %}
implicit object JDKDateDoNOk extends JDKDateOk with ValidDateOrNumericType[java.util.Date]
implicit object JodaDateTimeDoNOk extends JDKDateOk with ValidDateOrNumericType[org.joda.time.DateTime]
implicit object BigIntDoNOk extends BigIntOk with ValidDateOrNumericType[BigInt]
implicit object IntDoNOk extends IntOk with ValidDateOrNumericType[Int]
implicit object ShortDoNOk extends ShortOk with ValidDateOrNumericType[Short]
implicit object ByteDoNOk extends ByteOk with ValidDateOrNumericType[Byte]
implicit object LongDoNOk extends LongOk with ValidDateOrNumericType[Long]
implicit object FloatDoNOk extends FloatOk with ValidDateOrNumericType[Float]
implicit object BigDecimalDoNOk extends BigDecimalOk with ValidDateOrNumericType[BigDecimal]
implicit object DoubleDoNOk extends DoubleOk with ValidDateOrNumericType[Double]
{% endhighlight %}

All one has to do to introduce a new type as valid to ``$lt`` is define an implicit instance of ``ValidDateOrNumeric[<YourType>]``.  Circling back to our original problem, the solution to "How do I allow ``ObjectId`` in ``$lt``" is quite easy:

{% highlight scala %}
implicit object ObjectIdOK extends ValidDateOrNumericType[org.bson.types.ObjectId]
{% endhighlight %}

As long as that is in scope of your code when you invoke your query statement (such as ``"_id" $lt new ObjectId(timestamp)``) it will construct a valid query to send to MongoDB.

Of course, there is a lot more that we can do with Type Classes than mere filtering.  In my next post, I'll show you how to take decoupling and separation of concerns to the ultimate extreme using Type Classes.
