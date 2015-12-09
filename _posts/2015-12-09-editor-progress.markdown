---
layout: post
categories: [editor, updates]
title: The editor is really coming together!
post_author: Paul Chiusano
---

Even though the editor is just a small piece of the overall platform, having a v1 of it is super important. Without a way to create and interact with Unison programs, there's no way to see or get at all the amazing functionality the platform can offer.

Well, I've been working on the editor for a while. At times it's felt like it would take forever... but I've continued chugging away, learning a lot in the process. Over the past few weeks, something really exciting has happened---I feel like I can see the destination visible on the horizon. I'm not there yet, but it's not some indeterminate distance away, it's _right over there_. I've seen real glimpses of what programming can be like in this highly-structured, type-directed, compile-error free programming environment... and it feels _absolutely awesome_. A little further along and I just know it---I'll wonder how the hell I ever survived so long using text editors and IDEs.

Here's a recent video showing progress which [I tweeted about](https://twitter.com/unisonweb/status/672275565630459904):

<iframe src="https://player.vimeo.com/video/147679458" width="500" height="368" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

This is an implementation of a lot of the [design discussed here](/2015-10-14/fluent-editing.html). It shows the editing of a single term. Some notes:

* When the explorer is open, typing two spaces accepts the current selection and advances to the next nearby blank, and typing one ',' accepts the current selection, inserts a new blank, and moves to it. These actions are meant to make the editing experience more fluid, with less back and forth between navigation mode and explorer mode.
* The term layout is computed automatically, with the expression wrapping sensibly onto multiple lines when it gets too wide. List literals are parsed in the explorer, but notice the user doesn't have to remember to balance the square braces (it's not possible to create an ill-formed or ill-typed expression).
* Terms matching the query but which are ill-typed are still shown in the explorer, but are unselectable. When nothing is selectable in the explorer, it turns red; when only one thing is selectable, it turns blue, and when multiple items are selectable, it is gray. The idea here is that during rapid entry of expressions, the user can often just observe the color of the explorer box without having to mentally parse its contents in detail (for instance, the user might type just the first letter or two of an identifier, watch the box turn blue, and hit space twice to accept and advance).
* There's often _less typing_ overall than in a text editor. You only have to disambiguate _navigation_ actions to semantically meaningful movements, and you only have to disambiguate _editing_ actions to semantically meaningful replacements. Overall you have less to specify to the computer, less typing, which leaves you with more time to think about what matters---assembling your program!

Lots of things aren't implemented yet:

* Full support for the current language---`let` and `let rec` blocks, lambdas, etc
* Various refactoring actions, like abstracting parameters out into a function argument or let binding
* Symbol editing
* Type editing, for adding type annotations

What is nice though is that the way each of these actions is encoded fits in the same overall scheme, which makes them very easy to implement. You go to the spot you want to edit, bring up the explorer, and valid replacements _and actions_ (like the above) are shown. It's only exposing actions via more ad hoc interactions that takes a bit more work. Of course, the explorer can grow smarter about presenting possible suggestions, but that's "easy".

Something I've noticed is that at this stage, every small addition to the available action set makes a huge difference in perceived fluidity and expressiveness. Just adding the `,` keystroke to the explorer for building up a list literal (or more generally doing any kind of insertion) made the experience feel way more direct and fluid. I'm excited to see how it feels when some of the other actions are hooked up.

When I get a few more features implemented and bugs ironed out, I'll post the editor online here on this site, for anyone to try right in the browser!
