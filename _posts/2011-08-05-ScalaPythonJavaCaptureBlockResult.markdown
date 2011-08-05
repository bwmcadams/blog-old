---
title: Immutability and Clever Variable Usage in the Land of Blocks and Branches
tags: tech scala effectivescala bestpractices python
layout: post
---
Last night, I found myself unconciously refactoring some Scala code (I don't recall if it was something I wrote or someone else did at this point). As I looked at what I was doing I realized that many Scala developers don't seem entirely aware of one of my favorite features.  What I'm talking about is effectively capturing values from multibranch block statements in Scala.  Used correctly they can greatly decruft complicated code as well as helping us use mutability in places we might not expect an easy way to do so.

In typical C-like languages (such as C, C++, and Java) we are restricted in our syntax should we wish to capture a value when running many branching blocks such as if-else statements, switch statements and even for/foreach constructs. When we find ourselves wanting to set the value of a variable within each possible condition or iteration, we need to declare a mutable variable before the block.  We then mutate this variable within each condition or iteration.  Take this example from Java:

{% highlight java %}
boolean valid = false;

String status = null;
if (valid) {
    status = "VALIDATED";
} 
else {
    status = "INVALID";
}
{% endhighlight %}

Not only have we had to declare a mutable value, but adding insult to injury the typical usage pattern includes declaring it as `null` (FACT: Everytime you explicitly use null, someone drowns a basket of adorable, whimpering puppies. Even if your language paradigm practically requires it.).  Within our if/else block we have established a new value for `status` based on the value of `valid`. Granted, this is a somewhat weak example of my point, as you could just as easily have expressed this in a more concise ternary statement:

{% highlight java %}
String status = valid ? "VALIDATED" : "INVALID";
{% endhighlight %}

This simplification breaks down quickly once our block becomes more complex.  Even the addition of a single `elseif` would eliminate the usefulness of the ternary syntax:

{% highlight java %}
// In this case, valid is now an int that can hold several states
int valid = -1; 

/* valid = SomeMethodCallThatReturnsValid() */

String status = null;

if (valid == 1) {
    status = "VALIDATED";
} 
else if (valid == 0) {
    status = "INVALID"; // A simple not-valid indicator
} 
else if (valid == -1) {
    status = "UNINITIALIZED_ERROR";
}
else {
    status = "UNKNOWN_ERROR";
}
{% endhighlight %}

We've already bounced ourselves back to the land of `null` and forced mutability (can you hear those poor, helpless puppies trying to learn how to swim?). We will encounter similar (or potentially worse) limitations in the case of a for/foreach construct.

{% highlight java %}
List<User> users = new List<User>();

for (DBRow row : databaseQuery(queryArgs)) {
    users.add(new User(row.get("username"), row.get("first_name"), row.get("last_name"));
}
{% endhighlight %}

Admittedly, this last block isn't *that* bad, but it still irks me to **have** to declare a mutable list to facilitate it.  We probably could work around with a builder pattern but **(a)** as far as I know Java lacks a builtin List Builder and **(b)** We'd still probably be "building" a mutable list.

These paradigms obviously don't restrict or limit everyone, but those of us who try to pay attention to immutability and the like start to grind our teeth awfully fast when encountering these patterns.  There has to be a better way, right?  How do other languages handle these constructs and related problems?

The Pythonic Way
----------------
For those that know Python — a language that I have long loved, and using Scala has only reinforced that as I learn &amp; understand more of the FP goodness in Python — there are some nice tricks to alleviate the previously demonstrated pain (and save those puppies).  Unfortunately, some of these tricks also stink a bit too much like syntactic magic; I tend to think at least one of them violates Python's "explicit over implicit" rules too.

{% highlight python %}
# Again, valid is now an int that can hold several states
valid = -1 

/* valid = some_method_to_set_valid() */

if valid == 1:
    status = "VALIDATED"
elif valid == 0:
    status = "INVALID"
elif valid == -1:
    status = "UNINITIALIZED_ERROR"
else:
    status = "UNKNOWN_ERROR"
{% endhighlight %}

Here's that magic I'm talking about — despite us not explicitly declaring/initializing it, Python will instantiate a variable `status` at the *outer* scope (that is to say, outside of our if-else block).  This is incredibly useful for cutting down line noise related to unecessary declaration of variables; unfortunately it is also a common confusion point for beginning Pythonistas. 

    *"Where the hell did this `status` variable come from? It wasn't ever declared in an outer scope!"*

While we've avoided initializing `status` to null, we still encounter the pesky mutability issue.  We ultimately ended up with a variable `status` which can be mutated later.  This isn't ideal, but as far as I know Python has no way to enable/enforce immutable state.

Python also can take us closer to an ideal state with the for/foreach constructs.  My big gripe with the previous Java for loop code was the need to declare a mutable `List` and incrementally add to it.  Python has a cure for what ails us here, in the form of generators — a feature which, as you'll see, I'd give my left pinkie (or maybe just the last millimeter of my left pinkie nail) to have in Scala as well.

Here's how a smart Pythonista might express our database processing loop in a way that limits the declaration of mutable variables:

{% highlight python %}
def process_dbusers(dbResults):
    for entry in dbResults:
        yield User(row.field('username'), row.field('first_name'), 
                   row.field('last_name'))

users = process_dbusers(db_query(query_args))
{% endhighlight %}

With this code, we've already quickly wandered into the land of "Things Java Can't Even Pretend To Do" here (which I think highlights my amusement at people arguing Java's power over Python.  It's all relative when you know the right syntax ... ).  To best handle this expression we've constructed a higher- order function `process_dbusers`, which takes in a database results iterator, returning a list-like construct with the results.  I want to emphasize the term *list-like* here because what we got is not, in fact, a list (rather it is "Iterable").  A quick peek at the REPL (if you don't already have [IPython](http://ipython.org) you are really missing out) evinces this:

{% highlight python %}
users
# <generator object process_db at 0x10ca93690>

type(users)
# <type 'generator'>
{% endhighlight %}

Python supports a special type of expression (which again, I'd love to have in Scala and have been playing with ways to support) called a *generator*.  In addition to being immutable, a generator is also lazy; it does not evaluate the entire list at creation, instead evaluating each member as it is first read (notably though, Python generators do not memoize and cannot be iterated repeatedly).  Note that Python's `yield` keyword differs from Scala's; it may be easiest to think of it as a "super-return" or "return on crack" in that it actually suspends execution (via coroutines) after returning *each* value.  

The 'generator' produced is actually an iterable value; requesting the 'next' item resumes execution, runs the next iteration of the internal for loop again, yields the return value and suspends again.  Not only is this generator value immutable, but it should be significantly more efficient in many cases; if we don't need to store each `User` but process them as we read them, we should see a lot better resource usage.

There is one other way to express the previous statement without needing a nested function set or the 'yield' keyword, using "generator expressions".  The downside to the `yield` based generators is that they must be wrapped inside another function, making them messy to inline with other code. With Generator Expressions we can get the same behavior in a compact, inline one-liner.  A Generator Expression is syntactically identical to a Python [List Comprehension](http://docs.python.org/tutorial/datastructures.html#list-comprehensions) with one key difference:  You must enclose the expression in parentheses ( `(` and `)` ) instead of square brackets ( `[` and `]`).  While a List Comprehension will return a list, enclosing that same statement in parentheses instead of brackets will produce a generator. This Generator Expression works as if we had written a multiline `for` construct with `yield`:

{% highlight python %}
# Define a tuple of test values
items = ('Foo', 'bar', 'Baz', 'spam', 'eggs', 'apples', 'oranges')

# Use a List Comprehension, returns a list
itemList = [item for item in items]
# ['Foo', 'bar', 'Baz', 'spam', 'eggs', 'apples', 'oranges']
# Use a Generator Expression which returns a generator
itemGen  = (item for item in items)
# <generator object <genexpr> at 0x10cafa050>

# You can also filter these expressions
# Return a list which filters to return only items which are 'food'
foodList = [item for item in items if item in ('spam', 'eggs', 'apples', 'oranges')]
# ['spam', 'eggs', 'apples', 'oranges']
# The same filter, but as a generator expression
foodGen = (item for item in items if item in ('spam', 'eggs', 'apples', 'oranges'))
# <generator object <genexpr> at 0x10cafa140>
# Generators are iterable ...
foodGen.next()
# 'spam'
foodGen.next()
# 'eggs'
{% endhighlight %}

We've certainly, on the Python side, gotten closer to a world which can exist without unecessary variable initialization and sane immutability... but aren't entirely where I'd like to be.  To understand things better let's look, finally, at what Scala lets us do.


The Scala Way
--------------

A rough take on the code I was optimizing last night can be used to highlight the power of Scala's way of solving the kinds of problems we are discussing.  Here's a Scala version of our if-elseif-else construct from Java:

{% highlight scala %}
val valid: Int = someMethodCallThatSetsValidity() 

var status: String = null // ack, mutable *and* null inited - A scala programmer's worst nightmare

if (valid == 1) {
  status = "VALIDATED"
} else if (valid == 0) {
  status = "INVALID" 
} else if (valid == -1) {
  status = "UNINITIALIZED_ERROR"
} else { 
  status = "UNKNOWN_ERROR"
}
{% endhighlight %}

There are lots of things that are egregious here from the eyes of an experienced Scala programmer, but it is also pretty close to the style I would expect from a fairly new Scala programmer.  There is of course a *much* better way to do this in Scala without any magic tricks.

One of the cool little edge behaviors of Scala's syntax is the ability to capture the return value of most block statements — even if-else statements.  This lends itself to quickly shortening our code *and* enforcing immutability on the final value of status.

{% highlight scala %}
val valid: Int = someMethodCallThatSetsValidity() 

val status = if (valid == 1) { 
  "VALIDATED" // returns "VALIDATED"
} else if (valid == 0) {
  "INVALID"  // returns "INVALID"
} else if (valid == -1) {
  "UNINITIALIZED_ERROR" // returns "UNINITIALIZED_ERROR"
} else { 
  "UNKNOWN_ERROR" // returns "UNKNOWN_ERROR"
}
{% endhighlight %}

Much better! The way that Scala evaluates this code, you can think of each branch of the block (the body of each "if", "else if" and "else" statements) as an anonymous function.  Because it is evaluated this way, Scala allows us to return values, which can be captured from the entire block (The use of the explicit `return` keyword is typically considered bad form in Scala which is why it is omitted here).  If `valid` has a value of `1`, this block will return `"VALIDATED"`.

The type of `safe` in this code will be inferred to `String`, because each branch returns a `String`. Although in Scala it is usually recommended to let type inference do its work wherever possible and avoid explicit type annotations, variables capturing values from a block like this might be a good exception to the rule.  

The argument that I make is that if the type inference system finds multiple *differing* types in each branch, it will search backwards on the type hierarchies for the closest "ancestor" type of each type.  This can quickly lead to the type of `status` being something far too generic (or even unexpected in the case of a bug).

We can see this quickly by changing the return of our `else` branch to a boolean:

{% highlight scala %}
val status = if (valid == 1) { 
  "VALIDATED"
} else if (valid == 0) {
  "INVALID" 
} else if (valid == -1) {
  "UNINITIALIZED_ERROR"
} else { 
  false 
}

/* Turn on "Power Mode" in the repl to dump the type (:power) 
scala> :type status
Any
*/
{% endhighlight %}

This is not good, and often will result in a bug.  If you were expecting `String`, you will be sorely disappointed. The easiest fix to this is to explicitly annotate an expected return type, which forces the compiler to help you enforce expectations:

{% highlight scala %}
val status: String = if (valid == 1) { 
  "VALIDATED"
} else if (valid == 0) {
  "INVALID" 
} else if (valid == -1) {
  "UNINITIALIZED_ERROR"
} else { 
  false 
}
/* Fails to compile!
<console>:29: error: type mismatch;
 found   : Boolean(false)
 required: String
         false 

*/
{% endhighlight %}

We can also easily simplify this whole block into a pattern match, which will read more clearly in Scala.  Fortuitously, Scala also lets us capture the return values from `match` blocks, following the same rules (including the Type Annotation concerns) as in if-else.

{% highlight scala %}
import annotation.switch

// Even better, cut out one more variable storage by calling our valid check inline
val status = (someMethodCallThatSetsValidity(): @switch) match {
    case 1  => "VALIDATED"
    case 0  => "INVALID"
    case -1 => "UNINITIALIZED_ERROR"
    case default => "UNKNOWN_ERROR"
}
{% endhighlight %}

*The use of the @switch annotation is a trick I learned from [Josh Suereth's](http://suereth.blogspot.com/) absolutely superb book [Scala in Depth](http://www.manning.com/suereth/); when used with certain matches (they typically need to match numeric values) they ensure the compiler generates a much more efficient JVM bytecode for the match.*

I'll leave the evaluation of that last as an exercise for the reader.  Fundamentally it is a simple restatement of our previous if-else blocks and should make sense.

Now that we've looked at branching block statements and value capture in Scala, we're left with one last item — for constructs (We aren't going to discuss monad operators such as `foreach`, `map`, `flatmap`, etc here). 

{% highlight scala %}
for (x <- 1 until 42) x
// Loops 42 times, but returns / prints nothing >
{% endhighlight %}

Unlike other the previously examined branching blocks (if-else and matches), Scala's `for` loops do *not* implicitly return the last value.  I would make an educated guess that this is because most developers would not expect or require the default behavior of a `for` loop to generate return values.  I'd also go so far as to posit that this is also a very sane default. 

Much like Python, to generate a return value from a `for` loop in Scala we must explicitly declare our intention to return a value. While (as I lamented previously) Scala lacks support for Python style generators, it does support a form of List Comprehensions which allow us to get the behavior we want.  For a Python developer, the rules for constructing these in Scala may get quickly confusing.  The first is that `return` is not a valid keyword at *all* inside of a Scala `for` statement (Incidentally, Python allows it as a 'valid' keyword, but the function will always return the result of the first loop iteration and never progress).

{% highlight scala %}
for (x <- 1 until 42) return x
/* Fails to compile 
<console>:29: error: return outside method definition
       for (x <- 1 until 42) return x 
*/
{% endhighlight %}

Instead, we need to use a special keyword, `yield`, to signify our intention to create a List Comprehension rather than a simple loop.  Like in Python, `yield` changes the behavior of a `for` loop.  However, the behavior of the `yield` keyword in Scala differs significantly from Python, as it does not invoke generator behavior.  The use of `yield` inside a Scala `for` loop will produce results along the lines of those of Python's square bracket enclosed List Comprehensions:

{% highlight scala %}
for (x <- 1 until 42) yield x
/* scala.collection.immutable.IndexedSeq[Int] = Vector(1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42)
*/
{% endhighlight %}

The output of the `for` loop which uses `yield` will always be an *immutable* `Seq[_]`, letting us save that result for later usage much like we did with our Python generators and comprehensions.  And of course, Scala's `for` loops allow for filters just like Python's:



The intention here wasn't to "bash" on Java or even champion Python or Scala; rather, to highlight the ways that different syntactic features add power, flexiblity and (in the case of things like mutability) potential safety to our code.  Though I'm accused of being a Scala fanboy, I want to clearly reiterate generators as something that I still think Python gets as a *major* edge over Scala.  I work in all three of the highlighted languages on a daily basis and though I probably enjoy Java the least, find gems and strengths in each tool as I switch between them (to be honest though, most of Java's gems these days exist in the JVM and the JDK libraries rather than the language).

It may easily be said that I am biased, but I consider the improved functionality highlighted in Python and Scala to be significantly more powerful (and yet less complex in many ways) than Java's approach.  Mutating variables, null initializations and general spaghetti code for the sake of expressing something that should be simpler to express are all things that lead to workplace violence and big bonus checks for employees of straight jacket manufacturers. 

With our improved examples in Scala, we didn't need to create any mutable placeholder values, initialize anything to `null` *or* worry about ending up with a still mutable list to contend with.  What we got was a pure, unadulterated, cleanly constructed immutable `Seq`, just like grandmom used to bake! And it smells freaking *delicious*.

-b 
