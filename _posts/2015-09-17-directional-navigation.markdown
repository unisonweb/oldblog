---
layout: post
categories: [updates, editor]
title: Progress on editor navigation controls
post_author: Paul Chiusano
---

I have a few videos to demo the navigation control I'm adding to the editor. First, let's look at this old video:

<iframe src="https://player.vimeo.com/video/136965195" width="500" height="387" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

This shows layout of a Unison term, and you can see mouse positions being resolved to paths into the underlying term. All this is done with just a single mouse listener, using the [annotated layout tree tech I've discussed previously](https://github.com/unisonweb/platform/blob/master/shared/src/Unison/Doc.hs).

Next up, I worked on adding highlighting of the current selected path and allowing keyboard navigation. Here was my first attempt. Not very nice. In this video, I press the right arrow several times, then the left several times. Notice how the selection jumps around and doesn't feel very natural:

<iframe src="https://player.vimeo.com/video/139499010" width="500" height="328" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

On the one hand, it's easy to explain what's happening---pressing the right arrow moves the selection to the leaf whose left edge is closest (in terms of horizontal distance) to the starting selection region. But it's a bit jarring to not penalize shifting the selection along the opposite axis of the requested movement. When we press 'right', we don't expect the selection to move _down_ just because an element below happens to have a left edge that's a bit closer. I played with various functions for ordering possible destination regions, but it felt like playing whack-a-mole. One type of navigation would start to feel nicer while another case then behaved strangely. After fiddling with this for way too long, I decided to take a step back and try a different approach. In retrospect seems obvious. Here's the result:

<iframe src="https://player.vimeo.com/video/139606000" width="500" height="343" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

In this video, I press: down, down, down, right, up, up, right, right, right, right, left, left, left, left. I'd say the results are as expected. Whereas the previous algorithm just received the _set_ of leaf regions in the whole layout, the new algorithm directly traverses the annotated layout tree (ALT) data structure used behind the scenes. This is a much better approach.

Something that is nice about this mode of traversal is that it is based on the _geometric layout of the term_, rather than the tree structure of the underlying term. In the above examples, these happen to coincide, but for binary operators or more generally for [DFOs](/2015-08-05/dfos.html#post-start), there's no such guarantee. Let's have a look.

The expression `x + y` is actually represented as the Unison term `App (App + x) y`. But if we're at `x`, it feels nicer to me if moving right puts us at `+`. While `y` is the next leaf in a traversal of the term tree, the leaf immediately to the right _geometrically_ is `+`. 

Navigation controls:

* Up, Down, Left, Right: Move to the leaf above, below, to the left, or to the right. These move the selection geometrically, based on how the term is laid out, not based on the term's tree structure. Possible keybindings: the arrow keys and/or `hjkl`.
* Leftmost, Rightmost: Move to the leaf furthest to the left or right. Also move geometrically. Possible keybindings: `<shift>+<left>` and `<shift>+<right>`.
* Expand, Contract: Expand moves the selection to the parent. Contract moves the selection to the leftmost child (geometrically). Possible keybindings: `<shift>+<up>` and `<shift>+<down>`.

It should be obvious that you can navigate anywhere in the document using these controls, and it can be quite efficient, too. Of course, you can also use the mouse if you want.

Semantic editing can be much better for accessibility than plain text editing, because there is less information to specify. With a text editor, we have to disambiguate which characters in a buffer the user is referring to without any information about what these characters mean. Selections that don't even correspond to a parseable fragment (like `1,2) xyz; bar }}`) are still allowed and the editor forces us to disambiguate whether we intended one of these nonsense selections. If you're physically able to use a full keyboard and mouse, this extra specification of information doesn't take much additional time so you may not notice it, but if you were unable to use a mouse, it becomes much more apparent.

Here are a couple other use cases:

* Voice-directed programming: I love the idea of standing in a room with a massive projector and dictating editing commands. If I'm using a text editor, the amount of information I have to specify is large and makes this cumbersome (not to mention inaccurate). With a semantic editor, the amount of information being specified is much smaller, to the point where I think this can become quite usable.
* Gesture-directed programming: Perhaps in conjunction with voice-direction, hook up a Kinect or similar device, and translate gestures into editing commands. Again, with less info to specify, this becomes much more usable.
* Really, the editor shouldn't have strong opinions about _how_ editing and navigation actions are triggered. It should allow multiple inputs, with the user free to choose whichever input mode(s) are most convenient for them.

Something else I've been keeping in mind that's related to this is the _programming on a tablet_ use case. With a semantic editor, programming on a tablet becomes a lot more realistic, even for serious programming, because there's so much less to specify.

### Other updates and remarks

I haven't had a lot of time to work on the project lately. I've been pretty busy with wrapping up some paid work and also with buying a house and moving! Now that things are settling down (we just moved last week!), I'll probably have some more time to work on Unison. Coming up, I'm going to work on adding the remaining navigation controls, and then editing capabilities. I figured out a nice way of doing this so that people can actually try out the editor without having to download anything or have a node running. If this works out as I hope it will, future posts will have a live link you can test out, not just videos. Of course, you can also always [build the project locally](https://github.com/unisonweb/platform) if you want to try things out beforehand.

#### Nix / GHCJS workflow woes and some possible solutions

I've gotten pretty frustrated with the Nix/GHCJS workflow. The time between changing some Haskell code and seeing the results in the browser is on the order of _minutes_ with the current setup. Pretty bad. There are a couple things going on here:

* The project is currently set up as three separate subprojects, `shared/`, `editor/` (depends on `shared/`), and `node/` (also depends on `shared/`. The navigation functions most naturally belong in `shared`.
* When launching the `editor/` nix-shell, Nix recompiles `shared/` from scratch if it's changed. I'm guessing it just doesn't have enough information to do incremental compilation. All it sees is that the hash of the source has changed; therefore the project is rebuilt? Or is there some way to configure Nix so that it's smarter about this?
* GHCJS compilation is slow, especially with use of template Haskell. There's quite a bit of TH in `shared/` for things like JSON serialization.
* `cabal build` is slow, compared to a REPL `:reload`

Happily, it looks like there are improvements coming down the pipeline that will make most of this go away. The `improved-base` branch of GHCJS will have faster template haskell handling and [a REPL](https://twitter.com/acid2/status/614076905990582272). At least if everything is in a single project, it seems like refresh times should be as fast as a REPL `:reload`.

The other option, available right now, is to switch development to Linux. The Reflex-DOM FRP library I'm using to write the editor has two backends, one which compiles to JS and runs in the browser via GHCJS, and one which uses Webkit GTK and runs as native Haskell code. I'd have to do a little work to convert some foreign JS code to GHCJS-DOM Haskell, but I think this would be straightforward. I don't have a linux machine, but I could either buy one or develop in a VM.

Either way, I might combine the `shared/` and `editor/` projects, at least temporarily, for ease of development. I separated them initially just to make it easier to reason about what would end up getting compiled to JS, but that doesn't seem as important as fast refresh times when doing heavy development on the editor.

That's all for now!
