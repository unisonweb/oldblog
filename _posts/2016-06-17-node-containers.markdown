---
layout: post
categories: [design]
title: Node containers and lightweight, zero-copy node provisioning
post_author: Paul Chiusano
---

We are working on the distributed programming API implementation. After finishing [the guts of it](https://github.com/unisonweb/unison/blob/topic/distributed/node/src/Unison/Runtime/Remote.hs), some questions lingered about how to tie everything together. I went off and did some pondering, and now several things are more clear. I'm not sure how much sense this post will make to other people but wanted to do some kind of writeup.

Let's start with the basics. What is a `Node`? As in, what is the runtime representation of a `Node` value in Unison. Well, let's look at its API:

```Haskell
at : Node -> a -> Remote a
here : Remote Node

instance Monad Remote
-- some other effects can be lifted into `Remote`
```

_Note:_ In the past I've made a distinction between `Remote!` and `Remote` (no '!') but I'm going to ignore that here for clarity.

Using the `Monad Remote`, a computation can bounce around from `Node` to `Node`. We can think of `Node` as a _location where computation occurs_, and Unison takes care of serializing / syncing up any missing dependencies when moving a computation between nodes.

A very simple representation of a `Node` might be a host name and/or IP address, plus a public key or hash of a public key. This is all the information you need to be able to contact the node via an encrypted channel. (Unison does not deal directly with the question of tying that public key to some particular entity, though that might be handled with a separate layer in pure Unison code). For easy sharing, we might encode this as a URL like `http://myhost.com/a90f8dfa9s9d8f9234` where the fingerprint of the public key appears after the hostname.

When contacting a node, we assume that at the node's host a server process is running, say HTTP for now.
 I'll call this server the _node container_, and it will administer _multiple Unison nodes_. So far, we haven't talked about how to create or destroy `Node` values. Here is a proposal:

```Haskell
spawn : Lifetime -> Sandbox -> AccessControl -> Budget -> Remote Node

data Budget
data Sandbox
type AccessControl = List Node

data Lifetime
  = Ephemeral -- node resources get deleted on garbage collection of node reference
  | Linked -- node resources get deleted when parent node goes away
  | Resetting Seconds -- node resources get reset to zero after period of inactivity
  | Leased Seconds (Remote Bool) -- node invokes the Remote Bool periodically and shuts down if false
  | Root -- sticks around forever, unless explicitly destroyed
```

What are all those parameters passed to `spawn`?

* `Lifetime` controls when the `Node` is destroyed. I gave a straw man of what that might look like. The general idea is that a `Node` is not something you have to remember to destroy explicitly.
* `Sandbox` controls what functions [and capabilities](/2016-05-18/iot.html#post-start) it has access to, what other nodes it may contact, and how much computing resources it gets (amount of memory, compute time, etc).
* `AccessControl` is an access control list that controls what other nodes may send it a computation via a call to `at`. Recall that the node reference includes the public key (or a hash of it), and as part of node handshaking, we would verify that the sender has access to the private key associated with that public key.
* `Budget` is a bit hazy. The idea is that it's a value that can be used to enforce some policy regarding provisioning of nodes. Perhaps the `Budget` only allows for a max of 10 nodes running, consuming a total of no more than 10GB of memory.

With this model, spawning nodes is meant to be extremely cheap and involve no copying of data, basically just generating a key pair and that's it. You can generate hundreds of thousands of nodes if you wish. We also get rid of any per-connection notion of sandboxing---the `Node` _is_ the sandbox. Any nodes in the access control list that can contact the `Node` gets access to the full set of capabilities determined by the `Sandbox`. This might seem like a limitation (what if I want `Alice` to get a different sandbox than `Bob`??), but it's not---just create separate nodes for each of your client nodes that you wish to sandbox differently.

Notice that any capabilities granted to the `Node` spawned must be granted quite explicitly.

Now here's some interesting common use cases:

* You want to share some personal information with a third party. Let's say it's your `HomeAddress`, which you keep in a `Reference HomeAddress`. Every so often, you update that `Reference`. When a third party wants your `HomeAddress`, you create a new `Node` with a `Sandbox` that _just_ gives them the ability to _read_ that `Reference`---the sandbox is basically just a `Remote HomeAddress`, and nothing else. The third party `Node` runs this computation (which contacts your sandboxed `Node`) and uses the result whenever it needs to mail you something.
* You want to allow your buddy Tom to send you a list of movie recommendations. He could send you an email with a blob of text, but wouldn't it be great if he could send you a typed, computable value, a `List Movie`? (And if you're Tom, maybe you already have such a list sitting around someplace in your Unison node.) Once again, you could create a sandboxed `Node` which exposed just the ability to _write_ a `List Movie` (which might get merged into a common collection that you then perform computation on to aggregate the suggestions of multiple friends). Perhaps Unison can function as a typed, computable messaging service, where we can send computable values around rather than just blobs of text and images!

### Implementation

For the implementation of the node container:

* There's a single HTTP server running on the container. All messages sent to a `Node` managed by this container route through this server. Conceptually even nodes talking on the same container route the messages through this server, though perhaps this can be optimized to use IPC.
* There's a single _runtime store_ associated with the container. The runtime store doesn't include things like metadata, only what is needed to actually execute code. Importantly, all nodes managed by the container [have the same 'universe' and never need to do any syncing of hashes](/2016-05-18/shared-node-stores.html#post-start) when sending computations among each other.
* We have (at most) 1 OS process for each node in the container. A node may be 'active', 'idle', or 'asleep'. If it's 'idle' or 'active', it is backed by a running OS process. If it's asleep, it's just backed by some bits on disk, which I'll explain in a minute. 'idle' nodes may be put to sleep after a long enough period of inactivity.

The container has some ephemeral state. Here's a description of that state and how it gets updated:

* A `Map Node Status` (call this the _status map_). The `Status` indicates whether the node is idle, active, or asleep, and the current `Lifetime` of the node. If the node is idle, the time of last activity is tracked. If the node is idle or active, we have a handle to the actual OS process as well and can communicate with it via standard in and standard out. The `Status` also records the amount of memory devoted to the node.
* Idle nodes are put to sleep after a period of time. The process is simply shut down.
* On receiving a message tagged for some node in its container, it would:
  * Check if that `Node` is asleep; if so, spin it up in the 'idle' state.
  * Send the node (now awake) the message, and mark the node as 'active'
* On receiving a 'node destroyed' message, it just deletes that node from its status map and its persistent state. The container isn't responsible for the details of what it means to destroy a node; that is delegated to the nodes themselves.
* On detecting a node error, it would simply kill the running process (if it is not already dead) and transition the status to 'asleep'. The node will then exist in the 'asleep' state and will be woken up when it next receives a message.
* The server can receive a verified 'node idle' message from a node, which just transitions the node in its status map to 'idle'. Nodes are responsible for indicating that they are idle.

The container also has some persistent state:

* A `Map Node Budget`, tracking the `Budget` associated with each `Node`. Conceptually, idle or active nodes count against the `Budget`, sleeping nodes do not. When spinning up nodes, the container may allocate more or less than requested, depending on available resources left in the budget.
* A `List Node`, the directory of nodes in the container. On startup, each one of these is nodes is loaded into the status map in the 'sleeping' state.

For each node, _N_, we have some persistent state:

* The set of persistent state references allocated by the node. So any `Index` (persistent key-value store) or `Reference` (a single persistent value). This is kept updated as _N_ runs. An observation here is that mutable state of any kind is non-transferable. We can transfer a snapshot of the current state, or a program to fetch the current state, but not the state itself! 
* The set of nodes spawned by the node (call these _child nodes_), including the full key pair, encrypted using N's public key. This is also kept updated as _N_ runs.
* The `Remote PrivateKey` for _N_. This could be as simple as `pure private_key`, which means that the container will have as persistent state the private key for _N_, or it could be a remote computation that contacts the parent or some other node to obtain the private key. Some details to work out here, but the idea is that the node container can avoid persisting private keys in unencrypted form---the keys would just exist temporarily in memory, and attackers (or administrators!) who have managed to log into the container machine wouldn't have access to them.
* The `Sandbox` for the node
* The `Budget` for the node
* The `Lifetime` for the node
* If the `Lifetime` associated with the node is `Ephemeral`, we can skip storing the private key, since `Ephemeral` nodes only exist as long as they are not garbage collected by their parent node

Nodes respond to a few messages:

```Haskell
data Message = Destroy Proof | Eval e
```

`Eval e` evaluates some expression on the node. This may trigger messages being sent to other nodes, but note that these messages are all logically routed through the container server.

A `Node` given a `Destroy` will check the `Proof` for evidence of knowledge of the node's private key (perhaps a hash of that private key, or even the key itself). Assuming it checks out, it shuts down, which terminates the process and deletes any persistent state allocated by the node. As its last action it notifies the container that it has shut down.

Some notes:

* Nodes can also self-destruct, for instance a `Leased` node might fail to obtain a renewal and self-destruct. In the event of a self-destruct, the container is notified.
* A node may trigger destruction of a child node it created. For instance, if a node is given a lifetime of `Ephemeral`, then it sticks around only as long as the parent node keeps a reference to it. After that, the parent will detect that the child node has become garbage and send it a `Destroy` message. Since it will know the private key, it can provide the proof needed.
* Before shutting down, a `Node` will destroy any `Linked` child nodes.

The idea here is that we don't ever really have to worry about explicitly destroying nodes. They go to sleep if inactive, and get destroyed automatically under the conditions we specify.

I'll be implementing this next week. Stay tuned.
