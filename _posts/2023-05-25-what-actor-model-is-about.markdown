---
layout: post
title:  "What Actor Model is really about"
date:   2023-05-25 22:42:42 +0200
categories: actors akka actor-model uppercut
---

TL/DR: read [1] and watch [2]:
1. [Actor Model of Computation](https://arxiv.org/pdf/1008.1459)
1. [The Actor Model (Hewitt, Meijer, Szyperski)](https://www.youtube.com/watch?v=7erJ1DV_Tlo)

During recent years "Actor Model [of computation]" (or simple "Actors") turned into a buzzword. But as usually happens with buzzwords, more "buzz" leads to less understanding of what it is actually about. There are dosen of implementations of the Actor Model in a variety of languages, most of them usually even make sense and ome of them are just masterpieces (Erlang VM, Akka). Heck, I even implemented one: [Uppercut](https://github.com/sergey-melnychuk/uppercut/)!

In this post I will try to structure my understanding and findings about Actor Model as well as best (and worst) use-cases for it. To start with, I just must warn that "everything is an actor" is just freaking wrong! I strongly encourage you to read the [paper](https://arxiv.org/pdf/1008.1459) and vatch the [video](https://www.youtube.com/watch?v=7erJ1DV_Tlo) if you want to achieve deep understanding of the topic once and for all.

Not "Everything is an Actor". Only Actor is an Actor. Message is not an Actor, it is a Message. Channel (the way messages travel between actors) is not an Actor, it is a Channel. Environment is not an actor... You got my point. So what is an Actor then?

> An Actor is a computational entity that, in response to a message it receives, can concurrently:
> - send a finite number of messages to other Actors;
> - create a finite number of new Actors;
> - designate the behavior to be used for the next message it receives.

Note that there is no mention of memory, network, threads, mailboxes, dead-letter-queues, supervisors, "let it crash" and so on. Just message passing & message handling. Note also that there is no such concept as a "reply" from an Actor, all communications between Actors are async and one-way. The whole point of Actors is:

> All physically possible computation can be directly implemented using Actors.

Simply put Actor model is about data-flow vs. control-flow. The control-flow is basically how modern hardware operates - there is an `IP` index that points to the next instruction. An (un)conditional jumps operate directly on this index to literally "control the flow" of execution. For example, a very simple branching instruction controls the flow to go either to address `[X+ N]` and call `do_this` function, or to address `[X+ M]` and call `do_that` function:

```
if x == 1 {     ## X+00: jne [X+ M]
    do_this();  ## X+ N: call [do_this]
} else {        
    do_that();  ## X+ M: call [do_that]
}
```

Same logical result in Actor Model can be achieved using adjusting data-flow:

```
handle(X):                  ## Actor message handler
  if X == 1 {
    this_actor ! DO_WORK(X) ## The work item is sent to `this_actor`
  } else {
    that_actor ! DO_WORK(X) ## The work item is sent to `that_actor`
  }
```

"So, Can I implement *it* using Actors?" - yes, of course you can. And this is the problem. Actor Model is not a silver bullet, and in practice Actors need to be run on physical hardware that most likely implements [Von Neumann architecture](https://en.wikipedia.org/wiki/Von_Neumann_architecture). This means that (likely shared) memory, netowrk and threads need to be dealt with.

If I condense my experience with Actor Model to a single sentence, it will be

> Actor Model is a good fit for event-based solutions only.

If a solution you're building is event-based (as opposed to what I call state-based), the Actor Model allows scalable and robust way to use message-passing for event processing. Just use tools like [Akka Streams](https://doc.akka.io/docs/akka/current/stream/index.html) to declaratively implement how exactly you want your events/messages processed and that's pretty much it! [Kafka Streams](https://docs.confluent.io/platform/current/streams/overview.html) is probably worth mentioning in this context as well.

From a concurrency standpoint, Actor is a synchronization primitive (Actor instance "owns" it's state in a mutually exclusive way and does not allow sharing it directly), similar to a `Mutex`. So if you want to implement Actors directly (and not use tools like Akka Streams), you better know what your are doing, because there is a good chance you don't.

If a business logic of a solution is better described in term of (relatively) small changes (events), then it is likely that it is going to benefit from applying an [Event-drived architecture](https://en.wikipedia.org/wiki/Event-driven_architecture). But if it is a regular RPC (REST/SOAP/etc), with relatively large traffic between peers (say up to couple of megabytes JSON sent from a server as a response to a client), then using Actor Model is going to be useless at best and maybe even harmful at worst.

I've seen a lot of troubles where Actor Model was misused, but probably the most harmful one is to block the thread while processing a message in a context of an Actor. All the Actors in the Actor System are likely being run on some kind of thread pool, thus blocking Actor thread pretty much partially blocks the capacity of the whole Actor System to make progress. Reasong for such misuse is likely a lack of understanding how specific actor framework works under the hood, as well as lack of knowledge how to operate with async primitives (e.g. `Future`s in an actor context). You don't need actors to implement request-response pattern! Just use any web-framework FFS! Check out [Interaction Patterns](https://doc.akka.io/docs/akka/current/typed/interaction-patterns.html) for more details.

Note on type safety. When dealing with Actor Model, a useful abstraction is so called "Location Transparency", meaning that the sender of the message does not know where the received would be (meaning the same host or another host, acessible over the network). So in order to send/receive messages over the network in a generic way, the only type available is `&[u8]` (or owned `Vec<u8>`). So the Actor Model is absolutely type-safe, if you know what I mean.

When desigining my own implementation of the Actor Model, I used a metaphore of a post office, and it worked out pretty nice. The Actors can communicate across the network (effectively implementing "Location Transparency"), and implement interesting protocols like [PAXOS consensus](https://github.com/sergey-melnychuk/uppercut/blob/develop/examples/paxos.rs) and [Gossip membership](https://github.com/sergey-melnychuk/uppercut/blob/develop/examples/gossip.rs). 

Learn more about Uppercut:
- [Uppercut Design Guide](https://github.com/sergey-melnychuk/uppercut/blob/develop/doc/design-guide.md)
- [Uppercut Mio Server](https://github.com/sergey-melnychuk/uppercut-lab/tree/master/uppercut-mio-server)
