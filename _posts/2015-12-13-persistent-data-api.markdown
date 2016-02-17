---
layout: post
categories: [design]
title: Initial sketch of the Unison persistent data API
post_author: Paul Chiusano
---

_Also see [part 2](/2015-12-22/data-api-implementation.html) and [part 3](/2016-01-25/pcbt-merges.html)_

I'm taking a break from work on the editor to do a design writeup. This post presents a design for the Unison persistent data API. With this API, users will be able to store _arbitrary_ Unison values, _including functions_, without having to deal with any parsing or serialization logic. Unlike SQL, this API doesn't awkwardly force the user to encode everything as tuples or named tuples---sum types (or any other type) are totally fine to persist and query for.

Though the API doesn't force awkward encodings onto the user, it's not dynamically typed or schemaless either. A `Data` object has a particular type, it's a `Data a`, which denotes a set of values of type `a`. Thus a "schema migration" isn't some dynamically typed SQL script that's run against a live database and might explode at runtime, it's an ordinary Unison function, statically checked using the Unison typechecker and trivially testable on a mock dataset.

All right, let's look at the API:

```Haskell
-- Create an empty `Data` value
empty : Remote! (Data a)

-- Add to a `Data` value
add : a -> Data a -> Remote! ()

-- Keep only values matching the pattern
keep : Pattern a -> Data a -> Remote! ()
```

_Note:_ The `Remote!` type constructor is a bit like an `IO` type. Its use is not really essential here, the key point is just that these functions have imperative effects. See [the distributed evaluation API](/2015-06-02/distributed-evaluation.html) for more info on it.

A `Data a` denotes a set of values of type `a`, and `empty` is the only function for creating `Data` values. It conjures up a globally unique identifier to serve as the first-class handle for referencing the data set. Unlike SQL, datasets are first-class; you can pass them to functions, store them in lists, and so on. Notice that the `add` function has zero constraints on it---any Unison value at all can be persisted in a `Data` object.

A `Pattern a` denotes an `a -> Boolean`, but the API for creating them is limited. I'll make the API completely explicit here for clarity, but there will likely be much nicer syntax for creating and viewing these patterns (or any other applicative functor) in the Unison editor:

```Haskell
negate : Pattern a -> Pattern a
literal : a -> Pattern a
blank : Pattern a
ap : Pattern (a -> b) -> Pattern a -> Pattern b
```

The `negate` function inverts a pattern---it fails whenever the original pattern succeeds and succeeds whenever the original fails. The `literal a` pattern matches any value with the same hash as the normal form of `a`---literally the same value as `a`. The `blank` pattern is `True` for any input. And the `ap` lets us build up larger patterns which may contain `blank` values. For instance:

```Haskell
bobs : Pattern (Text, a)
bobs = literal (,) `ap` "Bob" `ap` blank

lefts : Pattern (Either a b)
lefts = literal Left `ap` blank

-- nicer syntax/display -
--  bobs = ("Bob", _)
--  lefts = Left _
```

The `bobs` pattern will match tuples whose first element is `"Bob"`, and the `lefts` pattern will match `Either` values which are `Left`. Thus, there is no special handling of tuples, and no need to contort your data model to represent every concept using tuples. If you data model is best described by a `Data (Either a b)` or any other sum type, that's what you should be able to persist and query on!

Let's now look at querying:

```Haskell
query : Data a -> Results a

empty : Results a
add : a -> Results a -> Results a
keep : Pattern a -> Results a -> Results a
union : Results a -> Results a -> Results a
intersection : Results a -> Results a -> Results a
-- etc
```

The `Results a` is a pure description of how to fetch a result set of `a` values. These descriptions may then be combined in various ways to build up or refine a query. In the end, we often want to look at some values matching the query. Here's one way to do that:

```
ascending : Permutation a -> Ordering a
descending : Permutation a -> Ordering a
then : Ordering a -> Ordering a -> Ordering a

sort : Ordering a -> Results a -> Stream a
stream : Results a -> Stream a

take : Number -> Stream a -> Stream a
map : (a -> b) -> Stream a -> Stream b
-- etc

run : Stream a -> Remote! (Vector a)
```

