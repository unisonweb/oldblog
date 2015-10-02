---
layout: post
categories: [editor, updates]
title: First round-trip connecting editor front-end to typechecker
post_author: Paul Chiusano
---

### Patreon security breach

Important notice: for folks who are [supporting my work on Patreon](https://www.patreon.com/pchiusano?ty=h), Patreon had a security breach. [Read the details here](https://www.patreon.com/posts/3457485), though I find the disclosure a bit confusing / vague. They apparently do not store "full" CC numbers (okay, good, but how many digits then?), but bcrypt+salted passwords were taken and they recommend changing your password. Assuming they just store the last four digits of your card and have bcrypt+salted the passwords, that is not catastrophically bad, though I'd still change your password and definitely change it if you use that password anywhere else important (which hopefully you don't).

The more worrisome thing for creators (not patrons) on the site is they mention something about SSNs being taken (creators have to give their SSN, I guess so Patreon can generate a 1099), but they were "safely" encrypted with 2048-bit RSA. Do they mean individual SSNs were encrypted? If so that would be trivial to crack since there aren't many SSNs. I hope they release more information about the breach and I've pestered them about it (on Twitter of all places, since they don't provide any offical forum for making comments or even _asking questions about the breach that affects their users_).

### Update on recent progress

I did some refactoring to allow the Unison editor to run "headless", using a local, in-memory node rather than a remote Haskell server. This opens the door to a few things:

* Embeding the editor in existing static webpages. I plan on publishing a version of the editor semi-regularly here on this site. (See below)
* Implementing other code stores, like using browser local storage.
* Supporting 'mixed' client / server evaluation. When evaluating a computation for which the local node has all the needed dependencies (simple arithmetic and pure functions, etc), we can avoid a round-trip to the server.
* Using the exact same logic for testing as is used when contacting a real remote Unison node. Since [`Node` is just an interface](https://github.com/unisonweb/platform/blob/master/shared/src/Unison/Node.hs#L52), the editor can be agnostic to the implementation and it's easy to swap in a `Node` implementation that makes actual requests to a Haskell node server.

Here's a demo:

<iframe src="https://player.vimeo.com/video/141140754" width="500" height="415" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

This is the same rendering code we've seen before, but now we're obtaining type information from the local node (which contains the typechecker) and displaying that dynamically, based on what subexpression the user has selected. Not bad. I spotted a few typechecker and layout bugs which I'm carefully tiptoing around in that video, but when I get those fixed I'll actually post the "editor" (more of a 'viewer' at this point, really) here on this site so people can play with it.

Most of the work was just in abstracting over the hashing function used (probably a good idea anyway), since the SHA3-512 function from the cryptohash package is not something I wanted to compile to JS, and I don't think its implementation is pure Haskell anyway. For now I'm just using [Murmur hash](https://hackage.haskell.org/package/murmur-hash) for the in-memory node. This isn't suitable as a cryptographic hash, but it's fine for now and trivial to swap out. It might be nice to find a pure Haskell implementation of SHA3-512 (or any other good hash) with minimal dependencies, or a hash that has both a Haskell and a JS implementation.

Now that this is hooked up, I can start working on exposing the various editing actions! More updates to come.
