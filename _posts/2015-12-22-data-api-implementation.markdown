---
layout: post
categories: [design]
title: How laziness brings good query performance without an insane black box optimizer
post_author: Paul Chiusano
---

I did a [writeup of the Unison persistent data API last time](/2015-12-13/persistent-data-api.html#post-start). After writing that I was feeling inspired and decided to do some implementation work to convince myself the API was implementable and could be made efficient. A lot has come out of that, and this is a writeup. This post has three parts:

* First, I'll explain the problems with SQL, and why Unison needs something much better.
* Next, I'll sketch the basic idea of how laziness ameliorates much of the need for fancy optimizers, by making the efficiency of operations much less order dependent and providing "fusion for free". This paves the way for vastly simpler query interpreters.
* Lastly, I'll give a sketch of an indexing data structure, which I'll call a _prioritized critical bit tree_ (or 'PCBT'). It's one of a class of structures which can provide reasonable indexing performance for _all dimensions simultaneously_. And it has some other interesting properties, like the ability to provide incremental, streaming views of _any ordering of a dataset_.

### The trouble with SQL

In SQL, one is presented with the abstraction of an unordered set of tuples, queryable in all sorts of ways. This is the elegant part. But as you quickly find out, only a small number of the ways of interpreting queries have any hope of being efficient (especially so with most existing database tech), and many queries that are expressible have no efficient execution plan whatsoever. There's then an entirely out-of-band concept of _indices_, which dramatically affect performance, but which aren't really explicitly part of the programming model. Instead, the programmer writes queries against this leaky abstraction and relies on an amazingly complicated _black box optimizer_ to hopefully come up with some sensible execution plan. This leads to a lot of back and forth of writing code whose effect on execution isn't clear up front, then submitting it to the optimizer and inspecting the query plan to make sure it was _what was intended_ and/or sensible.

On top of this we have some very dubious choices of syntax (it originates from back in the day when people thought high-level languages should "read like English", rather than have a consistent, compositional structure) and the fact that the language is entirely first-order and has almost no capacity for abstraction.

The amount of complexity this all generates is astounding and it certainly keeps a lot of people employed. In Unison, we are tossing out all this complexity, and giving programmers [a simple, predictable programming model for working with persistent data](/2015-12-13/persistent-data-api.html#post-start). By "predictable", I mean there is no query optimizer; how queries are interpreted is always completely transparent to the programmer. But by combining a few tricks, we make queries much less sensitive to how they are composed, so the programmer can generally assemble their queries in whatever order they prefer, while ensuring that the query still runs fast regardless.

Is this really possible? I think so. To give some indication of how these ideas are playing out, and how simple they can be, I have an (untested!) implementation of a query interpreter in about [250 lines of code](https://github.com/unisonweb/platform/blob/master/node/src/Unison/Runtime/Pcbt.hs), supporting filters, unions, intersections, joins, limit queries and sorting.

### Laziness to the rescue

When you do things strictly, it becomes very important for performance to do them in the correct order, or in an explicitly fused fashion. Suppose I have two lists, a very long list `a : [12,8,42 ..]`, and `b : ["alice", "bob", "carol"]`. I could filter `a`, then `zip` the result with `b`, or I could zip `a` and `b` and then filter afterwards:

```Haskell
filter (\(a,b) -> f a) (a `zip` b)
filter f a `zip` b
```

But, since `zip` generates a list which is the shorter of the two list lengths, with strict lists, it's very important to do the zipping first, so the output list gets smaller and the filter then has less work to do. With strict lists, one might also be tempted to try to manually fuse operations or write them using monolithic loops, at a loss of modularity.

With lazy lists, it doesn't really matter what order you do the operations in, because the `filter` and the `zip` are both being done incrementally, and are effectively fused. But although we get the same runtime performance as if we had fused the operations manually (modulo some constant factors), we still get to define our transformations in a compositional fashion, using lots of little functions that each "do one thing". Obviously this is not a new idea and I'm not claiming any credit for it.

In the world of databases, we have a similar problem, in that query performance is very sensitive to how queries are executed. For instance:

```Haskell
(huge join tiny) join huge2
(huge join huge2) join tiny
```

Assuming `huge` and `huge2` both come with huge result sets, and `tiny` is just a few rows, we definitely want to do `huge join tiny` first, which will generate a tiny result set, quickly, which can then be joined against `huge2`, also quickly.

The other grouping, doing `huge join huge2` first, requires a lot more work, and depending on how `huge` and `huge2` are indexed, might be extremely inefficient. If one or both of `huge` and `huge2` isn't indexed on the join columns, performance is going to be abysmal.

So we have the same sort of problem, and perhaps laziness can once again come to the rescue. But wait a minute, aren't databases already lazy about things? It's not like join produces a strict, in-memory list of rows. Conceptually, it usually produces a stream of rows, with a very definite order to these rows. But this isn't quite what we want. Operations on indexed tables do not (in general) produce another indexed object, they produce a stream _which is no longer indexed in the same way_ and thus supports _way fewer operations efficiently_.

_Aside:_ Anyone know of a database for which join and all other operations retain "indexability" as a general rule?

To see the difference, let's go back to lists for a minute. When you do `zip a b` for two lists, `a` and `b`, the result is also a list. Likewise if you `filter f a`, the result is a list. The algebra is _closed_---every operation produces a list, and as long as we stay within the world of lists, laziness buys us fusion for free.

For Unison, let's try exploiting the same principle. We'll define an indexing data structure, the prioritized critical bit tree (others in a similar vein might work too), and ensure that all query operations return a _lazily constructed_ instance of that same data structure. Thus the data starts out indexed efficiently, and all query operations retain indexing structure, but we lazily instantiate these structures to obtain the same sort of fusion we got with lists.

<a id="pcbts"></a>
### Prioritized critical bit trees (PCBTs)

Let's take a look at a candidate indexing structure. Consider the following set of strings:

```
abracadabra
abra1
abra2
abra3
abra4
c1abrax
c2abrax
c3abrax
ca1bray
ca2bray
ca3bray
```

In a trie, we have a tree in which the root of the tree branches based on the first character (which is either an 'a' or a 'c' here), all children of the root branch based on the second character, and so on.

In a prioritized critical bit tree, we generalize the idea of a trie. Rather than always visiting the characters in order, we visit the characters in _whatever order will tend to minimize the result set most rapidly_. For this example, we might first branch based on the first character, this roughly splits the result set in half:

```
abracadabra
abra1
abra2
abra3
abra4

c1abrax
c2abrax
c3abrax
ca1bray
ca2bray
ca3bray
```

If we're in the 'a' branch though, we don't want to branch based on the second character (which is 'b' for that whole group), instead we choose whatever slot has the _highest entropy_, in this case, it's index 4, which is different for each string in the group.

Likewise, when we're in the 'c' branch, we might branch based on the last character, the one which is either 'x' or 'y'. In the 'x' branch, we would then branch based on the third character:

```
-- for this group, branch on second character
c1abrax
c2abrax
c3abrax

-- for this group, branch on third character
ca1bray
ca2bray
ca3bray
```

Notice that we don't visit the characters of the input query in strictly increasing order. We might jump around in the input arbitrarily. The name "prioritized critical bit tree" comes from the fact that we inspect the most critical bits as early as possible in the search.

That's the basic idea. We can view the structure as a kind of automaton, optimally constructed to make evaluating certain queries efficient.

We can generalize away the notion of the "position" to a more general notion of a path. Here's an in-memory representation of the data structure:

```Haskell
data Pcbt p a
  = Tip (Maybe a)
  | Branch p (Pcbt p a) (Pcbt p a)
```

That is, we can either be at a leaf, which may have a result, or we may be at a branch, which tells us which path of the input to inspect. For simplicity, each path is assumed to point to a single bit (we can generalize this to other alphabets, at a cost of a more complex implementation). If that bit is zero, we take the left branch, otherwise we take the right branch. Notice that this easily supports wildcard queries, useful for doing range queries, among other things---just check both branches.

This data structure can support a lot of operations efficiently, by making clever use of the structure. Like some other ways of doing indexing, it effectively keeps the data indexed across multiple dimensions simultaneously.

<a id="z-ordering"></a>
It's instructive to consider how PCBTs compare to something like [Z-ordering](https://en.wikipedia.org/wiki/Z-order_curve) when being used to store a set of pairs. Here's an example dataset:

```
("abc", 11123)
("abc", 11124)
("abc", 11125)
("xyz1", 211961)
("xyz1", 211962)
("xyz2", 211963)
("xyz2", 211964)
```

In Z-ordering, we simply interleave the bits of the dimensions. If we form a trie based on that ordering, then the root of the trie will inspect the first bit of the first dimension, the second level of the trie will inspect the first bit of the second dimension, the third level of the trie will go back to inspect the second bit of the first dimension, and so on back and forth until either dimension runs out of bits. (For simplicity, I'm not considering any sort of Patricia/radix trie where bits are inspected in larger chunks.)

Now consider the query `(_, 211963)`. We are specifying a wildcard for the first dimension, and a particular value for the second dimension. In a Z-ordered trie, we need to search both branches at the root, since the query just has a wildcard for that first dimension. At the next level, we pick a particular branch based on the value `211963`. Then we're back to inspecting the second bit of the first dimension, and again we have to branch due to the wildcard.

Although the branches of the search get trimmed out as we reveal more bits from the dimension which is fixed by the query, there's still quite a lot of branching to the search.

In a PCBT, because we always inspect whatever bit is maximally informative, we end up branching a lot less and the search thus terminates more quickly. The first dimension has less than two bits worth of information (it only has 3 unique values), so we end up visiting it less often, and branching less as a result.

Now consider the query `("xyz1", _)`. Here Z-ordering seems to have an advantage, since it will examine bits from the first dimension sooner than the PCBT. The problem is that Z-ordering inspects _the wrong bits_. It inspects the bits of the first dimension in order, and the early bits don't provide much information! Beyond the 'x' character, it's not until we get to the end of `"xyz"` that we obtain any new information. Thus, while the PCBT visits the bits of the first dimension deeper in the tree, when it does so it picks _the most critical bits_ to inspect. It therefore "catches up" to Z-ordering in terms of the amount of information it gains about each dimension.

_Aside:_ I wonder if PCBTs asymptotically outperform naive Z-ordered bit tries for all queries, given some assumptions!

That is the basic idea. Some stuff I didn't discuss:

* How to do various operations on PCBTs lazily.
* How to handle backing a PCBT by a persistent store, especially in the presence of inserts and deletes.
* Various engineering / technical issues around making a PCBT fast. I'm not sure of many of these things myself, but they feel solvable.

If you'd like to take a look at the current implementation [check out this file](https://github.com/unisonweb/platform/blob/master/node/src/Unison/Runtime/Pcbt.hs). The thing that was most surprising to me was that it seems it's possible to define an incremental sorting algorithm on PCBTs which sorts the dataset _using any ordering_, in constant memory. Moreover, the sorting can be quite efficient by making clever use of the PCBT structure.

I welcome your comments! I'd be especially interested if people could point me to other indexing data structures or approaches, or give their thoughts on how PCBTs relate to other data structures. Although PCBTs are pretty cool, the key is just to have any multi-indexed data structure which can be generated and transformed lazily. I suspect there might be other data structures fitting that bill, possibly with more research and experience behind them.

_Thanks very much to everyone who has discussed this stuff with me, in particular to Ed Kmett for some fruitful discussions about various data structure choices._

### Appendix 1: Explicit query models

Something that is interesting about the PCBT is that it is parameterized on a query model. We can visit the bits in whatever order we want, depending on what queries we want to be most efficient, while still getting reasonable performance on other queries. We can build the PCBT adaptively, based on the data it contains and/or the queries run against it, but we could also expose to the programmer the ability to configure the query model quite explicitly.

A simple query model might consist of a `Histogram (Pattern a)`, a collection of patterns along with weights assigned to each. Unison could then provide a function like:

```
empty' : Histogram (Pattern a) -> Remote! (Data a)
```

for initializing a data object.

### Appendix 2: Efficient type-based search

One of the things the Unison editor requires is a reasonably fast way of finding potentially matching terms for a type. It turns out that PCBTs provide an answer here. We build a PCBT of terms, keyed by their types _and their metadata_. The paths of the PCBT will be paths into the metadata (like characters in the name) and/or paths into the type. Type variables correspond to wildcards. As the user types in the explorer, we do PCBT lookups to quickly narrow down the set of possible matches.

The PCBT can't do exact type matching (for instance if the same type variable is mentioned in two places), but it can provide a conservative upper bound on the set of matches. The actual typechecker then comes in to refine this much smaller set of possibilities to an exact set, by doing actual typechecking.
