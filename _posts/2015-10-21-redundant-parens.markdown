---
layout: post
categories: [editor]
title: Eliminating redundant parentheses
post_author: Paul Chiusano
---

I was thinking a little bit more about how to nicely layout expressions. In Unison, the user doesn't explicitly determine how expressions are laid out down to the level of individual line breaks and spacing. The layout engine handles most of this and the user only controls how [particular functions render when applied to arguments](/2015-08-05/dfos.html#post-start).

Consider the expression `foo (x + 1) y z`. The parentheses aren't inserted by the user, the layout function just knows that parens are needed around `x + 1` because the surrounding context has higher precedence. Likewise, if the expression needs to be broken to fit into a certain width, it will get displayed as:

```Haskell
foo
  (x + 1)
  y
  z
```

However, I realized something recently. Now that we're putting each argument onto its own line, why are we still bothering with parens? The indentation is enough to indicate grouping, so _the parens are now redundant_. Let's just do:

```Haskell
foo
  x + 1
  y
  z
```

This is really the same sort of idea as using indentation in place of `{}`-delimited blocks (or any other token-delimited block). To me it looks cleaner _and_ since the use of whitespace is guaranteed to match the structure of the syntax tree, it's also unambiguous---the expression `(foo x) + 1 y z` (which would not even typecheck) would not be laid out this way, it would get broken as:

```Haskell
foo x
+ 1 y z
```

But, if you're ever unsure about the structure, you can easily hover over any part of the syntax tree you want and use the [expand/contract navigation actions](/2015-09-17/directional-navigation.html) to see the structure explicitly.

One last remark: this nicely sidesteps the question of where to put the parens for an expression that is broken onto multiple lines. Suppose we have: `bar (foo (x + 1) y z) p q`, if we have to break the `bar` and `foo` calls onto multiple lines, where do we put the parens? How about:

```Haskell
bar
  (foo
    (x + 1)
    y
    z)
  p
  q
```

Hmm, maybe we should move that closing paren after the `z` over past the width of the widest line in the block? Or maybe we draw a big stretched out paren that matches the height? Or maybe put the parens in the middle?

```Haskell
bar
   foo
    (x + 1)
  ( y       )
    z
  p
  q
```

I don't like any of these options. Now check out what we get if we use the rule that a broken function call doesn't need parens around each argument:

```Haskell
bar
  foo
    x + 1
    y
    z
  p
  q
```

Readable and unambiguous! I'll need to make a tweak to the prettyprinting library to allow this, but other than that it's a simple change to make.