A `Permutation a` denotes a permutation of the _paths_ into `a`, which generalizes the notion of a column. A simple `Permutation` might be `id : Permutation a` (the identity permutation, rearranges nothing) or a `Permutation (a,b)` (swaps the `_1` path and the `_2` path). A `Permutation` can then be converted to an `Ordering`, either ascending or descending, and orderings may be chained together using the `then` function. Together these functions let us express arbitrary ways of sorting ("sort by the third element of the tuple, ascending, then the first element descending"), the only constraint is that we must order based on information already present, at least when manipulating a `Results` object. (You have more flexibility after the `Results` is converted to a `Stream` or list.)

Once you're done manipulating your `Results`, you use `stream` or `sort` to convert to a `Stream`, which can be further manipulated and eventually `run`.

See [the appendix](#appendix) for details on how permutations are built up and some possible variations on this design.

Something that might not be obvious is that this API can be used to implement indices or key-value stores. Given a `Data (k,v)`, we can find the value(s) associated with a given key using the pattern `(k,_)`, or we can query in the other direction with the pattern `(_,v)`. In a future post, I'll describe a data structure that can be used to implement the actual store and which can answer all pattern queries efficiently, as well as supporting all the other `Results` operations like `union`, `intersection`, etc. The data structure comes in a few flavors, one of which is adaptive and tunes itself to efficiently run the most common queries while still being reasonably efficient for other queries---this covers the 80% case. But it can also be used in a completely explicit fashion, with the programmer specifying the "query model" which determines exactly how the structure is organized and what pattern queries are most efficiently handled.

## That's it

That's the API. Storing and querying for data should be simple, and shouldn't force you to work with weird representations of your data types or worry about parsing and serialization. As [in the distributed evaluation API](/2015-06-02/distributed-evaluation.html#what-about-customizing-the-encoding), you can of course customize how things are encoded just by _using regular Unison code_, before the data is persisted and/or after it comes out of the store.

Now why _isn't_ it this simple today? The reason: databases and languages are usually designed and implemented completely separately. There's a runtime barrier between the language and the database, and all static type information gets tossed out when crossing this barrier, just like going to and from text files or streams of bytes on a network socket. The programmer thus deals with parsing and serialization explicitly at this barrier, and there's also often an impedance mismatch since the database represents concepts in different ways than our programs (or can't represent some concepts well at all, like sum types). All this generates a huge amount of knock-on complexity. Taking a step back and addressing the problem in a cohesive way with better assumptions eliminates all that complexity. This is [a running theme around here](/2015-06-02/distributed-evaluation.html)!

_Also see [part 2](/2015-12-22/data-api-implementation.html) and [part 3](/2016-01-25/pcbt-merges.html)_

### <a id="appendix"></a>Appendix: bulk inserts, permutations, and joins

For bulk inserts, we can use the following API:

```Haskell
stream : Permutation a -> Results a -> Stream a
(++) : Stream a -> Stream a -> Stream a
map : (a -> b) -> Stream a -> Stream b
-- etc

adds : Stream a -> Data a -> Remote! ()
```

The `stream` function converts a result set to an ordered stream of values. We assume that `Stream` has a list-like API with `map`, `filter`, etc. The values of a `Stream a` can then be inserted to a `Data a`. This is one simple way of doing arbitrary "migrations", though of course it involves doing a full reinsertion of all the data from the original data set. See the discussion of permutations below for ideas on other options.

The `adds` function is also useful for synchronizing a data set with some external source---in this case the `Stream a` might be built from some periodic source of updates to a third-party dataset.

#### Permutations

Here's an API for building permutations:

```Haskell
id : Permutation a
compose : Permutation a -> Permutation a -> Permutation a
swap : Path a b -> Path a c -> Permutation a

id : Path a a
compose : Path a b -> Path b c -> Path a c
_1 : Path (a,b) a
_2 : Path (a,b) b
-- etc

swap _1 _2 : Permutation (a,b)
```

#### Joins

A similar idea can be used for expressing joins:

```Haskell
join : Join a b -> Results a -> Results b -> Results (a,b)
empty : Join a b
and : Join a b -> Join a b -> Join a b
align : View a x -> View b x -> Join a b

_1 : View (a,b) a
_2 : View (a,b) b -- etc
compose : View a b -> View b c -> View a c
```

And we can use a similar data type to build up strongly-typed migrations that can be interpreted efficiently by the data store.
