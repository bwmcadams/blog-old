---
title: MapReduce with MongoDB 1.8 and Java
tags: tech mongodb mapreduce java
layout: post
---
In my [last post](http://blog.evilmonkeylabs.com/2011/01/27/MongoDB-1_8-MapReduce/), I introduced the 
new MapReduce features introduced in MongoDB 1.8, which is now [available as a release candidate](http://www.mongodb.org/display/DOCS/1.8+Release+Notes).  Most importantly the temporary collection system has gone away, now requiring that you specify an output parameter.  With that required output comes new options for how to create incremental output using the *merge* and *reduce* output modes.

As I write this, we are prepping new releases of our [Java Driver](http://www.mongodb.org/display/DOCS/Java+Language+Center) (v2.5) and our [Scala Driver, Casbah](http://api.mongodb.org/scala/casbah) (v2.1) which are intended to support MongoDB 1.8's new features including incremental MapReduce.  Since I implemented the APIs for the new MapReduce output in both drivers, I thought I'd demonstrate the application of these new output features to the previous dataset.  This post is focused on the Java API, but a Scala one will likely follow.

As a reminder (or a primer for those who skipped my [last post](http://blog.evilmonkeylabs.com/2011/01/27/MongoDB-1_8-MapReduce/)), I've been testing the 1.8 MapReduce using a dataset and MapReduce job originally created to test the [MongoDB+Hadoop Plugin](http://github.com/mongodb/mongo-hadoop).  It consists of daily U.S. Treasury Yield Data for about 20 years; the MapReduce task calculates an annual average for each year in the collection.  You can grab a copy of the entire collection in a handy mongoimport friendly [datadump from the MongoDB+Hadoop repo](https://github.com/mongodb/mongo-hadoop/raw/master/examples/treasury_yield/resources/yield_historical_in.json); here's a quick snippet of it:
 
{% highlight javascript %}

{ "_id" : ISODate("1990-01-10T00:00:00Z"), "dayOfWeek" : "WEDNESDAY", "bc3Year" : 7.95, "bc5Year" : 7.92, "bc10Year" : 8.03, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 7.91, "bc3Month" : 7.75, "bc30Year" : 8.11, "bc1Year" : 7.77, "bc7Year" : 8, "bc6Month" : 7.78 }
{ "_id" : ISODate("1990-01-11T00:00:00Z"), "dayOfWeek" : "THURSDAY", "bc3Year" : 7.95, "bc5Year" : 7.94, "bc10Year" : 8.04, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 7.91, "bc3Month" : 7.8, "bc30Year" : 8.11, "bc1Year" : 7.77, "bc7Year" : 8.01, "bc6Month" : 7.8 }
{ "_id" : ISODate("1990-01-12T00:00:00Z"), "dayOfWeek" : "FRIDAY", "bc3Year" : 7.98, "bc5Year" : 7.99, "bc10Year" : 8.1, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 7.93, "bc3Month" : 7.74, "bc30Year" : 8.17, "bc1Year" : 7.76, "bc7Year" : 8.07, "bc6Month" : 7.8100000000000005 }
{ "_id" : ISODate("1990-01-16T00:00:00Z"), "dayOfWeek" : "TUESDAY", "bc3Year" : 8.13, "bc5Year" : 8.11, "bc10Year" : 8.2, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 8.1, "bc3Month" : 7.89, "bc30Year" : 8.25, "bc1Year" : 7.92, "bc7Year" : 8.18, "bc6Month" : 7.99 }

{% endhighlight %}
        
The map function I'm using extracts the year from the date, and the 10 year benchmark value:

{% highlight javascript %}
function m() { 
    key = typeof( this._id ) == "number" ? this._id : this._id.getYear() + 1900; 
    emit( key, { count: 1, sum: this.bc10Year } ) ;
}
{% endhighlight %} 

... while the reduce function aggregates the data by year, creating a set that can be averaged.  Remember that MongoDB reduce tasks have to be able to be called repeatedly, so the output is crafted to match the input: something that becomes even more important when we say, ask MongoDB to re-reduce our output with the old data.

{% highlight javascript %}

function r( year, values ) { 
  var n = { count: 0, sum: 0 } 
  for ( var i = 0; i < values.length; i++ ){ 
      n.sum += values[i].sum; 
      n.count += values[i].count; 
  } 
   
  return n; 
} 

{% endhighlight %}

We'll round it all out out with a quick and dirty finalize function which can calculate the current average.  Note that I'm keeping all the intermediate data around for demonstrating "reduce" mode.

{% highlight javascript %}
function f( year, value ){
  value.avg = value.sum / value.count;
  return value;
}
{% endhighlight %}

First, we'll need to stick these functions into some Java strings to pass around for our testing:

{% highlight java %}
        String m = "function() { key = typeof( this._id ) == \"number\" ? this._id : this._id.getYear() + 1900;" +
                   "emit( key, { count: 1, sum: this.bc10Year } );";

        String r = "function( year, values ) { var n = { count: 0, sum: 0};" +
                   " for ( var i = 0; i < values.length; i ++ ) { n.sum += values[i].sum; " + 
                   " n.count += values[i].count; } return n; }";

        String f = "function( year, value ) { value.avg = value.sum / value.count; return value; }";%}

{% endhighlight %}


The Java API now allows you to pass an optional **[MapReduceCommand.OutputType](http://api.mongodb.org/java/2.5-pre-/com/mongodb/MapReduceCommand.OutputType.html)** value, which controls the type of output received.  If one is not specifed, the output collection is assumed to be **REPLACE** --- namely, the standard mode in which the named collection is replaced completely with the output of the MapReduce job.  Looking at **INLINE** as our example, we can call the new method in collection.  Feel free to set the output collection name to *null* or a throwaway value; it is ignored by the Java driver in Inline output mode.

{% highlight java %}
MapReduceOutput out = coll.mapReduce(m, r, null, MapReduceCommand.OutputType.INLINE, null);

for ( DBObject obj : out.results() ) {
    System.out.println( obj );
}

{% endhighlight %} 

Which should output each DBObject in the results to the screen.  The new MapReduceOutput code detects the result set type from MongoDB and provides it in `results()` as a **Iterable<DBObject>**---whether the results are **INLINE** or stored in a collection.  Note that I did not specify the `finalize` function here, as the interface on **DBCollection** doesn't accept it as a parameter.  Alternately, we could construct a **[MapReduceCommand](http://api.mongodb.org/java/2.5-pre-/com/mongodb/MapReduceCommand.html)** instance, which allows us to set `finalize`:

{% highlight java %}
MapReduceCommand cmd = new MapReduceCommand( coll, m, r, null, MapReduceCommand.OutputType.INLINE, null);
cmd.setFinalize(f);

out = coll.mapReduce(cmd);

for ( DBObject obj : out.results() ) {
    System.out.println( obj );
}

{% endhighlight %}

Using the **MERGE** and **REDUCE** Output Modes follows much the same pattern.  I'll leave figuring out **MERGE** as an exercise for the reader (it should be easy if you read my last post) but here's how we would handle a **REDUCE** on two halves of the same year.  

{% highlight java %}
cmd = new MapReduceCommand( coll, m, r, "yield_historical.out", MapReduceCommand.OutputType.REDUCE, 
                            new BasicDBObject("_id", new BasicDBObject(
                                                        "$gte", new Date( 101, 0, 1 )
                                                     ).append(
                                                        "$lte", new Date( 101, 5, 1 )
                                                     )
                                            )
                           );

cmd.setFinalize(f);

/** Ignore output of first half... */
coll.mapReduce(cmd);

/** Reduce in second half */
cmd = new MapReduceCommand( coll, m, r, "yield_historical.out", MapReduceCommand.OutputType.REDUCE, 
                            new BasicDBObject("_id", new BasicDBObject(
                                                        "$gt", new Date( 101, 5, 1 )
                                                     ).append(
                                                        "$lte", new Date( 101, 11, 31 )
                                                     )
                                             )
                           );

cmd.setFinalize(f);

out = coll.mapReduce(cmd);

for ( DBObject obj : out.results() ) {
    System.out.println( obj );
}

{% endhighlight %}


That's it! Go forth, and MapReduce...
