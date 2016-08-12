---
layout: post
categories: [updates]
title: A huge milestone - first distributed Unison program run successfully!
post_author: Paul Chiusano
---

As of [this PR](https://github.com/unisonweb/unison/pull/88), we got our first distributed program to run successfully. The program dynamically spawns two Unison nodes and ping-pongs a number back and forth between them until the count reaches 5. This exercises a huge amount of code - the `BlockStore` implementation that Sam has worked on, Arya's parsing code, the node container code I wrote which handles spawning of new local nodes, the distributed programming API implementation, which is responsible for automatically transporting the computation back and forth between the nodes, and lots more:

```Haskell
Remote {
  n1 := Remote.spawn;
  n2 := Remote.spawn;
  let rec
    ping i = Remote { 
      i := Remote.at n2 (i + 1); 
      if (i >= 5) (pure i) (pong i); 
    };
    pong i = Remote { i := Remote.at n1 (i + 1); ping i; }
  in ping 0;
}
```

Notes on syntax - `Remote { ... }` is an effect block, desugared much like a `do` block in Haskell, using calls to `Remote.bind` and `Remote.pure`. This isn't specific to `Remote`, a `Vector { ... }` effect block will desugar to calls to `Vector.bind` and `Vector.pure`, and we could allow arbitrary, dynamic `m { ... }` there (haven't bothered with this yet).

### Why is this awesome???

This short program is _deliberately missing a whole bunch of boring stuff_. You don't open up a socket or deal with communication channels in any explicit way, and you certainly don't write manual parsing or serialization code on your Unison nodes (unless you are interacting with non-Unison services). You simply _declare_ that you want the computation moved elsewhere, and the runtime makes it so. How... refreshing! Have a look at these two lines:

```Haskell
i := Remote.at n1 (i + 1); 
if (i >= 5) (pure i) (pong i);
```

Here, we transport the value `i + 1` to another node, `n1 : Node`. We are also transporting the _continuation function_ `i -> if (i >= 5) ...` to `n1`, which will be applied on the result of `i + 1` to continue the computation from there (which can terminate or bounce to the other node). The function or value being transported in this way could be a function with a million transitive dependencies; any that are missing will be synced as part of the inter-node protocol and cached for future use.

The other very nice thing here is that both nodes, _and programs that spawn nodes_ are _first-class values_. You can stash nodes in a list, pass them to functions, pass them to functions which are passed to other nodes, and so on. In this trivial program, we've hardcoded a couple locally spawned nodes, but perhaps you can see how easy it would be to abstract over that, just using the ordinary means of abstraction and combination of the Unison language. Now imagine writing a single Unison program which abstracts over its node provisioning, and can therefore be run either with purely local nodes (for testing), or on a massive distributed cluster!

Lastly, observe that _the Unison program describes its own node provisioning and deployment_. You don't write a program that describes just a single OS process, then use a bunch of ad hoc tools to provision machines, get your code to those machines, and run your programs. You describe, with a single, typed program, what computing resources are provisioned, and 'deployment' is just a trivial byproduct of the Unison inter-node protocol which syncs any needed definitions to the nodes you provision. In other words, provisioning, deployment, and orchestration can be talked about directly in Unison, in a first-class way, and you can _abstract_ over details and build reusable bits of logic, just like you would with any other interesting logic you have in your programs. Nice!!

### What's next?

Now that all this is basically working, we'll be focused on exploring what's possible with these APIs. We have a lot of nice ingredients:

* A reasonable, pure language for doing local computation
* The ability to spawn new nodes locally
* The distributed programming API which lets us contact other nodes and evaluate computations there (or spawn other nodes remotely)
* A persistent data API, which lets us trivially persist arbitrary Unison terms and look them up later

In case you missed it, here's v0 of the persistent data API:

```Haskell
-- Simple local, persisted key-value storage

data Index k v

empty :: Remote (Index k v)
lookup :: k -> Index k v -> Remote (Maybe v)
insert :: k -> v -> Index k v -> Remote ()
delete :: k -> Index k v -> Remote ()
```

Those are the actual type signatures - all Unison values can be persisted in this way, including functions! Just like in the distributed programming API, you don't deal with manual serialization or encoding to the database, or decoding on the other end.

Lastly, some other notable stuff in the PR that is of general interest:

* A number of additional builtins, including if, True, False, >, <, <=,==, >=. The comparison ops are specialized to numbers at the moment in the parser. In the editor, we could easily overload these symbols, even without typeclasses.
* Typechecker improvements - let bindings without free variables are generalized, for instance the following now typechecks: `let id x = x; y = id 42; z = id "woot" in z`, with `id` being applied at two different types.
* `Unison.Typechecker.Components` has an algorithm for minimizing let rec scopes as much as possible, using Tarjan's strongly connected components algorithm which [someone else was nice enough to already have implemented](https://hackage.haskell.org/package/GraphSCC-1.0.4/docs/Data-Graph-SCC.html). The container runs this algorithm on submitted programs. In conjunction with the improved let generalization, this means you can submit larger Unison programs to the container as a giant let rec block, building up whatever polymorphic functions you need for your "main" expression. Your helper functions / prelude will be generalized like you'd expect, and you don't have to worry about ordering your bindings manually.

More updates soon!
