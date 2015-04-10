---
title: Pickling Case Classes to Memcached with Scala
categories: 
  - scala 
  - pickling 
  - artisanbytecode

comments: true
layout: post
---
Recently, I've been working on a rewrite of [Sluggy Freelance](http://www.sluggy.com) - a friend's site which I've worked on for about a decade now. Caching is a big part of keeping site cost down, and over the years I've come to trust [Memcached](http://memcached.org). Fast, lightweight, and easy, Memcached has served me well over the years over several iterations of the site... from Perl, PHP, and Python.

Today, I'm rewriting the site in [Scala](http://www.scala-lang.org) with [Scalatra](http://www.scalatra.org) and [React.js](http://facebook.github.io/react/). As a result, I'm discovering all sorts of new fun that I haven't dealt with in Scala yet. One of these is using Memcached, and specifically serialising/deserialising Case Classes. With the combination of a good Memcached library - [shade](http://github.com/alexandru/shade), in this case - and [Scala Pickling](https://github.com/scala/pickling)\*, I've found a powerful combination. 

So far, I'm working with a datastructure representing navigation: the current book, chapter, and section as well as lists of others of those based on context. As a bonus... I'm actually serialising case classes that are fetched from Slick.

<!--more-->
Here's the navigation structure:

{% highlight scala linenos %}
case class NavigationStruct(
  book: Book,
  allBooks: Seq[Book],
  section: Section,
  currentSections: Seq[Section],
  chapter: Chapter,
  currentChapters: Seq[Chapter]
)
{% endhighlight %}

We'll leave the content of `Book`, `Section`, and `Chapter` out, but suffice to say they are standard case classes made up of a variety of primitive fields and a few Joda `LocalDate`s for good measure.

Here are my imports, setting up memcached :

{% highlight scala linenos %}
import shade.memcached._
// shade uses futures so you'll need the execution context
import scala.concurrent.ExecutionContext.Implicits.{global => ec}

{% endhighlight %}

Now, given the topic of conversation what we need is two things:

  1. Code to convert a `NavigationStruct` case class to and from binary via pickling
  2. Code to tell the shade memcached client how to convert scala data to and from memcached

Luckily, we can combine these two things. shade provides a `Codec[T]` mechanism for encoding, into which we can wire our pickling. Here's a rough sketch of my `Codec[T]` for `NavigationStruct`:

{% highlight scala linenos %}
  implicit object NavStructCodec extends Codec[NavigationStruct] {
    def serialize(struct: NavigationStruct): Array[Byte] =  ???

    def deserialize(data: Array[Byte]): NavigationStruct = ???
  }

{% endhighlight %}

Our implicit `Codec[T]` object has two methods: `serialize(T): Array[Byte]` and `deserialize(Array[Byte]): T`. This is the shade memcached specific code: It leaves it up to us how we want to serialize/deserialize as long as we work with Arrays of bytes.

And so, in comes Scala pickling...
{% highlight scala linenos %}
import scala.pickling._
import scala.pickling.Defaults._
import scala.pickling.binary._
{% endhighlight %}
 While I import `binary` support from Pickling, it is worth noting that there's also JSON Support built in as outlined in [the docs]](https://github.com/scala/pickling).

 With the `scala.pickling.Defaults._` import comes a few implicits, which let us "pickle" (aka serialize) arbitrary objects:

{% highlight scala linenos %}
  struct.pickle.value 
{% endhighlight %}

And "unpickle" (aka deserialize) from raw bytes back to something useful:
{% highlight scala linenos %}
  data.unpickle[NavigationStruct]
{% endhighlight %}

That's actually it, at least for my simple data structures. Here's the final implicit object:

{% highlight scala linenos %}
  implicit object NavStructCodec extends Codec[NavigationStruct] {
    def serialize(struct: NavigationStruct): Array[Byte] = { 
      struct.pickle.value 
    }

    def deserialize(data: Array[Byte]): NavigationStruct = {
      data.unpickle[NavigationStruct]
    }
  }
{% endhighlight %}

This leaves me with an easy route to fetch/save between memcached, where codec handling is automatically taken care of:

{% highlight scala linenos %}
memcached.awaitGet[NavigationStruct](key)(NavStructCodec) match {
  case Some(navData) => 
    navData
  case None =>
    val nav = getNavigationData(date)
    memcached.awaitSet(key, nav, timeout)
    nav
}
{% endhighlight %}

Suffice to say, I'm quite happy with the simplicity.

\* It's been mentioned to me as well that [scodec](https://github.com/scodec/scodec) is a great solution for serialisation too, but I haven't yet gone down that road.
