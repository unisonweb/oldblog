---
layout: post
title: How would a hash collision be handled in Unison?
categories: [faq]
post_author: Paul Chiusano
---

Here's a question, what happens if there's a collision with the hashing scheme used by Unison. What would happen, and is this something we should worry about?

Currently, Unison is using SHA3-512. Assuming it's a good hash function, the odds of a collision are astronomically low---2<sup>-512</sup>, or about 10<sup>-154</sup>. To put this number in perspective, here are some much more worrisome low-probability events:

* [A cosmic ray corrupting the memory of the program while it's being run](http://stackoverflow.com/questions/2580933/cosmic-rays-what-is-the-probability-they-will-affect-a-program) (around 10<sup>-15</sup> events per second per byte of memory), a surprisingly regular occurence!
* [The earth being vaporized by a giant asteroid](http://www.killerasteroids.org/impact.php). Let's say extinction-level strikes every couple billion years or so.
* [All life in our corner of the galaxy being extinguished by a massive gamma-ray burst](https://en.wikipedia.org/wiki/Gamma-ray_burst#Rate_of_occurrence_and_potential_effects_on_life_on_Earth). GRBs apparently occur with alarming regularity in galaxies the size of ours, every 100k to million years, though they have to be directed toward us to have ill-effects.

All right, now that we're purely in the realm of sci-fi, here's what would actually happen in the event of a hash collision:

* If you create a definition in the editor that collides with but isn't equivalent to something in the code store, it would be possible to generate a notification, using a deep equality check. You might respond by making some trivial change like adding an unused let binding to the expression.
* If you are running a computation remotely, and the recipient has a colliding hash, this will likely result in a runtime error as the recipient will say "yeah, already got that hash", then link in the wrong definition at runtime! Sounds pretty bad, but we already have plenty of (much more likely) sources of errors when evaluating expressions remotely. For instance, the remote node could be a nefarious attacker that evaluates all expressions to `42` rather than running them. The remote node could also suffer from cosmic-ray-induced memory corruption. If you are evaluating computations remotely at some node, you presumably have some level of trust that the remote node will do so faithfully, or are making some accomodations for the fact that you may not fully trust the remote node (perhaps you are running the same computation at a few remote nodes and checking that they all agree, or perhaps you have some way of checking that the result is "reasonable" without having to do the full computation yourself). But, if you did start to get suspicious that a node were failing to run your computations due to hash collisions, you could actually track this down. It would be annoying to do so, but certainly possible.

More realistically, suppose that due to weakness in the hash function a nefarious attacker manages to find a collision, a definition `evilFunction` with the same hash as `innocuousFunction`. They need to get it to your node somehow. However, they have no way of doing that! The remote evaluation protocol won't accept a foreign hash if the local node already has that hash locally. And any unknown hashes accepted from foreign nodes are deemed 'provisional' and will be used only for evaluating the foreign computation, unless you explicitly decide to trust these hashes and promote them to be runnable in some other sandbox. The exact design of this is somewhat TBD, but the general idea is that you must opt in to each definition you want to trust. Foreign nodes cannot cause definitions to arrive on your node with a higher level of trust than you've explicitly assigned.
