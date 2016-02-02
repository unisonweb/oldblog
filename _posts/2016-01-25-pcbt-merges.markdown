---
layout: post
categories: [design]
title: Easy snapshot isolation and PCBT merges
post_author: Paul Chiusano
---

This post has some notes on how to implement database inserts and deletes for [the indexing data structure discussed in the last post][PCBT-post], the Prioritized Critical Bit Tree (PCBT).

[PCBT-post]: http://unisonweb.org/2015-12-22/data-api-implementation.html#post-start

As background, you may want to review the data structure being used for indexing, [the PCBT](http://unisonweb.org/2015-12-22/data-api-implementation.html#pcbts). It's similar to a radix trie except that the positions examined along each path are not necessarily monotonically increasing; instead, we examine whatever position has _maximal entropy_ with respect to the set of values contained in the subtree of the structure. This raises a few questions:

* How do we express inserts and deletes on this structure, while preserving (or at least preserving in an amortized fashion) its key invariants?
* How do we represent this data structure on disk, in a way that provides good locality and ease of implementing inserts and deletes?

Let's look at the first question. The PCBT as described is a static data structure, a labeled tree with certain properties that enable lots of operations. How do we modify this structure? We could:

* Modify it _in place_ somehow. For instance, an insert could traverse the PCBT much like a lookup would, but then insert the value at the leaf where the lookup "should have" found the value being inserted. Over time this will tend to result in a structure where the bits are no longer examined in the optimal order. One simple proposal: rebuilt the PCBT from scratch each time it doubles in size.
  * It still leaves a lot of questions open. How exactly do we "rebuild the PCBT from scratch" on an index that's much larger than available memory?
  * For databases, in-place edits introduce a lot of complexity (like how do we achieve isolation between queries without doing locking everywhere, which is complicated, inefficient, and error-prone to implement).

Let's reject that approach. The other class of approaches is to use some form of [_dynamization_](https://en.wikipedia.org/wiki/Dynamization). To dynamize a data structure, we provide a _merge function_ for the structure, and a merge function for _results_ of queries over the structure. There are various schemes we can use for deciding when to merge the various _fragments_ (my term) of the structure, but as a rule of thumb we'll aim for having a sequence of fragments which _increase in size exponentially_. To do a lookup, we check the smallest fragment (which likely lives in memory), then next fragment, which is say double the size, and so on. In the naive approach we end up doing a logarithmic number of lookups, but it's possible to improve on this (for instance, by using a bloom filter to find the fragment which could contain the value).

Because we never modify a fragment once it is created, achieving things like snapshot isolation becomes easy---just keep around the old fragments. Assuming we have a fast way of doing merges, we can achieve very high write speeds, too, as there is only contention at the level of the "insert queue" rather than contention on updates to individual nodes in the data structure.

Anyway, this is not a new idea, and some storage engines (like [LevelDB](http://leveldb.org/), I'm told) use it to implement fast edits.

## Merging PCBTs

I've already worked out how to merge PCBT _results_ for most query operations like `join`, `intersect`, `union`, and so on. The implementations have kind of an obvious quality to them; there are generally two cases to deal with at each level---the case where both trees being combined examine the same position at their root, and the case where they differ in what position they examine. A little thought reveals what must be done in each case. See [the code](https://github.com/unisonweb/platform/blob/master/node/src/Unison/Runtime/Pcbt.hs) for details.

What about merging the PCBTs themselves? Let's look at an example data set, consisting of a set of bitvectors (we'll generalize this later):

```
011010011
00100110
101100000
111101001110
10
001
10001
100110100011
1100011010000
```

For each column, from `0` to the length of the maximum bit-vector, we compute the number of 1s, 0s, and neithers (for bit vectors not long enough) in that column. Call this the "summary vector". The summary vector takes up very little space. If there are `N` elements in the PCBT, with a maximum length of `M`, storing the summary vector even naively is only `O(M * log N)` (that is an upper bound; Θ-bound is tighter). We will choose to store the summary vector for each node in our PCBT, which it can be shown increases memory usage by at most a constant factor. (Note for instance that the leaf level summary vector contains all the information of the original bit vector, and is within a constant factor in size, so we can just store summary vectors if we wish.)

Given a summary vector, the root of the PCBT should be the position with "maximal information". One scoring function is to just multiply entropy by total count (total count is just 0s + 1s + neithers). If all bit vectors have the same length, this is just the position with maximum entropy. If vectors are of different lengths (as in the above example), positions receive a linear weight based on how many vectors even have each position. Thus, a position with exactly 50/50 1s and zeros but only 4 vectors gets a lower score than a position with 55/45 but a thousand vectors.

To merge two PCBTs, we combine their summary vectors, then use this to decide the root position. We then partition the full data set based on this position. The 0s all go in one set, the 1s all go in another. Notice this can be done streaming, and we can compute the summary for the 0-set and 1-set as we go. We then run the same algorithm on each subset, recursively.

That's the basic idea. Simple!

Something to be worked out still---there is almost definitely a way to leverage this structure to do the merge operation using more "block copies", rather than having to literally examine every bit of every element on each level of partitioning. Especially in the common case where the PCBTs are "well-aligned" and corresponding nodes examine the same positions in the bit vectors.

Also not worked out yet---we need a file format for this structure that can be built up incrementally but has good locality. This is not trivial.

### Merge schedules

Now that we have our merge operation, we just need a schedule for deciding when to do merges. A simple schedule is just to merge fragments whenever the size of the nth fragment exceeds half the size of the (n-1)th fragment. As in a binary counter, this occasionally leads to long chains of 'carries'. There are various ways of deamortizing this so that only a constant number of merges are done in the worst case for each insert. [Ed Kmett](https://github.com/ekmett) has [a few talks on cache-oblivious data structures](https://yow.eventer.com/yow-2014-1222/functionally-oblivious-and-succinct-by-edward-kmett-1701) (also [here](https://www.youtube.com/watch?v=6nh6LpcXGsI)) that discuss how to do this sort of thing nicely. But I'll leave this work for later.

### Appendix: extensions

* Deletions can be handled by simply marking elements as deleted. They contribute `-1` to the summary vector, and the partition function can strip these elements out as it goes.
* Generalizing, we can store data types which are not merely a single flat bit vector, but a collection of paths, where each path has an associated bit vector. This works out nicely for storing tree structures, like Unison terms or types. The idea for merging is the same, but instead of a summary _vector_, we have a summary _trie_, indexed by paths.
* It can be convenient to allow "wildcard" bits. That is, we might wish to store the vector `100**1*1`, where the `*` matches either a 0 or a 1. This is really useful for things like performing fast type-based lookup, like what the Unison editor has to do in the explorer. Some of the types we wish to store may have variables which can be bound to any constant.
