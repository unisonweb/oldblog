---
layout: post
categories: [updates]
title: Creating lambdas in the editor, calling higher-order functions
post_author: Paul Chiusano
---

I've been doing some more work on the editor again. Here's a video showing a call to `map`:

<iframe src="https://player.vimeo.com/video/153777547" width="500" height="486" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

And here's another video, showing building up a multi-parameter lambda:

<iframe src="https://player.vimeo.com/video/153777546" width="500" height="475" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

When you type a symbol in the explorer, one of the options presented is creating a binder (either a lambda, a `let`, or a `let rec`) that introduces that symbol. This makes for a pretty fluid experience. In the first video, I could type:

    map  x  +  x  1  [_  1,2,3,4,5,6,7,8

Which creates the expression:

    map (x -> x + 1) [1, 2, 3, 4, 5, 6, 7, 8]

Notice that what I have to type is _fewer characters_ than what I have to type in a text editor! This is going to be very typical. We only have to disambiguate to the editor among possibilities that are well-formed and well-typed, and we don't have to worry about formatting the code---the editor lays out the code for us, inserts parens, and even omits the parents when [indentation is sufficient to indicate grouping](http://unisonweb.org/2015-10-21/redundant-parens.html) (notice at the end of the first video, when the two arguments to `map` flow onto separate lines, the editor omits the parens around the lambda).

The other thing is that it's not just less typing, the stuff that I do type is very close to what I would ordinarily type in a text editor, not some totally foreign set of keyboard actions that wlll take a while to commit to muscle memory. It feels "close enough" to raw text editing that I'm not put off by it. My only serious complaint about the fluidity is I dislike having to enter operators in prefix order, typing `+`, then its two arguments (Lisp programmers would have no trouble getting used to this). I have some ideas on how to improve on this as well, though.

I spotted a bunch of bugs while working on this. I'd like to get most of those cleaned up before posting the current version of the editor online for people to try.

### Update on persistent data API work

I have decided for the time being to put aside work on the full [persistent data API and implementation](/2016-01-25/pcbt-merges.html#post-start). While I think that work is promising and I can't wait to dig in (or find a volunteer interested in helping out!), I don't want to hold up the rest of the project to work on what is basically experimental database tech. Instead, I'd just like to expose a very simple binding to an existing key-value store like [LevelDB](http://leveldb.org/). This will independently useful and will unblock a lot of other use cases, like writing Unison-based 'apps'. I'll write up a Unison API for this in another post.

For the time being, I'm going to be focused on getting v1 of the editor 'done'. I'll consider it success when I can write some simple program, like mergesort or some Euler problems. Once that is in place, the other libraries can start getting filled in, like the persistent data API, the distributed evaluation API, and so on.

