---
layout: post
categories: [musings]
title: The trouble with adoption of standards
post_author: Paul Chiusano
---

![Standards](http://imgs.xkcd.com/comics/standards.png)

_credit: [XKCD](https://xkcd.com/927/)_

Standards are a good idea. Getting people to agree ahead of time on an arbitrary convention makes it easier for systems to communicate. So why aren't they adopted? Here's the basic problem:

* Using a standard requires an explicit integration step which is (very often) extra work. For instance: the data exists in your database, and providing an API to access some of that data in some standard form is extra work. That's time your pointy-haired-boss would prefer you spend implementing MOAR FEATURES!!
* Standards often don't provide value on their own, independent of network effects.
* Before the standard is adopted, there are no real network effects from its use, so early adopters of a standard don't benefit directly. That is, they are a _cost_ for early adopters.
* Conforming to a standard can be a design constraint (another cost)
* Therefore, potential early adopters face the choice of 1) investing resources and paying costs for an uncertain standard with potentially no future or 2) waiting until it looks like the standard is being widely adopted enough for their investment to pay off.
* _And everyone knows this_, which generates [common knowledge](https://en.wikipedia.org/wiki/Common_knowledge_(logic)) that investment in integration with standards is often unlikely to pay off, and a negative feedback loop. (I don't have much incentive to integrate with a standard now, and I know everyone else has that same incentive, therefore I can reason that the standard is unlikely to be widely adopted, which means I _really_ shouldn't bother integrating with it now.)

Now here are some solutions. I'll use a running example: a standard for user accounts and logins.

* Eliminate or minimize the explicit integration step. For user accounts and logins, this might involve providing a nice library in a variety of languages for implementing the standard. Notice that building these libraries takes work. It's unlikely to just happen spontaneously.
* Piggyback on an existing platform, like Facebook or Google, or an existing internal company effort (like Twitter spearheading OAuth), or an otherwise well-funded platform.
* Provide value independent of network effects.
* Win over enough early adopters to provide enough network effects that the adoption snowballs. Example: get Google to implement your standard. Now everyone else is more likely to want to integrate with that standard.
* The brute force approach: pay early adopters. That is, raise a sum of money enough to offset the cost to early adopters. I've never actually heard of this happening in the wild.

Let's take a step back and rethink things a bit. _Why_ is there an explicit integration step? It's because the standard is specified in a form which differs from the _computable form_ in which we do our real work. Even worse, if a software system is proprietary, only the people with internal access to that proprietary system can do the work of converting from whatever computable form they use internally to whatever the form is that's dictated by the standard.

Compare this to the experience of working with ordinary typed values in your programming language of choice. The language and its type system provides a common platform for exchanging data and functions between different 'software systems' (pieces of code). Standards are 'just' shared libraries that multiple projects use, and adoption of these libraries doesn't face the same collective action problem as 'real' standards do because we are implicitly using some of the above 'solutions'. Even in cases where multiple libraries are developed for representing the same thing, in the same niche, the ecosystem tends to converge due to network effects. Importantly, when the code is open source, people other that the original authors can convert values to standardized form---it's just a regular function converting from one type or another!

The [solution being adopted by Unison](/2016-05-18/iot.html#post-start) is exactly of this flavor. Unison is a common, open platform. Standard extensions to it can arise and be integrated with trivially by anyone else on the web. The key is that we're extending what can be talked about in the language and type system---in Unison, we have first-class, typed values that can refer to large distributed systems, and so the ability of the language and type system to act as 'quasistandards' is expanded accordingly.
