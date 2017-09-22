---
layout: post
categories: [musings]
title: Mobility of computations obviates the need for session types
post_author: Paul Chiusano
---

_Note: [Session types](http://www.di.unito.it/~dezani/papers/sto.pdf) came up in some conversations I had at [Scala World](http://scala.world) this year._

That is, a protocol spread across two nodes A and B exchanging data over a session-typed channel can be more simply expressed (without fancy types) via a single computation that just hops between A and B during different phases of the overall protocol. No channels need to be involved in the programming model. This approach, making computations mobile, also scales to protocols involving arbitrary numbers of nodes (where session types fail or get even more complicated). That is, rather than thinking of the protocol as involving two _separate_ computations communicating over a channel, it's better to view what's happening as a single computation that involves multiple nodes.

The continuations at transfer points of this single computation would reflect a changing sequence of types as the computation moves along, but because we aren't forcing communication to happen over a single channel, these intermediate types are just "existential" and don't make their way into the programming model.

This also explains why session types don't come up much when doing single-node programming: in single-node programming, all values and computations are "mobile" and there often isn't a need to stick a communication channel between parts of your code.
