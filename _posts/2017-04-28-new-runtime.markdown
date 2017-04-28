---
layout: post
categories: [updates, design]
post_author: Paul Chiusano
title: Work on fast new Unison runtime
---

I'd like to get a first release of Unison out by end of 2017 or sooner. The release won't be a toy or proof-of-concept; provided you don't mind being on the bleeding edge of a new language, this first release should be usable for real work. The big things missing before that can happen are:

* Completed Unison language, typechecker, and codebase editor / "build tool"
* A "real" Unison runtime, which runs Unison code at hopefully close to native speeds
* Implementations of distributed programming API and other core libraries

The past couple months I've been doing R&D and tinkering on different implementation strategies for the runtime. The "hard" way of doing this is to write an interpreter in C, your own GC, eventually implement your own JIT (possibly with help of LLVM), and spend maybe 10 years before you have something that runs reasonably fast. Yikes!

The easier way is to just write an interpreter in some fast JIT'd, GC'd language, but write it in a peculiar way (using "partial evaluation" or "specialization") such that interpretation overhead disappears and code runs at near-native speeds. (See [The Three Projections of Doctor Futumura](http://blog.sigfpe.com/2009/05/three-projections-of-doctor-futamura.html) and [Truffle & Graal](https://blog.plan99.net/graal-truffle-134d8f28fb69))

There exist frameworks like Truffle & Graal (also RPython) that can yield very fast code but I was curious what kind of results I could get just by writing a manually specialized interpreter (in Scala), and then just relying on ordinary JIT'ing by the JVM to convert to efficient code. If it works, we get a fast runtime in a tiny amount of code, without needing to write our own GC or other runtime services, and without needing to write our own JIT. Besides that, you have tremendous flexibility---you can implement tail calls or coroutines even though the JVM doesn't provide these things out of the box, etc.

So how well does this work? From what I can tell: __really well__. Although I've only done some informal experiments, it looks like speed can approach Java straight line performance. For instance, in one test I did, I was seeing a while loop in Scala take 1 microsecond to sum up 1k numbers, and a partial evaluated interpreter take about 1.5 microseconds. (As a sanity check, 1 microsecond to sum 1k numbers would be 1 billion numbers per second on my 1.7Ghz CPU, which is very reasonable.)

I'll post when I'm further along and have more definitive results, but it looks like we could have a fast Unison runtime sooner rather than later!
