---
title: Forcing Scala Compiler 'Nothing' Checks
tags: scala effectivescala stupidpettricks
layout: post
---
Since early in its history, Casbah has had a helper method called `getAs[T]`, where `T` is "Some type you'd like to fetch a particular field as". Because of type erasure on the JVM, working with a Mongo Document can be annoying -- the representation in Scala is the equivalent of a `Map[String, Any]`. If we were to work with the `Map[String, Any]` in a standard mode, fetching a field *balance* which is a `Double` would require manual casting.

{% highlight scala %}
val doc: DBObject = someCollection.findOne()

val balance = doc.get("balance")
{% endhighlight %}

We have already hit another issue here -- in Scala, invoking `get` on a `Map` returns `Option[T]` (Where, in this case, `T` is of type `Any`). Which means casting has become more complex: to get a `Double` we also have to unwrap the `Option[Any]` first. A lazy man's approach might be something hairy like so:

{% highlight scala %}
balance.getOrElse(null).asInstanceOf[Double]
{% endhighlight %}

In the annals of history (when men were *real* men, and small furry creatures from Alpha Centauri were *real* small furry creatures from Alpha Centauri), the above became an annoyingly common pattern. A solution was needed - and so `getAs[T]` was born. The idea was not only to allow a shortcut to casting, but take care of the `Option[T]` wrapping for you as well. Invoking `getAs[Double]` will, in this case, return us an `Option[Double]`. 

But not everything is perfect in the land of `getAs[T]` -- if the type requested doesn't match the actual type, runtime failures occur. Worse, if the user fails to pass a type, the Scala compiler substitutes `Nothing`, which *guarantees* a runtime failure. Runtime failures are bad -- but fortunately, [Miles Sabin](http://twitter.com/milessabin) & [Jon-Anders Teigen](http://twitter.com/jteigen) came up with an awesome solution.

<!--more-->

Initially, the best we could do within Casbah was attempt to detect the compiler substituted `Nothing` and warn the user at runtime. Less than ideal, I know.

{% highlight scala %}
def getAs[A <: Any: Manifest](key: String): Option[A] = {
  require(manifest[A] != manifest[scala.Nothing],
    "Type inference failed; getAs[A]() requires an explicit type argument " +
    "(e.g. dbObject.getAs[<ReturnType>](\"somegetAKey\") ) to function correctly.")

  underlying.get(key) match {
    case null => None
    case value => Some(value.asInstanceOf[A])
  }
}
{% endhighlight %}

Great -- we prevent people from utterly failing to pass a type to `getAs` by throwing an exception at runtime. A bit like closing the barn doors after the horses escaped, and somewhat counter to the point of compiled languages. Fortunately, Miles Sabin knows a lot of great compiler tricks and Jon-Anders has superpowers (which he uses for good, not evil). Using some of Miles' tricks, Jon-Anders has fixed Casbah (as of 2.3.0+) to make `getAs[T]` fail utterly at *compile time* when no type is passed.

The secret to this trick is essentially that the Scala compiler *hates* ambiguity. In order to substitute `Nothing` as a type argument when one isn't supplied, the Scala compiler has an implicit for `Nothing` scoped. If one were to exacerbate the situation by introducing an additional implicit for `Nothing`, the compiler would fail when no type argument is passed. 

With this in mind, we can morph `getAs` to work with a type class instead of a standard type argument. 

{% highlight scala %}
def getAs[A : NotNothing](key: String): Option[A] = {
  underlying.get(key) match {
    case null => None
    case value => Some(value.asInstanceOf[A])
  }
}

sealed trait NotNothing[A]{
  type B
}
{% endhighlight %}

Our previous unbounded type argument is replaced with the new type class boundary of `NotNothing` and the runtime `Nothing` check is removed. We also need concrete instances of our type class, which is where the real magic comes into play.

{% highlight scala %}
object NotNothing {
  implicit val nothing = new NotNothing[Nothing]{ type B = Any }
  implicit def notNothing[A] = new NotNothing[A]{ type B = A }
}
{% endhighlight %}

Now, any application of `Nothing` will trigger the ambiguity problem -- the Scala compiler won't figure out how to resolve the type argument. This trick works because `Nothing` is at the *bottom* of Scala's type hierarchy. Were I to call `getAs("balance")`, the Scala compiler would attempt to fill in `Nothing` as the type argument. However, both implicit conversons for `nothing` *and* `notNothing[A]` will match -- causing ambiguity and compilation fails.

The moral of the story -- knowing the ins and outs of the type system and compiler corners can do great things for improving the functionality of your code.
