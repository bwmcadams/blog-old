---
layout: post
title: "Path Dependent Types in Scala"
date: 2016-04-11 15:39:19 -0600
comments: true
categories: scala
---

Often confusing to newer Scala developers, "Inner classes" in Scala do not behave the way they do in many other languages. Thus, they can be perceived to be 'inconsistent' from expectations in their behavior. 

It isn't possible, given the following class definition:

```scala
class Outer {
  class Inner
  def put(inner: Inner): Unit = ()
}
```

To instantiate the `Inner` class directly:

```scala

scala> new Outer.Inner
<console>:12: error: not found: value Outer
       new Outer.Inner
           ^
```


Instead, we need an *instance* of `Outer` to create an `Inner` instance:

```scala
val outer1 = new Outer
val inner1 = new outer1.Inner
val outer2 = new Outer
val inner2 = new outer2.Inner
```

In other words, the *path* to an Inner instance depends upon the Outer instance. Note as well that there is no true relationship between `outer1.Inner` and `outer2.Inner`:

```scala
scala> outer1.put(inner1)

scala> outer1.put(inner2)
<console>:16: error: type mismatch;
 found   : outer2.Inner
 required: outer1.Inner
       outer1.put(inner2)
```

The `put` method on `outer1` will not accept just any arbitrary instance of `Inner`: It must receive one derived from its own instance. ”Great“, you may say, ”but where is this useful?”. In the end, a powerful use of Path Dependent types is creating a tighter coupling between two related classes, in which the compiler can help us avoid mistakes.

Imagine, if you will, a (relatively simple) model of `Airplane`, `Seat`, and `Passenger`:

```scala
case class Passenger(firstName: String, lastName: String, middleInitial: Option[Char])

class Airplane(flightNumber: Long) {
  case class Seat(row: Int, seat: Char)
  
  def seatPassenger(passenger: Passenger, seat: Seat): Unit = ()
}
```

Given two separate `Airplane`s, it should be impossible to accidentally sit on the wrong plane!

```scala
scala> val flt102 = new Airplane(102)
flt102: Airplane = Airplane@4b8d604b

scala> val flt506 = new Airplane(506)
flt506: Airplane = Airplane@4dc27487

scala> val kelland = Passenger("Mike", "Kelland", None)
kelland: Passenger = Passenger(Mike,Kelland,None)

scala> val mcadams = Passenger("Brendan", "McAdams", Some('W'))
mcadams: Passenger = Passenger(Brendan,McAdams,Some(W))

scala> val nash = Passenger("Michael", "Nash", None)
nash: Passenger = Passenger(Michael,Nash,None)

scala> val flt102Seat1A = new flt102.Seat(1, 'A')
flt102Seat1A: flt102.Seat = Seat(1,A)

scala> val flt506Seat21A = new flt506.Seat(21, 'A')
flt506Seat21A: flt506.Seat = Seat(21,A)

scala> val flt506Seat20F = new flt506.Seat(20, 'F')
flt506Seat20F: flt506.Seat = Seat(20,F)
```

If three members of the BoldRadius team are headed on journeys, but only two are boarding the same flight... what if we are so wrapped up in conversation that one of us tries boarding the wrong plane?

```scala
scala> flt506.seatPassenger(nash, flt506Seat20F)

scala> flt506.seatPassenger(mcadams, flt506Seat21A)

scala> flt506.seatPassenger(kelland, flt102Seat1A)
<console>:19: error: type mismatch;
 found   : flt102.Seat
 required: flt506.Seat
       flt506.seatPassenger(kelland, flt102Seat1A)
```

Excellent! The compiler catches that while Michael Nash and myself are headed out on the same flight, Mike Kelland isn't. When Mike tries to board the wrong flight... the type system prevents it.

In short, we can use the Path Dependent Type logic to prevent crossing over into runtime bugs – if we're smart about our domain design.

