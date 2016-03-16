---
layout: post
categories: [updates]
title: Semantic editing makes programming faster and easier than text editing
post_author: Paul Chiusano
---

I decided to do a head-to-head speed test comparing the current Unison editor to raw text editing in Vim. Below, here's me writing the same code in the Unison editor and in Vim, typing at similar speeds, strictly using the keyboard with both:

<video width="100%" height="auto" controls="true"><source src="videos/speed-test.mp4" type="video/mp4"></video>

Overall, they are pretty close. The Unison editor wins out by a bit in this run, but I did other runs where the text editing was a bit faster. Here's my analysis:

* In the Unison editor, we don't need to type full identifier names, and name resolution is type-directed. For instance, when I defined `salaries` by wrapping `employees` in a function call, I didn't even bother typing anything, since `employees` was already the first hit! I just hit `<Enter>`, then 'a' to wrap the current selection in a function call. Likewise, I didn't need to type out `fold-left` in its entirety.
* We don't need to close _or open_ parentheses. For instance, when supplying the first argument to `map`, the editor helpfully inserted parens around the lambda for us.
* Some tweaks to the keyboard interactions and explorer searchbox parser could probably improve the speed of the Unison editor a bit. For instance, the way you currently introduce a list is by typing `[_`, accept and advance, then just giving the list elements separated by commas as you would when text editing. It would be a bit more natural for the searchbox parser to correctly handle `[<expression>` so this can be done with a single action. An optimized keyboard interaction is introducing a new binding to a `let` or `let rec` block would be nice, too.
* Not really a factor in these videos, but Unison navigation is potentially more efficient, since it can more easily target semantically meaningful units of code.

None of these factors are all that significant though. If you look at what programmers spend their time doing when programming (other than _thinking_ of course), it's actually stuff like:

* Adding imports to the code to be able to resolve names
* Doing google / hoogle searches to even find the name of the function you're interested in
* Looking up / reading documentation
* Compiling the code, then reading type errors, then fixing those errors 
* Large-scale refactorings ("add an extra argument to this function and update all callers")
* Small-scale refactorings ("factor this value out into a let binding")
* Filling in 'boring' chunks of code whose implementations are at least partially specified by the types (like what a tool like [djinn](http://lambda-the-ultimate.org/node/1178) or another solver can work out)
* Formatting code

These are the areas where Unison starts winning. In Unison, there are no imports, type-directed search is tightly integrated into the editor, documentation can be displayed inline (imagine a slide-out panel in the explorer, for instance), there's no compilation phase the programmer ever waits for, no compile errors to decipher and errors are caught earlier, refactorings can be nicely integrated, and so on.

Although none of the individual features here are particularly new, Unison can put it all together into a tightly-integrated, cohesive experience. Besides just the direct time saved of each individual feature, there's also the (highly nonlinear) [improvements in programmer flow due to not needing as many context switches](http://pchiusano.github.io/2015-03-26/type-errors.html).

But it's not just about time savings. The semantic editing experience just feels overall _easier_. It feels like snapping a puzzle together. There's less to think about. Fewer arcane details of the tooling. Though it's not currently a primary goal of Unison, I could imagine teaching some version of this environment to a kid in middle school, or a spreadsheets user, and not feeling like I needed to apologize constantly. The current programming experience is needlessly complicated, and it doesn't have to be.
