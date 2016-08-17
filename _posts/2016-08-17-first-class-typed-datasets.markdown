---
layout: post
categories: [updates]
title: First-class, typed, persistent datasets
post_author: Paul Chiusano
---

More progress. Right now we have a really minimal persistent data API which provides just a single type, a key-value store, called `Index k v`:

```Haskell
data Index k v

empty : Remote (Index k v)
insert : k -> v -> Index k v -> Remote Unit
delete : k -> Index k v -> Remote Unit
lookup : k -> Index k v -> Remote (Optional v)
...
```

Notice there's zero boilerplate or constraints on those signatures, you can persist and key on any Unison value at all... including functions or even another `Index` if you wish!

Also nice is that references to these `Index` values can be serialized and transferred to other Unison nodes. For instance, here's a program that spawns two Unison nodes dynamically, creates an `Index` on one node, then transfers the computation to the other node and does a lookup on that `Index`:

```Haskell
Remote {
  n1 := Remote.spawn;
  n2 := Remote.spawn;
  ind := Remote {
    -- Remote.transfer : Node -> Remote Unit
    Remote.transfer n1;
    ind := Index.empty;
    Index.insert "Unison" "Rulez!!!1" ind;
    pure ind;
  };
  Remote.transfer n2;
  Index.lookup "Unison" ind;
}
```

At runtime, an `Index` is represented by a `(Node, address)`, where the `Node` indicates the 'owner' of the `Index`, and `address` is some sort of address pointing into that node's local storage. Of course, we retain static type information, so you can't insert the wrong type of value or make a wrong assumption about the type returned by a `lookup`.

An interesting observation is that the various `Index` operations have to live in `Remote`---we can't have them exist in a purely local `IO`-like effect. The reason is that we may have transferred the `Index` to another node with a different local storage layer, and lookups _must_ remotely contact the originating node.

This also raises some interesting possibilites for nodes that have some data which is at least _readable_ by other nodes directly, which is good for read-heavy workflows. I'm tracking that [in this issue](https://github.com/unisonweb/unison/issues/92) as something to explore later.
