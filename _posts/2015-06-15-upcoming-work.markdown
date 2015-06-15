---
layout: post
categories: [updates]
title: Upcoming work
post_author: Paul Chiusano
---

Quick update: The [design work](/design) I've been absorbed with the past few weeks feels like it's at a pretty good stopping point for now. Working through the [well-typed editing story](/2015-06-12/editing.html#post-start) was really satisfying since that has felt like a big unknown for the project.

With some of this design work "done" (okay, maybe not done, more like lots of fun remaining details to work out as well as the engineering work of implementation), I'm going to work on doing writeups for various projects that are relatively well-defined and could in principle be worked on concurrently by different folks. I'll write these up as [regular GitHub issues](https://github.com/unisonweb/platform/issues) and ping people who I think could be a good fit for them and who I think might be interested. Of course, if you see any items that you'd like to help out with, don't be shy about speaking up!

Coming up, I'm going to get back to working seriously on the Unison editor rewrite. All this design work is not very useful if there's no way to edit the code!

Lastly, a shoutout to [John Ericson](https://github.com/Ericson2314), who has been working on getting the Nix build cleaned up (which in turn led to some yak-shaving on [try-reflex](https://github.com/ryantrinkle/try-reflex) and [nixpkgs](https://github.com/NixOS/nixpkgs)). The current setup in master is [somewhat kludgy](https://github.com/unisonweb/platform/pull/14), and John had a lot more knowledge of Nix than me and stepped up to help out. This is much appreciated, so thanks John!
