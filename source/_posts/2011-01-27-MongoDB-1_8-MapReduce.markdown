---
title: A Look At MongoDB 1.8's MapReduce Changes
categories: 
  - mongodb 
  - mapreduce
comments: true
layout: post
---
MongoDB 1.7.5 shipped yesterday, and is expected to be the last 'beta' release of what will become MongoDB 1.8.  As part of the release, I've been doing testing of the new MapReduce functionality and thought this a good time to highlight those changes for people.

If you aren't new to MongoDB MapReduce, the most important thing to note since MongoDB 1.6.x is that temporary collections are gone; it is now *required* to specify an output.  Previously, if you omitted the _out_ argument MongoDB would create a temporary collection and return its name with the job results; In non-sharded MongoDB setups these temporary collections would go out of scope and be cleaned up when the connection closed.  Unfortunately, for sharded setups it wasn't possible to safely clean these up–--they would remain behind and clutter up the database.  For this and other reasons the temporary collection feature was removed. There is good news though: they've been replaced with an even better system for saving the results of MapReduce jobs!

While the _out_ argument is now a required parameter in MapReduce jobs, it has a number of options for controlling what MongoDB does with results.  If you're running a truly one-off job where you don't need to keep the results later, MongoDB now supports returning results "inline".  Be careful here though: your results are being returned in a single document and are subject to the document size limitations of MongoDB (16MB per document in 1.8).  To use inline results, set the value of _out_ to a document `{inline: 1}`.  The result object will contain an additional key _results_ which contains the MapReduce output; the _result_ field will be omitted.

As with previous versions of MongoDB, you can specify a collection name (as a string) in the _out_ argument.  If the named collection already exists MongoDB will replace it entirely with the MapReduce results. Along with the inline mode, MongoDB 1.8 introduces support for "merge" and "reduce" output modes; instead of replacing the target collection MongoDB can be instructed to reconcile the MapReduce results with the existing data.  To use these modes, set the value of _out_ to a document with a key of either "merge" or "reduce" and a value of the collection to save to.

The difference in "merge" and "reduce" has to do with MongoDB does when it encounters duplicate keys in both the existing collection and the MapReduce results.  In "merge" mode, MongoDB will simply overwrite the existing key with the new one from the MapReduce output.  In "reduce" mode, MongoDB will run the reduce function again with both the new and old data, saving those results to the collection (you remembered to make your reduce function idempotent, right?). **UPDATE**: If you specified a "finalize" function, MongoDB will re-run this after the "reduce" runs.

