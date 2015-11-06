---
layout: post
categories: [updates, editor]
title: Building the fancy autocomplete widget for use in the editor
post_author: Paul Chiusano
---

I'm deep in the middle of implementing [the design discussed in the earlier post](/2015-10-14/fluent-editing.html). The 'navigation' mode stuff we've seen before, it' mostly done and [shown here](/2015-09-17/directional-navigation.html). The other mode is the 'explorer' mode. You navigate around, select a subexpression for editing, and up pops a crazy advanced autocomplete box with a ton of functionality, which is what I've been building (have just finished v1, actually), learning a lot about Reflex in the process. It's a complex component, with a lot of requirements:

* Static content which is loaded asynchronously, when the explorer is first opened, showing the current type, the types of any local variables in scope, etc. We don't have all that type information up front, when the explorer first opens, and thus need to query for it. Documentation associated with the current selection is also static content that might need to be fetched.
* Dynamic results which are fetched asynchronously, and filtered based on what the user has typed in the searchbox. Some queries may be fully answered using local state previously fetched, and other queries may require a trip to the node.
* Keyboard-based navigation using the arrow keys. Selection should remain sticky when async results arrive. If I've selected "Foo" in the explorer, and new results come in, "Foo" should remain selected.
* Keyboard-based accept/cancel.
* Mouse-based navigation, accept, and cancel.
* Searchbox-triggered accept and/or accept-and-advance actions. For instance, ending the searchbox with two spaces might accept the current selection, move to the next location, and open the explorer at that location, without any laborious mode-switching.
* Support for accumulating state. Data fetched by the explorer should be returned and available (if needed) the next time we open another explorer.
* Search results may be shown which aren't _valid_ or selectable. We don't want to silently avoid showing the user something ill-typed. Instead, if there aren't well-typed results, the ill-typed results which otherwise match the query are shown, unselectable, with some sort of visual indicator to make it clear they aren't valid.
* Support for indicating results are partial, support for dynamically loading additional results, etc.

A tall order!

Since this explorer widget is going to be used for editing _terms_, _types_, _type declarations_, and other stuff like _symbols_ (for instance when renaming a function parameter), I wanted something somewhat generic. Building components like this is easy with Reflex, you have total control over the results, and the full power of Haskell to abstract over whatever you wish. After some futzing around, I've landed on the following signature, which I'll explain:

```Haskell
explorer :: forall t m k s a . (Reflex t, MonadWidget t m, Eq k, Semigroup s)
         => Event t Int
         -> (s -> String -> Action m s k a)
         -> Event t (m ())
         -> Dynamic t s
         -> m (Dynamic t s, Event t (Maybe a))
explorer keydown parse topContent s = ...

data Action m s k a
  = Request (m s) [(k, Bool, m a)] -- `Bool` indicates whether the choice is selectable
  | Results [(k, Bool, m a)]
  | Cancel
  | Accept a
```

Let's walk through this:

* The return result is a `Dynamic t s` (for any state accumulated by the various async request made), and an `Event t (Maybe a)`, which will be `Nothing` if the widget closes with a cancellation action, and `Just a` if it closes due to an accept action. It's then up to the parent node to do something with this accepted value. Note that the `a` type can be anything, so it might have some instruction to the caller like "replace the selection with this term, and reopen the explorer for the next expression to my right in the layout".
* The first parameter, `keydown`, receives the keyboard event to use for up/down keyboard navigation within the explorer. You don't want to just bind to the global keyboard event for the entire window. With the DOM, you can attach keyboard listeners on a per-element basis, but they are only consulted when the element has "focus". The DOM's concept of focus is pretty ugly, rather fragile, and it's requested using a _side-effecting function_, gak! In a nice FRP system like what Reflex provides, it's easy to focumulate your own concept of focus and manage it explicitly using regular code, not DOM magic.
* `parse` handles parsing the searchbox. It receives the current state, of type `s`. In the simplest case, `s` might be a list of results, keyed by string, and `parse` does some string matching. Note that `parse` can trigger acceptance or cancellation of the explorer, or another asynchronous request for more results. The `Results` constructor indicates that the results can be fully satisfied without a trip to the node. The `Request` constructor triggers an asynchronous fetch of additional state. The results are tagged with a key and a boolean. The key is used to provide stickiness of selection as new results arrive, and the boolean indicates whether the result is selectable or not.
* `topContent` is static content filled in asynchronously. It's shown right below the explorer.

[The implementation of this function](https://github.com/unisonweb/platform/blob/master/editor/src/Unison/Explorer.hs#L38) is 60 lines of code. And it actually works... the results are reallly ugly at the moment, but the implementation supplies some CSS classes that can be used to make it look nice. If you are for some reason interested in seeing the ugly test page I put together, you can build the code and then [launch this file](https://github.com/unisonweb/platform/blob/master/editor/explorer.html).

Now that the tough part seems worked out, I'm going to use this to implement the term explorer. Then we can hook everything together and try writing some actual Unison expressions for real! Since the language has `let` and `let rec` bindings, we can actually write some nontrivial programs. I'm excited for this moment, and I plan on posting the editor right here online for people to try out. I look forward to getting feedback from people on what they think about the editing experience!
