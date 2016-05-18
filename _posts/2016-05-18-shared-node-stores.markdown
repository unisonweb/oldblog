---
layout: post
categories: [updates, design]
title: Shared node storage and more efficient distributed execution
post_author: Paul Chiusano
---

_First, a quick update: I'm back from paternity leave and I also just wrapped up some client work so I'll be focused almost exclusively on unison for at least the next several months. I'm super excited to make more rapid progress now that I can really devote my full attention to it!_

I been thinking a lot about the implementation of the [unison distributed programming API](/2015-06-02/distributed-evaluation.html). Until now, I've been assuming there's is a one-to-one mapping between nodes and node stores---each node comes with its own store. So, for instance, you might have a million nodes, and the identity function (which has a unique hash) might be stored separately at every one of them. On the one hand, storage is pretty cheap, but allowing multiple nodes to reference the same store lets us _highly optimize_ the communication protocol used in the distributed programming API.

Recall that when node Alice sends a computation to node Bob for evaluation, we need to make sure that Bob has all the hashes referenced by Alice's computation. This syncing step takes time and bandwidth, and though the hashes can be cached for next time, if we can skip this step entirely, even better. If two nodes can reference the same underlying store (and this is even made explicit in the protocol), then we get to skip this step, which means less bandwidth usage and also a greatly decreased cost of spawning new nodes as we don't need to pay a potentially large one-time syncing cost. I would much prefer that Unison nodes be cheap to create---the programmer should be able to create and manage nodes according to whatever logical decomposition of their program they think makes the most sense.

This insight is useful if you have a bunch of nodes running on a single machine that you manage, but it's also useful when building cloud services atop Unison. You might choose to have clusters of nodes in your service all share the same storage layer (perhaps atop a cluster-wide distributed key-value store), with the benefit of hugely optimizing any communication between these nodes.
