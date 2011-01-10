---
title: Exploring Scala with MongoDB
tags: tech mongodb scala casbah
layout: post
---

2010 proved to be a great year for growth and adoption of many fledgling technologies---not least among them, [MongoDB](http://mongodb.org/ "MongoDB") and [Scala](http://scala-lang.org).  Scala is designed as an alternative language for the Java platform, with a focus on scalability. It merges many of the Object Oriented concepts of languages like Java and C++ with the functional tools of Erlang, Haskell and Lisp with a bit of the dynamic natures of modern languages like Ruby and Python.  This flexible nature has sped Scala's adoption in the technology stacks of platforms like [LinkedIn](http://www.scala-lang.org/node/7806), [Twitter](http://www.artima.com/scalazine/articles/twitter_on_scala.html), [FourSquare](http://www.10gen.com/video/misc/foursquare) and many more.  By running on the JVM Scala has a strong affinity for working alongside existing Java applications, which allows users to build on their existing technology investments.

For 2011, MongoDB has added official support for Scala with the release of [Casbah](http://api.mongodb.org/scala/casbah/latest/), a Scala driver for MongoDB.  Casbah is built around the existing MongoDB Java Driver to give it a strong foundation, but designed to take advantage of many of the idioms of Scala such as a strong collections library, fluid syntax for building DSLs and functional concepts like closures and currying.   

Because it is designed to be easy to work with for Scala users, Casbah introduces a more 'friendly' syntax for creating MongoDB Objects, using Scala's Map syntax:


{% highlight scala %} 
import com.mongodb.casbah.Imports._

/** Create an object directly */
val newObj = MongoDBObject("foo" -> "bar",
                           "x" -> "y",
                           "pie" -> 3.14,
                           "spam" -> "eggs")

/** Or, use a builder interface */
val builder = MongoDBObject.newBuilder
builder += "foo" -> "bar"
builder += "x" -> "y"
builder += ("pie" -> 3.14)
builder += ("spam" -> "eggs", "mmm" -> "bacon")
val newObj = builder.result
{% endhighlight %}

The goal of this syntax is to be more readable, similar to what one might expect from a dynamic language like Ruby or Python.  In contrast, the same statements in Java tend to be more verbose:

{% highlight scala %}
import com.mongodb.*;

DBObject newObj = new BasicDBObject();
newObj.put("foo", "bar");
newObj.put("x", "y");
newObj.put("pie", 3.14);
newObj.put("spam", "eggs");

/** or, builder style */

BasicDBObjectBuilder builder = BasicDBObjectBuilder.start();
builder.add("foo", "bar");
builder.add("x", "y");
builder.add("pie", 3.14);
builder.add("spam", "eggs");
builder.add("mmm", "bacon");
DBObject newObj = builder.get();

{% endhighlight %}

The semantics of working with Collections and Cursors in Casbah are similar to the Java driver they wrap, with a bit of Scala-friendly syntactic sugar added for things like [for comprehensions](http://www.scala-lang.org/node/111).  Where Casbah really shines is in its use of a DSL syntax for creating MongoDB Queries. 

{% highlight scala %}

/** This Query Object ... */
val query = new MongoDBObject(
                "foo" -> MongoDObject("$gte" -> 5, "$lte" -> 10),
                "baz" -> 5,
                "x" -> "y",
                "n" -> "r"
            )
/** Can be constructed instead with the Query DSL: */
val queryDSL = ("foo" $gte 5 $lte 10) ++ ("baz" -> 5) ++ ("x" -> "y") ++ ("n" -> "r")

/** Easily create negated statements. 
    Instead of a nested DBObject constructor like this: */

val ltGt = MongoDBObject(
            "foo" -> MongoDBObject(
                "$not" -> MongoDBObject(
                    "$gte" -> 15, 
                    "$lt" -> 35.2, 
                    "$ne" -> 16)
                )
            )

/** Use Casbah's Query DSL to say it much simpler */
val ltGtDSL = "foo" $not { _ $gte 15 $lt 35.2 $ne 16 }

{% endhighlight %}

All of MongoDB's *$ Operators* including Geospatial Queries are supported by Casbah's DSL. 

This is just a small taste of what Casbah and Scala offer to the MongoDB user, but we encourage you to explore more.  [Version 2.01](http://api.mongodb.org/scala/casbah/latest/setting_up.html) is now available for download.

