---
layout: post
categories: [design]
post_author: Paul Chiusano
title: An API for distributed evaluation and building distributed systems in Unison
---

This post describes the design of an API for distributed evaluation of Unison programs, and for describing, deploying, and executing distributed systems written in Unison. The API I'll propose here is simple, powerful, and quite easy to implement given the assumptions made by Unison. (I would not be surprised if an initial implementation took less than a week.) I welcome feedback on the design, especially if you can poke holes in it! Questions are also very much welcome. _Disclaimer: none of this is implemented just yet, this is a design writeup._

In Unison, since everything is uniquely identified by nameless hash, there are never naming conflicts in sending a term, type, or runtime value from one Unison node to another (ignoring astronomically small chance of hash collisions for now). Languages with a global symbol namespace (just about every language) have more difficulties---in order to send a value from one node to another, we either need to come up with some ad hoc encoding for the particular sorts of values we'd like to send (a major source of boilerplate), ensure that both nodes agree on the meaning of each symbol being referenced in the global namespace (doesn't scale and leads to dependency hell), or apply some transformations to our programs to avoid ever having to send anything other than a small set of "ubiquitous" primitives whose meaning all nodes agree upon.

This last approach is pretty interesting. [Olle Fredrikson et al. have worked out the details](http://www.cs.bham.ac.uk/~drg/papers/ifl14.pdf) of how to evaluate arbitrary higher-order functional programs on a collection of nodes, _without ever sending a function between nodes_. You might be amazed this is even possible. But a little though reveals that this approach, while fascinating and probably quite useful in some contexts, will sometimes result in a huge amount of network communication. Consider the function:

```Haskell
map : (a -> b) -> Foo a -> Foo b
```

In evaluating `map`, we probably want the implementation of `map` for `Foo a` and the `a -> b` to live on the same node. (I've used `Foo` here to stand in for some data type that may not be "ubiquitous" across all nodes.) As Fredrikson's work demonstrates, we could still evaluate `map f foo` by bouncing evaluation back and forth between the node that has the function and the node that has the definition of `map` for `Foo`. But that's going to be pretty slow and involve lots of network traffic. The constraint of not being able to serialize arbitrary functions is quite limiting.

So, I'm led to the opinion that moving away from a global symbol namespace is the nicest choice to make here. Identifying entities by a hash of their content sidesteps lots of complexities and gives us a trivial, fully-generic way of sending values (and all dependencies of these values) between nodes. Let's start with this assumption and then attempt to build a nice API from there.

### The API

All right, so what's the API? Here's a super simple proposal:

```Haskell
type Node -- opaque

-- send the value to the remote node for evaluation
at : Node -> a -> Remote a

-- obtain a reference to the "current" node
here : Remote Node

unit : a -> Remote a
unit a = bind (loc -> at loc a) here

-- send the function to the remote node and apply it to the result
map : (a -> b) -> Remote a -> Remote b
map f a = bind (a -> unit (f b)) a

bind : (a -> Remote b) -> Remote a -> Remote b
```

Notice that `at` has no restrictions whatsoever on the `a` type. The `a` could be a function with a million transitive depedencies, it might close over free variables, as in `x -> at loc1 (foo x)`, whatever. _There are no restrictions._ Also notice there is no specification for how `a` values are encoded on the wire. The Unison platform provides a straightforward encoding of all possible values in the language (a bit like what one might get from a `deriving` clause in Haskell, except it works for _arbitrary functions_ too). This might seem like a limitation ("what if I want a different encoding?"), but it's not. More on that later in this post, though.

This API is missing something--the ability to specify that evaluation may proceed in parallel. We can fix that:

```Haskell
type Future a -- opaque

-- Like `at`, but returns a running `Future`
at' : Node -> a -> Remote (Future a)

-- Wait for the result of the future to become available
await : Future a -> Remote a
```

Thus multiple evaluations may be spawned at different nodes, and their results collected. There's no error handling, and that is fine for now. The only form of error recovery allowed in `Remote` will be functions like:

```
-- try evaluation at the first node, then the second node, until one succeeds
fallback : List Node -> a -> Remote (Future a)

-- try evaluation on each node, return first result, cancel the rest
race : List Node -> a -> Remote (Future a)
```

So we can recover from errors, but only to repeat the same computation. Or we can run duplicate copies of a computation. (You might be able to imagine some other useful functions, like `quorum`.) But we can't directly catch an error and then make a decision based on this fact. The idea here is that a `Remote a` value is just like an `a`; the distributed evaluation is just an "implementation detail". Later we'll introduce the type `Remote! a`, which includes some extra capabilities.

This API is perfectly suitable and it lets us express distributed, parallel evaluation of pure expressions.

#### Detour: algebraic effects

Using monads for tracking the `Remote` effect works fine, but it's a bit of a sledgehammer. I'm not the first person to observe that it's rather unfortunate that for one monad (the `Identity` monad), we get very nice syntax, whereas if we select a different monad (say `Remote`), we pay a heavier syntactic burden. It's not just about syntax, though. Real programs will often involve multiple effects, and monad stacks force us to pick a nesting of effects (which can impose some plumbing code) even in cases where the order of nesting is not significant. There are various ways of making the API for this nicer (the mtl classes certainly help), but none of these seem as elegant as just directly supporting _algebraic effects_ in the language.

If we added algebraic effects to Unison (something I'd like to explore), the syntax becomes lighter:

```Haskell
-- hypothetical syntax, completely made up
at : Node -> {e} a -> {Remote .. e} a

-- foo "hi" (at node1 (1 + 1))
```

And we just use ordinary function application to work with remote values, rather than monadic syntax everywhere. Effects propagate as you'd expect, and you can use type signatures to control what effects you'd like to allow in different parts of your programs. Conor McBride's nice work on [the Frank language](https://personal.cis.strath.ac.uk/conor.mcbride/pub/Frank/) has worked out a lot of the details, and I look forward to stealing ideas from it! There's also a lot of other good work on algebraic effects. More on all that in another post.

Monadic effects do have a lot going for them:

* They require no fancy type system features (not even typeclasses are really necessary)
* They are completely explicit. One can inspect a chunk of code and understand what effects are present and how they are handled, and it's very 'syntactic'. Some people like this. (I kind of like this aspect, or I suppose I have gotten used to it!)
* They are proven to work. Haskell programmers have been doing just fine with monadic effects (though there is something of a [cottage industry of figuring out encodings for algebraic effects in Haskell](https://hackage.haskell.org/package/effect-handlers)). Then again, lots of things are "proven to work". That doesn't mean they are the best solution or that we shouldn't consider alternatives!

But monadic effects impose enough of a tax that Haskell programmers often don't bother with making effects more fine-grained even in cases where it might be otherwise beneficial. Doing so imposes some plumbing code and worse syntax (compared to pure code), and so people tend not to do it unless there are other significant benefits. (So, we still have `IO`, which includes literally everything you could possibly do in Haskell, including read and write access to the file system and the ability to launch the missiles.) The ideal type system eliminates barriers to making types as precise as the programmer finds useful.

I'll continue using monadic syntax for the rest of this post for clarity.

### Implementation

The protocol between nodes is not complicated. Here's a sketch:

__Sender:__ I'd like you to evaluate the expression `#asf234j23 42 "hi" #2349asfGjs`.

__Recipient:__ Hmm, I'm missing a definition for the hash `#2349asfGjs`, but have all the others.

__Sender:__ All right, you might be missing dependenices of `#2349asfGjs` as well. Here's the full transitive set of hashes that `#2349asfGjs` depends on. Do you need any of those, too?

__Recipient:__ Er, yes, I also need hashes `X`, `Y`, and `Z`.

__Sender:__ Okay then, here's `X`, `Y`, `Z`, and `#2349asfGjs`. You now have everything you need to evaluate that expression.

__Recipient:__ _does evaluation_

The fast path is if the recipient has all the needed hashes, but the negotiation of what hashes are needed can be done quite efficiently.

Notice that the implementation is completely sessionless. We send a request to a node and get back a response. It's just like calling a regular function, except the evaluation may occur remotely, and we of course have types to track what's going on!

### A lower-level API

There's a lower-level, more imperative API that can be used to implement the above API and probably many others. In this API, we expose the actual actions of sending and receiving messages. Our expressions are no longer going to be pure in the same way as `Remote`, so we'll introduce a new effect type:

```Haskell
type Remote! a -- opaque
type Err = Timeout | Msg Text

remote! : Remote a -> Remote! a
```

So `Remote a` is strictly less powerful than `Remote! a`. Let's look at what else can we do inside of `Remote!`:

``` Haskell
-- create a communication channel
channel : Remote! (Channel a)

-- send a value to a node along a channel, unreliably
send : Node -> Channel a -> a -> Remote! ()

-- send a value to a node along a channel,
-- wait for acknowledgement of delivery
deliver : Node -> Channel a -> a -> Remote! (Future ())

-- wait for a value to arrive on a `Channel`
listen : Channel a -> Remote! (Future a)

-- kill a `Future` if it does not complete in the given interval
timeout : Duration -> Future a -> Future a

-- Obtain the value of a `Future`, catching exceptions
await! : Future a -> Remote! (Either Err a)
```

Let's look at an example of using this for a simple client-server program:

``` Haskell
prog : Node -> Node -> Remote! ()
prog node1 node2 = do
  c1 <- channel
  c2 <- channel
  client <- remote! (at' node1 (clientLogic node2 c1 c2))
  server <- remote! (at' node2 (serverLogic node1 c2 c2))
  await client
  await server

clientLogic : Node -> Channel Text -> Channel Number -> Remote! ()
clientLogic peer c1 c2 = ...

serverLogic : Node -> Channel Text -> Channel Number -> Remote! ()
serverLogic peer c1 c2 = ...
```

Something deliberately missing from this API is any sort of deallocation of `Channel` values. A `Channel` is a logical entity, and creating one just conjures up a GUID _without allocating an OS socket or other resources._ This is extremely important. For this API, the Unison node listens on just a single port, regardless of how many logical channels we're working with! Messages come in, tagged with a corresponding channel. If there is a call to `listen` registered for that channel, we deliver the message to the listener. A message with no corresponding listener is dropped immediately. There is no buffering of messages at this level, though that is easy to build atop this API. If the sender has used `deliver`, they will be notified of failure (or success) of message delivery. _Since we aren't ever grabbing a hold of any scarce OS resources like network connections, there's no finalization logic and the GC takes care of everything._

How... refreshing! Doing resource-safe I/O is hard, and there's a cottage industry of rather complex libraries for achieving it (I am an author of one such library in Scala). Can we solve these problems? Yes. But rather than solving hard problems associated with how to safely work with scarce resources, we can sidestep this complexity by _avoiding reliance on scarce resources in the first place._ Is this cheating? No. We're building a high-level API. Nothing forces us to import as-is whatever APIs the OS provides. Start looking more closely at how computing currently works and you'll quickly realize that many of the assumptions we take for granted are either arbitrary, or inherited from obsoleted machine constraints from the days when "640k ought to be enough for anyone". As Steve Jobs remarked ["everything around you that you call life was made up by people that were no smarter than you"](https://www.youtube.com/watch?v=UvEiSa6_EPA).

_Aside:_ I believe this API requires using weak pointers for the `listen` and `deliver` functions, since we want to deregister from the server loop if the returned `Future` values become garbage. I suspect variations on this API are possible that don't require using weak pointers in the implementation.

Another note about this API is we use _regular Unison code_ to deploy and launch our distributed program. (Notice we're using the `at'` function defined earlier.) This is preferred to the usual approach of using one language for writing the logic running on our nodes, and then an ad hoc configuration language/system (or collection of batch scripts!) to actually get our logic to run on a cluster of nodes! Deployment can be a complex task and like any complex task we have to describe with software, we owe it to ourselves to create our descriptions using real programming languages with powerful type systems.

The APIs I've given here are very simple to implement in Unison. Picking the right foundational assumptions to build on makes all the difference.

One final note---I mentioned the previous API could be implemented in terms of this lower-level one. I'll show how that works at the end of this post.

### Dynamic clusters

Something I've been thinking about is extending these APIs to handle dynamic clusters of Unison nodes, like what one gets access to on the various cloud computing platforms like AWS, etc. This needs more design work, but imagine having access to an infinitely scalable, dynamic cluster of nodes, easily provisioned on demand:

```
type Cluster -- opaque

-- obtain a node from the cluster
provision : Cluster -> Remote! Node

-- functions providing useful info for configuring
-- network topology
ping-time: Node -> Node -> Remote! Duration
load : Node -> Remote! Percentage
```

This is just meant to be suggestive, there are many details to work out (though not as many as you'd think). But think of the possibilities! An elastic cluster of nodes, which you can provision on demand to execute enormous distributed computations over massive data sets, as easily as you might write a sorting function over some local data sitting in memory!

Exciting stuff, and Unison will get there.

### <a id="security"></a>Security, privacy, and node-specific code

These APIs seem pretty handy, but what about security? If we are accepting thunks to evaluate from any other Unison node (or a nefarious attacker impersonating a Unison node), how do we protect ourselves from attackers running code that deletes our home directory or takes over our machine?? If Unison nodes were running arbitrary C code, this would be a difficult problem. But Unison is a purely functional language, and running pure code is always safe. The worst pure code can do is fail to terminate, which can be addressed just by giving foreign expressions being evaluated a time budget for evaluation.

All right, but sometimes we will actually want to allow some evaluation of effects. For instance, a node may wish to allow another node to read or write certain values to node local storage, or allow sending messages to a certain other node along a particular channel. So more generally, the recipient node of a foreign expression should evaluate the expression in a sandbox. 

The way this works is quite simple. A sandbox is represented as a collection of hashes and/or builtin function references. Since Unison expressions are [linked _only_ at runtime](/2015-05-22/why-compile.html#post-start), it simply isn't possible for an expression to statically "bake in" the definition of some function it shouldn't have access to. The only place it can obtain definitions for functions and ways of evaluating builtins is at runtime, from the runtime environment, so it is trivial to provide access to a limited set of capabilities.

A node can have a sandbox associated with every other node that attempts remote evaluation. The permissions can be very fine-grained. Node _Alice_ might allow node _Bob_ to remotely evaluate pure code, _not including subtraction_. Or perhaps _Alice_ allows _Bob_ to read just a single number. But _Alice_ might allow node _Eve_ to evaluate _all_ pure code. More general policies are possible, here are just a few ideas:

* Allow only evaluation of pure code (with a small time budget) from arbitrary nodes
* Allow all capabilities for any node that can prove ownership of a private key
* Allow read-only access to certain parts of the node data store, for any node that can prove ownership of a cryptographic token. Imagine keeping personal information (like your address, phone number, allergies or medical history) on a Unison node that you control, and releasing it in this way to authorized third parties.

This last point is pretty interesting, and deserves its own post. But briefly: we can start to imagine a world in which you keep all _your data_ on a Unison node that you control. You perhaps allow businesses or other parties to run computations on subsets of your data, with access controlled by expiring cryptographic tokens. By restricting the capabilities you assign to different sandboxes, you can prevent remote code from "phoning home" and sending your personal information back to some foreign servers, or otherwise mucking with information it shouldn't have access to.

#### Node-specific computations, keeping implementations private

There's one last use case which is trivially enabled by this architecture, and that is sharing access to an interface, without granting access to the implementation of some code you prefer to keep private. As an example, suppose you're a car insurance company and want to let people get rate quotes, but you don't want the details of how you come up with a rate become public. Right now, you build a page on your "website" (yes, those are scare quotes) whose main purpose is for the user to supply the necessary arguments to your (secret) rate quote function, which you of course run on your servers and then generate a "page" which has some rendering of the result of calling your secret function. Isn't this a bit silly?

Instead, you can simply grant other nodes access to a remote version of your function:

```Haskell
rateQuote : Age -> DrivingRecord -> Rate
rateQuote age record = -- TOP SECRET!

rateQuote' : Age -> DrivingRecord -> Remote Rate
rateQuote' age record = at here! (rateQuote age record)

-- resolved by the editor statically to the current node
here! : Node
```

You now configure your node to only expose `rateQuote'` to other Unison nodes for syncing. Remember that `rateQuote'` is a proper Unison term, with its own unique hash, which can be trivially shared with other nodes. Any other node may obtain a reference to `rateQuote'`, which is a function that will evaluate the `rateQuote` function on the car insurance company's node.

This is useful for other things besides just keeping implementation details private. A Unison node may reference functionality _specific to the node_. Eventually, Unison will grow a FFI for including foreign functionality, and Unison can be used to expose a nice API for either hardware or sensors specific to a particular node. We may have a stripped-down Unison node running on some physical device (your refrigerator, say), with particular physical sensors. Access to these sensors can be exposed to Unison and accessed by other nodes. This also works for exposing access to a cluster or network infrastructure. Say you want to expose a cloud service with a nice API, and you are managing your own servers. Using the Unison FFI, you wrap your ecosystem in a Unison API, and expose various `Remote` functions, which other nodes in the Unison web can consume easily. By using a common platform, Unison, with a super nice API, you have much less work to overcome switching costs and network effects that benefit larger and more established cloud service providers.

And so on. None of this stuff is difficult. Starting with a functional language with typed effects makes for a much easier starting point for sandboxing than "arbitrary x86 assembly".

### What about customizing the encoding?

I mentioned earlier that functions like `at` don't specify how values are encoded on the wire:

```Haskell
at : Node -> a -> Remote a
```

The platform-provided encoding of Unison values is always a straightforward mapping from the data type to a binary or JSON representation of that data.  This might seem like a limitation, but it's not really. If you want an encoding with different properties, you simply use _regular Unison code_ to convert to a different type whose natural encoding matches what you want.

So, for example, if we want to send a list from one node to another, and the list is expected to contain long runs of duplicates that we'd prefer get run-length encoded, we apply run-length encoding before migrating the computation to another node:

```Haskell
do
  vals <- map RLE.decode (at mynode (RLE.encode vals))
  ...
```

Here, `vals` is a regular `List a`, we're on `mynode` ready to evaluate some further computation, but the list was transmitted to `mynode` in run-length encoded form. This approach is simple and flexible. And as long as the encoding of Unison values is straightforward and easy to reason about, it also offers very fine-grained control. Undoubtedly, there are plenty of patterns to be discovered about how best to organize code that needs to do these sorts of conversions, but it's all going to be pure Unison, and we have the full power of the language available to us!

For instance, the run-length encoding example above follows a simple pattern that we can wrap up in a helper function:

```Haskell
-- move a value to a `Node`, using `b` as the wire format type
atVia : Node -> (a -> b, b -> a) -> a -> Remote a
atVia n (encode,decode) a = map decode (at n (encode a))
```

Overall, this is a nice programmer experience---we get a default which covers perhaps 80% of the cases and requires no extra work to use, but also retain the option of overriding behavior. Any custom encoding is explicit, straightforward, and keeps around type information rather than resorting to working directly with raw bytestreams. (Of course, in principle, nothing stops the programmer from customizing their encoder as far down as individual bytes.)

A related question is how to handle _sharing_. If we have an expression like `let x = ... in (x, x)` the fact that both elements of the pair point to the same memory address isn't observable to pure programs (observing it and making decisions based on sharing info would lead to violations of RT). But if we are transmitting the `(x,x)` pair, we may (or may not) want to preserve this sharing information. (If `x` is small enough, encoding the sharing information may be unnecessary overhead.) So what should the generic encoder do? Always preserve sharing? Never preserve it?

Rather than make a decision that won't be appropriate in all contexts, we can put the decision in the hands of the programmer:

```Haskell
type Thunk a -- opaque

-- convert argument to thunk whose body uses let bindings to
-- make sharing information explicit
share : a -> Thunk a

thunk : (() -> a) -> Thunk a
-- or perhaps just `thunk : a -> Thunk a` if argument nonstrict
```

So, `share (x, x)` gets turned into `thunk (_ -> let x = (x,x))`. The `share` function might seem a bit dubious---it observes sharing to do its work, _but_ since we can't distinguish the result from the original argument (except via the size of the serialized form, which isn't observable to Unison expressions), RT is preserved!

Like any other value, we can send the `Thunk` between nodes, and the body of a `Thunk` created via `share` is encoded just like any other unevaluated let binding. When the thunk is forced by the remote node, it reconstructs the same sharing information. With this approach, we can add sharing information at whatever scope we deem appropriate (including the top level), and have very fine-grained control. If we want, we can define the helper function:

``` Haskell
sharedAt : Node -> a -> Remote a
sharedAt n a = map force (at n (share a))

force : Thunk a -> a
```

This will capture all sharing information and ensure it gets reproduced at the remote node.

### Appendix: Implementation via the lower-level API

Here's how the higher-level API can be implemented in terms of the lower-level one. I'll make the thunking explicit:

```Haskell
-- A request is a channel for the input thunk to evaluate,
-- and a channel for the output. We are existential, so
-- forcing the thunk is really all we can do!
type Request =
  forall a . Request { gate : Channel ()
                     , replyGate : Channel ()
                     , input : Channel (() -> a))
                     , output : Channel (Either Err a) }

-- all nodes have a known `Request` channel which they `listen` on in a loop
requests : Node -> Channel Request

-- Remote really is just a restricted version of `Remote!`
type Remote a = Remote (Remote! a)

rethrow : Future (Either Err a) -> Future a

at' : Node -> (() -> a) -> Remote (Future a)
at' n a = Remote <| do
  i <- channel
  o <- channel
  gate <- channel
  waiting <- channel
  send n (requests n) (Request gate waiting i o)
  handshake <- listen gate
  await (timeout (seconds 10) handshake) -- connection now set up
  response <- listen o -- set up response listener
  send n waiting () -- let remote know we have listener registered
  send n i a -- send the actual thunk
  unit (rethrow response)
```

Like most imperative code, there are a lot of ways to mess this up (I'm not sure the above is correct), but here's the general idea behind this protocol:

* First, we establish some communication channels with the remote node.
* We then wait for the remote node to let us know they are listening, and inform them that we are listening for their reply.
* Then the thunk gets sent and we wait for the result. This is all asynchronous, we don't literally block threads at any point.
* All nodes have a loop running which handle the other end of this protocol.

The low-level API is quite expressive, but the intent is that it gets used to build higher-level APIs like the one given at the start of this post. Not having to worry about serialization, plumbing code, or resource safety makes life much easier.
