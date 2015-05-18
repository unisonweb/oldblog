---
layout: post
categories: [updates, language, typechecker]
title: Adding let blocks and setting the stage for further work on the type system
post_author: Paul Chiusano
---

First of all, thanks so much to [everyone who's already become a patron of the project](https://www.patreon.com/pchiusano)! ([More info](http://unisonweb.org/2015-05-11/funding.html#post-start))

Now, for a few updates:

[Dan Doel](https://plus.google.com/+DanDoel/posts) helped me out with implementing [`let` and `let rec` blocks](https://github.com/unisonweb/platform/pull/13). Dan and I worked together at Capital IQ on the [Ermine language](https://github.com/ermine-language) and I've been bouncing various insane ideas for Unison off him for a while. He's a great person to work with, super easy-going, and has much more experience than me reading about and implementing type systems. We got together in the evening last week after work, added `let` and sketched out how `let rec` would work (the changes to the typechecker are somewhat subtle). I later finished up an initial (buggy) implementation, which Dan was also kind enough to help me debug. Everything looks to be working now, so I'm going to declare victory!

We also both took a look at [the recent paper by Dunfield and Krishnaswami][gadts] showing how to extend [their earlier system](http://www.mpi-sws.org/~neelk/bidir.pdf) (which the Unison typechecker is based on) to handle GADTs, pattern matching, and existential types. The new paper looks more complex, but still doable.

[gadts]: http://unisonweb.org/2015-05-07/update.html#post-start

The [current Unison typechecker](https://github.com/unisonweb/platform/blob/master/node/src/Unison/Typechecker/Context.hs), including the `let`/`let rec` stuff just added, clocks in at 514 lines of code. Implementing the newer paper is probably going to increase that number, though [the authors claim it is "only modestly more complex than our earlier paper"][gadts] and it also would bring Unison's type system and feature set up to relatively modern functional language standards. I'm hoping to coax Dan to starting tinkering on a GADTs branch while I work on migrating the Unison editor away from Elm. (If you have any words of encouragement for Dan, please do leave a comment!)

### <a id="hashing"></a>A quick preview of Unison's nameless hashing

In Unison, terms, types, and type declarations are uniquely identified by a "nameless hash". It's "nameless" in the sense that what you call your variables doesn't affect hashing. So given:

```Haskell
id1 x = x
id2 y = y
```

Both `id1` and `id2` hash to the same thing and thus are considered the same term.

A bit more interesting is that hashing is normalized with respect to _linking_, so given:

```Haskell
id1 x = x
id2 = id1
```

Both `id1` and `id2` will have the same hash. That is, we hash the term as if all references have already been linked into the expression.

_Aside_: We could also convert to eta normal form before hashing. I'm not positive that's a good idea, but it would be simple to do.

Now, even more interesting is that hashing is normalized with respect to the _order of bindings in a let rec block_. In a `let rec` block, every binding can reference every other binding so the order of these bindings is not significant (except perhaps as documentation). For example:

```Haskell
-- version 1
v1 = let rec
  ping x = pong (x + 1)
  pong y = ping (x - 1)
in
  ping 0

-- version 2, notice order is flipped
-- and variables are named differently
v2 = let rec
  pong1 p = ping1 (p - 1)
  ping1 q = pong1 (q + 1)
in
  ping1 0
```

In Unison, [both these expressions have the same hash](https://github.com/unisonweb/platform/blob/master/node/tests/Unison/Test/Term.hs#L14), even though the names of `ping` and `pong` are different, we use different parameter names, and the order of the bindings is different. We preserve the order of the bindings only for display purposes. The hash is what uniquely defines the term, and all references to terms are by hash, not by name.

_Aside:_ Note that how `let` and/or `let rec` are _rendered_ in the editor is a separate issue from the language feature itself. We could render the bindings after the expression and the keyword `where`, above, below, in a tooltip on hover, whatever! The language feature is the same.

More generally, the hashing scheme has a nice way of dealing with _cycles_ of elements whose order is not significant (for instance, the order of constructors of a data type). I'll write up some technical documentation on Unison's hashing in a later post. There are some interesting implications for data types, because data types are sometimes identified by more than _just_ their structure. (Consider a sorted `Map` type, perhaps represented as a binary tree, but with additional invariants established and preserved by the functions which construct and operate on the tree.)

I hope this gives a little bit of a flavor for the Unison philosophy---anything which doesn't affect the meaning of an entity (be it a type, term, or whatever) should generally not be part of its identity. Instead, extra information belongs as metadata, separate from identity. This greatly simplifies many many aspects of the Unison platform, since we never have to worry about two entities with different meanings colliding in the namespace. Arbitrary data types and functions can thus be transmitted between Unison nodes without worry about whether the remote recipient might have conflicting definitions for some of the dependencies referenced by the data or function sent. Much more on that in future posts.

### Replacing Elm, project cleanup and restructuring

As I mentioned in [the last update](/2015-05-07/update.html#post-start), I'm going to be rewriting the Unison editor in something other than Elm. The major contenders were a [GHCJS][] solution and something in PureScript.

I've decided I'm going to _start_ by looking at [GHCJS][]-based options. Why? Well, the other major contender was [PureScript][], which has a promising library for doing UI stuff called [Halogen](https://github.com/slamdata/purescript-halogen). PureScript's type system is certainly powerful enough for my needs and the language seems really nice, but a non-Haskell language for the frontend isn't really my first choice. Having an all-Haskell stack means no code duplication between the node and editor, and I can do computations wherever is most convenient or efficient rather than always preferring to run things on the node server just to avoid having to duplicate code in the editor.

[GHCJS]: https://github.com/ghcjs/ghcjs
[PureScript]: http://www.purescript.org/

So, I've done some cleanup and restructuring of the code to prep for experimenting with a GHCJS-based implementation of the Unison editor. I've split the Haskell code into two subprojects, `shared/` and `node/`. The `shared/` project has a minimal set of external dependencies. Anything in `shared/` can be referenced by either `node/` or (eventually) `editor/`, and as a simple approximation, anything in `shared/` or it's external dependencies is something that could get compiled to JS. This forces me to be very explicit about adding code or dependencies to the editor, and ensures there's no unintentional mixing of stages. See [the README](https://github.com/unisonweb/platform/blob/master/README.md) for more details.

Splitting up the dependencies like this still doesn't quite go as far as I'd like, though, because it's pretty difficult to reason about what the dependencies are of external libraries, like stuff in base. For instance, I don't know if this is still true, but at one point [`Integer`](https://news.ycombinator.com/item?id=5951038) was/is used for the `Show Double` instance (for some reason). It seems insane to compile all of `Integer` to JS for whatever silly thing is being done in `Show Double`. The user experience I'd prefer is to have to _whitelist_ all the modules (and perhaps functions in these functions) getting compiled to JS, and get a _compile_ error if I make use of something that transitively references anything not on the whitelist. This way I completely understand everything getting compiled to JS, and avoid accidentally doing something silly.

The issue isn't just code size---I want to know what code is ending up in JS, especially since there's something of an impedence mismatch between JS and the usual Haskell compilation targets and I may want to audit the result to make sure it's what I generally expect. This isn't really a worry when native compiling, since the Haskell runtime is (unlike JS) very much designed to efficiently execute Haskell code.

Anyway, I'd be curious how other people deal with this who have more experience with the GHCJS ecosystem.

One last update: the project also now has unit tests, made using [tasty](http://documentup.com/feuerbach/tasty) which generates nicely colored, hierarchical test output. Assuming you've gone through the instructions in [the README](http://documentup.com/feuerbach/tasty) the tests can be run from the `node/` directory via `cabal run tests`.

For people who are interested in helping write some code for Unison, I'm sorry I'm not very organized on that front yet. But my intention is to try to organize the work into some well-defined sub-projects that interested folks besides me could work on. Please feel free to come say hello in [the Gitter chat](https://gitter.im/unisonweb/platform).

That's all for now, more updates to follow!
