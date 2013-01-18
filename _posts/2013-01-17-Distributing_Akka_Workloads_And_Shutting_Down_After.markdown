---
title: Distributing Akka Workloads - And Shutting Down Afterwards
tags: scala akka bestpractices
layout: post
---
Recently, as part of my role with the Professional Services team at [Typesafe](http://typesafe.com), I have been working on site at a customer who is using a lot of Akka and Play. During this time, I've gotten a chance to solve some interesting problems and answer obscure questions... which for those who like chasing these kinds of puzzles issues (like myself) is a fantastic way to spend the day (*and if this kind of thing sounds exciting to you, we're aggressively hiring for [this kind of work](http://blog.typesafe.com/send-akka-consultant-candidates-our-way-and-w) ;)* )

One item in particular came up recently as we tried to create a cron-style job to do interval data processing – big blocks of input data would be separated into individual instructions for processing, using [Akka 2.0.x](http://doc.akka.io/docs/akka/2.0.5/). The developer I was working with found that, among other things, using only a single actor to process all of their data items was not particularly performant. Further, once we solved this problem we couldn't figure out how to cleanly shut down Akka without interrupting any messages being processed. Fortunately, Akka offers simple answers to both of these problems... if you know where to look.

<!--more-->

By their nature, an Actor has a mailbox queueing all of the instructions sent to it, in order, and processes these messages one by one. In short, an individual actor is sequential, not parallel – performance is linear as we add more messages.

{% highlight scala linenos %}
package net.evilmonkeylabs.demo

import akka.actor.{ActorSystem, Actor, Props}
import akka.event.Logging

case class Message(msg: String)

class SimpleActor extends Actor {
  val log = Logging(context.system, this)

  def receive = {
    case Message(msg) => log.info("Got a valid message: %s".format(msg))
    case default => log.error("Got a message I don't understand.")
  }
}

object SimpleMain extends App {
  val system = ActorSystem("SimpleSystem")
  val simpleActor = system.actorOf(Props[SimpleActor], name = "simple")

  simpleActor ! Message("Hello, Akka!") // toss a message into our actor with the "!" send op
}
{% endhighlight %}

The obvious answer here is to spin up a *pool* of identical actors, all sharing the workload. While I seem to recall having to do a lot of custom work back in the early days of Pre-1.0 Akka, this is now *tremendously* easy to accomplish in Akka 2.0+, by the magic of [Akka Routing](http://doc.akka.io/docs/akka/2.0.5/scala/routing.html).

You may ask, just what is a Router in Akka? In simple terms, a Router is an Actor which proxies (and by its nature, [supervises](http://doc.akka.io/docs/akka/2.0.5/scala/fault-tolerance.html)) the mailbox for one or more child actors (Which I'll refer to as 'Routees' where possible), and routes messages to them with custom behavior. Akka provides a number of predefined Routers, and most of these are designed to have many child actors (aka 'routees') and forward a single inbound message to only *one* of these routees – though there are also several Routers which broadcast to all, including the very useful [`ScatterGatherFirstCompletedRouter`](http://doc.akka.io/docs/akka/2.0.5/scala/routing.html#ScatterGatherFirstCompletedRouter) (the use of which I'll cover in a future post).

In our case, what we wanted was several copies of the same actor, working together, but with a given message processed *only once* – so we aren't worried about Broadcast style routers for now. For this kind of task, there are two built-in router types that I would reach for personally: [`RoundRobinRouter`](http://doc.akka.io/docs/akka/2.0.5/scala/routing.html#RoundRobinRouter), and [`SmallestMailboxRouter`](http://doc.akka.io/docs/akka/2.0.5/scala/routing.html#SmallestMailboxRouter). The astute reader may also note the existence of a [`RandomRouter`](http://doc.akka.io/docs/akka/2.0.5/scala/routing.html#RandomRouter) – but my  background makes me somewhat wary of Random distribution of workload due to fear of [hot spotting](http://en.wikipedia.org/wiki/Hot_spot_\(computer_science\)).

The `RoundRobinRouter` and `SmallestMailboxRouter` are well suited for our needs here, so let's look at those. `RoundRobinRouter` sends the messages one by one through the list of routees – A, B, C, D then A, B, C, D again and so on. By contrasts, the `SmallestMailboxRouter` routes messages to the routee with the least messages currently in its queue, so that if one is running faster than others for some reason he can do some extra work. While this behavior is admirable for a more complex system, let's keep things simple in our example. We're going to use the `RoundRobinRouter` for these examples, as it gives us some predictable & well defined behavior to work with. Spinning up a router on top of an actor – and having it automatically spin up duplicates of that actor to route to - is a fairly straightforward process in Akka. We can leave our existing `SimpleActor` in place, and just change how we set it up.

{% highlight scala linenos %}
import akka.routing.RoundRobinRouter


object SimpleRoutedMain extends App {
  val system = ActorSystem("SimpleSystem")
  val simpleRouted = system.actorOf(Props[SimpleActor].withRouter(
                        RoundRobinRouter(nrOfInstances = 10)
                     ), name = "simpleRoutedActor")

  for (n <- 1 until 10)  simpleRouted ! Message("Hello, Akka #%d!".format(n))
}
{% endhighlight %}

Note the addition of a call to the `withRouter()` method on our `Props[ActorName]` declaration. Where a normal `Props[ActorName]` call sets up a single Actor, `withRouter()` will return us a Router with `nrOfInstances` child actors.  Here, we've setup a `RoundRobinRouter` with 10 routees; if we look at the output of running this new `SimpleRoutedMain`, we'll see our log entries have several different actor IDs in them:

{% highlight text %}
[INFO] [01/17/2013 15:32:51.897] [SimpleSystem-akka.actor.default-dispatcher-7] [akka://SimpleSystem/user/simpleRoutedActor/$f] Got a valid message: Hello, Akka #6!
[INFO] [01/17/2013 15:32:51.900] [SimpleSystem-akka.actor.default-dispatcher-8] [akka://SimpleSystem/user/simpleRoutedActor/$e] Got a valid message: Hello, Akka #5!
[INFO] [01/17/2013 15:32:51.900] [SimpleSystem-akka.actor.default-dispatcher-13] [akka://SimpleSystem/user/simpleRoutedActor/$d] Got a valid message: Hello, Akka #4!
{% endhighlight %}

With the previous example, we were very specific in our setup code – hardcoding the type of router we want as well as the number of routee actor instances. Hardcoding is rarely a good idea, and as such Akka also offers an easy way to make this externally configurable. We can change our router instantiation to read from the config instead:

{% highlight scala linenos %}
import akka.routing.{FromConfig, RoundRobinRouter}


object SimpleFileConfiggedRoutedMain extends App {
  val system = ActorSystem("SimpleSystem")
  val simpleRouted = system.actorOf(Props[SimpleActor].withRouter(FromConfig()),
                                    name = "simpleRoutedActor")

  for (n <- 1 until 10)  simpleRouted ! Message("Hello, Akka #%d!".format(n))
}
{% endhighlight %}

We've replaced our explicit instantiation of a `RoundRobinRouter` here with a call to `FromConfig()`, which tells Akka to find a matching configuration entry with the router setup details.  From here, we then just need to add an entry to our Akka configuration, in the `deployment` block, with the name we gave our Router:

{% highlight scala %}
deployment {
	/simpleRoutedActor {
		router = round-robin
		nr-of-instances = 5
	} 
}
{% endhighlight %}

Now we have an instance of `RoundRobinRouter`, spinning up and managing 5 identical copies of our `SampleProcessingActor` – and we can swap out the router type or even raise & lower the number of routees easily from our configuration. From our standpoint as a programmer, the `ActorRef` we get back from our router initialization is fairly transparent – messages we send to it get routed automatically to a routee, and replies can come back from those actors as well. This behavior is a boon for us, as it means we can begin sending messages to the router without worrying about any special instructions. 

Great! We now have a system for distributing our load. If we were feeling particularly adventurous, we could even combine routers with remote actors... but that's a different post, for another day.

## Cleaning Up After Ourselves 

Here's the part where we once again got stuck. Because we were building a cron job that was meant to run every once in awhile, do its work and then shut down, we found ourselves at odds with Akka's behavior. In order to enable it to run as a daemon and run over long periods of time processing messages at potentially unreliable intervals, Akka's [`ActorSystem`](http://doc.akka.io/docs/akka/2.0.5/general/actor-systems.html) starts up a pool of threads. This `ActorSystem` and its threads continue running – even after our `main` method completes and we expect exit.  For many types of applications this is ideal, as we want to run continuously; for a cron job however, we want to shut down when our work is done.

The first thought you have might be "Well, just throw in a `System.exit()` call!". Lest we forget, Actors are worked with asynchronously - we are not blocking while we wait for their actions to complete. We can demonstrate that quickly with a  block of code to interact with our Actors. Let's have our Actors print a message when they receive it, but also print as soon as our loop that sends messages *to* the actors completes.

{% highlight scala linenos %}
for (n <- 1 until 100)  simpleRouted ! Message("Hello, Akka #%d!".format(n))

System.err.println("Finished sending messages to Router.")
{% endhighlight %}
You might have expected a more sequential behavior out of this code, where everything went in order, such as:

{% highlight scala %}
Got a valid message: Hello, Akka #2
Got a valid message: Hello, Akka #3
Got a valid message: Hello, Akka #4
	...
Got a valid message: Hello, Akka #98
Got a valid message: Hello, Akka #99
Got a valid message: Hello, Akka #100
Finished sending messages to Router.
{%  endhighlight %}

Unfortunately, things don't work *quite* this way – the invocation to send a message to an Actor is asynchronous and returns immediately, not waiting for the Actor to process our message. Which means we probably saw our "Finished Sending" notification well before all of "Got a valid message" printouts. Herein lies our problem – if we force a `System.exit()` as soon as all of our messages are sent, we will shut down before the processing is done (especially if we are doing something involved like a database operation inside the actor). 

Similarly, if instead of System.exit, we were force the `ActorSystem` to shutdown, we would hit a problem. When the `ActorSystem` is shut off, Akka will not wait for all queued messages to be processed, and instead begins shutting all Actors down as soon as they finish their *current* message. Despite our progress with routers, this shutdown behavior is less than ideal for our purposes. Thankfully, there *is* a solution - but first, let's step back and take a quick look at how we shut down a single actor.

### Poisoning Actors (No, not Wallace Shawn)
<img src="/images/princess_bride_poison.jpg"/>

In order to facilitate the concept of "Finish what you are doing, and then shut down" with Actors, Akka offers `akka.actor.PoisonPill`. As a baked in, default behavior, all Akka Actors will automatically handle a `PoisonPill` message as an instruction to shut down. To use a `PoisonPill`, we send it to the actor *like any other message*. Because of this, it will enter the Actor's mailbox and only be processed when it is dequeued.  So if we load 10,000 "Do a task" messages to an Actor and then send a `PoisonPill`, we can rightly expect our tasks to complete before the Actor shuts down. This behavior is baked into the default receive handler of all Akka Actors:

{% highlight scala %}
      case PoisonPill               ⇒ self.stop()
{% endhighlight %}

Let's take a brief look at what happens when we use this `PoisonPill` with a single Actor, before taking a look at Routers:

{% highlight scala linenos %}
import akka.actor.PoisonPill 

object SimplePoisoner extends App {
  val system = ActorSystem("SimpleSystem")
  val simpleActor = system.actorOf(Props[SimpleActor], name = "simple")

  simpleActor ! Message("Hello, Akka!")
  simpleActor ! PoisonPill
  simpleActor ! Message("Boy, that was some tasty arsenic!")
}
{% endhighlight %}

If we run this code, we'll note that after the `PoisonPill` is sent additional messages sent to the actor disappear, as the target Actor has gone away. But what would happen if we tried this with Routers in play?

Unfortunately,  when sent to a Router, `PoisonPill` behaves quite differently from many users' expectations – as it is treated differently than normal messages to a router. Because of the way that default "Handle a `PoisonPill`" behavior is baked into all Actors (of which Routers are), Routers *do not* forward a `PoisonPill` to their routees, but instead take it as an instruction directed at themselves. 

This behavior can be surprising at first, especially because shutting an Actor down *also shuts down all of its children*, allowing the children only to continue processing their current message. Again, we find behavior contrary to what we might want.


### Broadcasting to Akka Routers 

Still determined to solve our shutdown problem, what we want to try now is ask each Actor that is routed to shut itself down *after its entire queue is processed*. A nice side effect of this is that when all of a Router's children shut down, the Router shuts itself down too. While the Routers we are currently using only route a message to a single Actor, it is possible to broadcast a message to all routees - using a special case class `akka.routing.Broadcast`.  When a Router receives a `Broadcast`, it unwraps the message contained within and forwards that message to *every Actor it is routing for*.

{% highlight scala linenos %}
import akka.routing.Broadcast

simpleRouter ! Broadcast(Message("I will not buy this record, it is scratched!"))
{% endhighlight %}

When running this code, we will see every actor in our Router setup repeat the message, "I will not buy this record, it is scratched!". Because the Router does not look at the message being broadcast once unwrapped, his trick works effectively for our task:

{% highlight scala linenos %}
for (n <- 1 until 10)  simpleRouter ! Message("Hello, Akka #%d!".format(n))
simpleRouter ! Broadcast(PoisonPill)
simpleRouter ! Message("Hello? You're looking a little green around the gills...") // never gets read

{% endhighlight %}

Great! Our Actors are now getting a correct command to shutdown, and allowing the Router above them to shut down gracefully too. We have timed this messaging to allow our full workload to complete before the shutdown, as well.

But... there's one more problem. If we look at our last block of code and run it, you might notice that the program *does not shut down*. This is because the `ActorSystem` remains running, and will not automatically shut itself down. We must instruct it to do so, but now we are back to our original problem – timing.

The best way that I have found to handle this problem is to take advantage of Akka's [Lifecycle Monitoring](http://doc.akka.io/docs/akka/2.0.5/scala/actors.html#Lifecycle_Monitoring_aka_DeathWatch), which allows us to create an actor who listens for notices that Actors have terminated. We need merely notify Akka that we'd like to hear about Terminations of a particular actor, and begin listening for those notices.

Since Akka will automatically shut down a Router when all of its routees have terminated, we should (rightly) expect a "Router Terminated" event soon after broadcasting a `PoisonPill` to our routees.

Here's a rough sketch of an "Overwatch" actor, who asks for Akka to watch two other actors (One our router, the other a simple actor we won't shutdown for), and when it sees the Router terminate, shuts down the `ActorSystem`:

{% highlight scala linenos %}
class SystemKillingRouterOverwatch extends Actor {
  val log = Logging(context.system, this)

  // Setup watching of our other two actors
  override def preStart() {
    context.watch(RoutedPoisonerWithShutdown.router)
    context.watch(RoutedPoisonerWithShutdown.simpleActor)
  }

  def receive = {
    case Terminated(corpse) =>
      if (corpse == RoutedPoisonerWithShutdown.router) {
        log.warning("Received termination notification for '" + corpse + "'," +
                    "is in our watch list. Terminating ActorSystem.")
        RoutedPoisonerWithShutdown.system.shutdown()
      } else {
        log.info("Received termination notification for '" + corpse + "'," +
                 "which is not in our deathwatch list.".format(corpse))
      }
  }

}
{% endhighlight %}

During the startup of the Actor (via the `preStart()` lifecycle method), we grab the references for two other actors and ask for Akka to `watch()` them. In the case that we see a `Terminated` message, which will contain an `ActorRef`, we compare the `corpse`'s body; if it is the Router, we shutdown the `ActorSystem`. If not, we can keep on going.

The body of `RoutedPoisonerWithShutdown` is just a tweak of what we've been building already, including poisoning an extra Actor to test termination, and setting up our overwatch:

{% highlight scala linenos %}
object RoutedPoisonerWithShutdown extends App {
  val system = ActorSystem("SimpleSystem")
  val router = system.actorOf(Props[SimpleActor].withRouter(FromConfig()),
                              name = "simpleRoutedActor")
  val simpleActor = system.actorOf(Props[SimpleActor], name = "simpleActor")
  // Start him after the others so their refs are available and he can grab 'em (lazy code)
  val overwatch = system.actorOf(Props[SystemKillingRouterOverwatch])

  router ! Broadcast(Message("I will not buy this record, it is scratched!"))

  simpleActor ! Message("If there's any more stock film of women applauding, I'll clear the court.")

  simpleActor ! PoisonPill

  for (n <- 1 until 10)  router ! Message("Hello, Akka #%d!".format(n))
  router ! Broadcast(PoisonPill)
  router ! Message("Hello? You're looking a little green around the gills...") // never gets read

}
{% endhighlight %}

Running this code, we'll see a notification that `simpleActor` terminated and we didn't care, followed by `simpleRoutedActor` terminating – to which we respond by shutting down the `ActorSystem`!

That's it; with a little basic knowledge we can now not only distribute our Akka workloads, but shut the system down cleanly when we are done with it!

*If you're interested in taking a closer look, I threw up a [repository in Github](http://github.com/bwmcadams/akka-router-shutdown-demo) with all of the code from this post*