Now that I've thoroughly confused you, lets dig into examples of each of these behaviors.  I've been testing the 1.8 MapReduce using a dataset and MapReduce job originally created to test the [MongoDB+Hadoop Plugin](http://github.com/mongodb/mongo-hadoop).  It consists of daily U.S. Treasury Yield Data for about 20 years; the MapReduce task calculates an annual average for each year in the collection.  You can grab a copy of the entire collection in a handy mongoimport friendly [datadump from the MongoDB+Hadoop repo](https://github.com/mongodb/mongo-hadoop/raw/master/examples/treasury_yield/resources/yield_historical_in.json); here's a quick snippet of it:
 
{% highlight javascript %}

{ "_id" : ISODate("1990-01-10T00:00:00Z"), "dayOfWeek" : "WEDNESDAY", "bc3Year" : 7.95, "bc5Year" : 7.92, "bc10Year" : 8.03, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 7.91, "bc3Month" : 7.75, "bc30Year" : 8.11, "bc1Year" : 7.77, "bc7Year" : 8, "bc6Month" : 7.78 }
{ "_id" : ISODate("1990-01-11T00:00:00Z"), "dayOfWeek" : "THURSDAY", "bc3Year" : 7.95, "bc5Year" : 7.94, "bc10Year" : 8.04, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 7.91, "bc3Month" : 7.8, "bc30Year" : 8.11, "bc1Year" : 7.77, "bc7Year" : 8.01, "bc6Month" : 7.8 }
{ "_id" : ISODate("1990-01-12T00:00:00Z"), "dayOfWeek" : "FRIDAY", "bc3Year" : 7.98, "bc5Year" : 7.99, "bc10Year" : 8.1, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 7.93, "bc3Month" : 7.74, "bc30Year" : 8.17, "bc1Year" : 7.76, "bc7Year" : 8.07, "bc6Month" : 7.8100000000000005 }
{ "_id" : ISODate("1990-01-16T00:00:00Z"), "dayOfWeek" : "TUESDAY", "bc3Year" : 8.13, "bc5Year" : 8.11, "bc10Year" : 8.2, "bc20Year" : null, "bc1Month" : null, "bc2Year" : 8.1, "bc3Month" : 7.89, "bc30Year" : 8.25, "bc1Year" : 7.92, "bc7Year" : 8.18, "bc6Month" : 7.99 }

{% endhighlight %}

<!--more-->
        
The map function I'm using extracts the year from the date, and the 10 year benchmark value:

{% highlight javascript %}
function m() { 
    key = typeof( this._id ) == "number" ? this._id : this._id.getYear() + 1900; 
    emit( key, { count: 1, sum: this.bc10Year } ) ;
}
{% endhighlight %} 

While the reduce function aggregates the data by year, creating a set that can be averaged.  Remember that MongoDB reduce tasks have to be able to be called repeatedly, so the output is crafted to match the input: something that becomes even more important when we say, ask MongoDB to re-reduce our output with the old data.

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


First, a quick look at "inline" mode (I'll leave plain old name a collection as an exercise to you, my humble reader).

{% highlight javascript %}
> res = db.runCommand(
...   { 
...     "mapreduce": "yield_historical.in",
...     "map": m,
...     "reduce": r,
...     "finalize": f,
...     "query" : { "_id" : { "$gt" : new Date(2000, 0, 1) } },
...     "verbose" : true , 
...     "out" : { "inline" : 1 }
...   }
... )
{
	"results" : [
		{
			"_id" : 1990,
			"value" : 8.552400000000002
		},
        /* ... */
		{
			"_id" : 2010,
			"value" : 3.3255026455026435
		}
	],
	"timeMillis" : 218,
	"timing" : {
		"mapTime" : NumberLong(168),
		"emitLoop" : 215,
		"total" : 218
	},
	"counts" : {
		"input" : 2690,
		"emit" : 2690,
		"output" : 11
	},
	"ok" : 1
}

{% endhighlight %}


To demonstrate "merge" and "reduce" mode, I'm going to use queries to break out the data a bit.  Lets look first at "merge", by first running MapReduce against the first half of the data, and then merge in the second half.


{% highlight javascript %}

> res = db.runCommand(
...   { 
...     "mapreduce": "yield_historical.in",
...     "map": m,
...     "reduce": r,
...     "finalize": f,
...     "query" : { "_id" : { "$lt" : new Date(2000, 0, 1) } },
...     "verbose" : true , 
...     "out" : "yield_historical.merged",
...   }
... )
{
	"result" : "yield_historical.merged",
	"timeMillis" : 223,
	"timing" : {
		"mapTime" : NumberLong(166),
		"emitLoop" : 217,
		"total" : 223
	},
	"counts" : {
		"input" : 2503,
		"emit" : 2503,
		"output" : 10
	},
	"ok" : 1
}
> db.yield_historical.merged.find({}, {"value.avg": 1})
{ "_id" : 1990, "value" : { "avg" : 8.552400000000002 } }
{ "_id" : 1991, "value" : { "avg" : 7.8623600000000025 } }
{ "_id" : 1992, "value" : { "avg" : 7.008844621513946 } }
{ "_id" : 1993, "value" : { "avg" : 5.866279999999999 } }
{ "_id" : 1994, "value" : { "avg" : 7.085180722891565 } }
{ "_id" : 1995, "value" : { "avg" : 6.573920000000002 } }
{ "_id" : 1996, "value" : { "avg" : 6.443531746031743 } }
{ "_id" : 1997, "value" : { "avg" : 6.353959999999992 } }
{ "_id" : 1998, "value" : { "avg" : 5.262879999999994 } }
{ "_id" : 1999, "value" : { "avg" : 5.646135458167332 } }
> 
{% endhighlight %}

That gives us our first half of the data; we ran that with a normal named collection output.  Lets merge in the second half:

{% highlight javascript %}
> res = db.runCommand(
...   { 
...     "mapreduce": "yield_historical.in",
...     "map": m,
...     "reduce": r,
...     "finalize": f,
...     "query" : { "_id" : { "$gt" : new Date(2000, 0, 1) } },
...     "verbose" : true , 
...     "out" : { "merge" : "yield_historical.merged" },
...   }
... )
{
	"result" : "yield_historical.merged",
	"timeMillis" : 242,
	"timing" : {
		"mapTime" : NumberLong(173),
		"emitLoop" : 236,
		"total" : 242
	},
	"counts" : {
		"input" : 2690,
		"emit" : 2690,
		"output" : 21
	},
	"ok" : 1
}

> db.yield_historical.merged.find({"_id": {$gt: 1998}}, {"value.avg": 1}) 
{ "_id" : 1999, "value" : { "avg" : 5.646135458167332 } }
{ "_id" : 2000, "value" : { "avg" : 6.030278884462145 } }
{ "_id" : 2001, "value" : { "avg" : 5.020685483870969 } }
{ "_id" : 2002, "value" : { "avg" : 4.61308 } }
{ "_id" : 2003, "value" : { "avg" : 4.013879999999999 } }
{ "_id" : 2004, "value" : { "avg" : 4.271320000000004 } }
{ "_id" : 2005, "value" : { "avg" : 4.288880000000001 } }
{ "_id" : 2006, "value" : { "avg" : 4.7949999999999955 } }
{ "_id" : 2007, "value" : { "avg" : 4.634661354581674 } }
{ "_id" : 2008, "value" : { "avg" : 3.6642629482071714 } }
{ "_id" : 2009, "value" : { "avg" : 3.2641200000000037 } }
{ "_id" : 2010, "value" : { "avg" : 3.3255026455026435 } }

{% endhighlight %}

To close out, lets take "reduce" mode for a quick spin.  We'll select a half of a year for the first part, and then reduce in the second half.

{% highlight javascript %}
> res = db.runCommand(
...   { 
...     "mapreduce": "yield_historical.in",
...     "map": m,
...     "reduce": r,
...     "finalize": f,
...     "query" : { "_id" : { 
...         "$gte": new Date(2001, 0, 1),
...         "$lte" : new Date(2001, 5, 1) 
...     } },
...     "verbose" : true , 
...     "out" : "yield_historical.reduced",
...   }
... )
{
	"result" : "yield_historical.reduced",
	"timeMillis" : 21,
	"timing" : {
		"mapTime" : NumberLong(6),
		"emitLoop" : 17,
		"total" : 21
	},
	"counts" : {
		"input" : 105,
		"emit" : 105,
		"output" : 1
	},
	"ok" : 1
}
> db.yield_historical.reduced.find()
{ "_id" : 2001, "value" : { "count" : 105, "sum" : 539.5599999999998, "avg" : 5.138666666666665 } }
{% endhighlight %}

That handles the first half... Let's grab the second:

{% highlight javascript %}
> res = db.runCommand(              
...   { 
...     "mapreduce": "yield_historical.in",
...     "map": m,
...     "reduce": r,
...     "finalize": f,
...     "query" : { "_id" : { 
...         "$gt": new Date(2001, 5, 1),
...         "$lte" : new Date(2001, 11, 31) 
...     } },
...     "verbose" : true , 
...     "out" : { "reduce" : "yield_historical.reduced" },
...   }
... )
{
	"result" : "yield_historical.reduced",
	"timeMillis" : 26,
	"timing" : {
		"mapTime" : NumberLong(9),
		"emitLoop" : 22,
		"total" : 26
	},
	"counts" : {
		"input" : 143,
		"emit" : 143,
		"output" : 1
	},
	"ok" : 1
}
> db.yield_historical.reduced.find()
{ "_id" : 2001, "value" : { "count" : 248, "sum" : 1245.1299999999997, "avg" : 5.020685483870967 } }
{% endhighlight %}

Of course, this does us no good if the results don't add up.  A quick comparison between the 'merged' output and the 'reduced' output validates our code:

{% highlight javascript %}
> db.yield_historical.reduced.find({_id: 2001})
{ "_id" : 2001, "value" : { "count" : 248, "sum" : 1245.1299999999997, "avg" : 5.020685483870967 } }
> db.yield_historical.merged.find({_id: 2001}) 
{ "_id" : 2001, "value" : { "count" : 248, "sum" : 1245.1300000000003, "avg" : 5.020685483870969 } }
{% endhighlight %}

There are some minor differences at a decimal level since we are working with floating point numbers here, but the results are the same.  

These new MapReduce output parameters are available in MongoDB as of version 1.7.4 (which is part of the unstable/development branch) and will ship with MongoDB 1.8. Leave a comment; I'd love to hear what clever tricks you can pull off with these new options.
