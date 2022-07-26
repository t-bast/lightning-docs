# Lightning privacy: from Zero to Hero

This article contains many privacy pitfalls of lightning, how you can avoid them, and how we plan
on addressing them in the future.

## Table of Contents

* [On-Chain to Off-Chain privacy](#on-chain-to-off-chain-privacy)
  * [Identifying lightning channel funding scripts](#identifying-lightning-channel-funding-scripts)
  * [Public Channels](#public-channels)
  * [Unannounced Channels](#unannounced-channels)
* [Off-Chain to Off-Chain privacy](#off-chain-to-off-chain-privacy)
  * [Payment correlation](#payment-correlation)
  * [Identifying mobile wallets](#identifying-mobile-wallets)
  * [Path-finding tricks](#path-finding-tricks)
  * [Trampoline](#trampoline)
* [Resources](#resources)

## On-Chain to Off-Chain privacy

Whenever you use lightning, there will be an on-chain footprint: you will be opening and closing
channels. It is important to ensure that this on-chain footprint doesn't leak too much information.

### Identifying lightning channel funding scripts

As we saw in the [lightning transactions](./lightning-txs.md) article, the channel funding
transaction uses a p2wsh 2-of-2 multisig.
While the channel is open, this is fine: an attacker looking at the blockchain only sees a p2wsh
output, which could contain anything.
However, when the channel is closed, we have to reveal that it is using a 2-of-2 multisig, which
is a strong indication that this may be a lightning channel.

Fortunately, this is very easy to fix with Taproot. Instead of using a p2wsh output, we will use
[Musig2](./schnorr.md) with a key path spend, which will be indistiguishable from a normal single
signature output.

However, there are two ways that a lightning channel can be closed: either cooperatively, where
both participants create a transaction that sends each one of them their final balance, or
unilaterally when one of the participants isn't collaborating.

In the second case, the transaction that is broadcast is the commitment transaction that we
described in the [lightning transactions](./lightning-txs.md) article. This transaction relies on
specialized scripts that are unique to lightning to allow participants to recover their funds.
These scripts will be visible on-chain and make it obvious that this was a lightning channel.
Unfortunately, there is probably no way to work around this issue while guaranteeing funds safety.
Node operators should avoid unilaterally closing as much as possible: this should only happen when
a peer is malicious or has disappeared.

### Public Channels

Lightning nodes currently advertize their channels in the lightning network by sending a
`channel_announcement` message to their peers, who will relay this announcement to the rest of the
network. This message contains the details of the funding transaction, which lets anyone running a
lightning node know which utxos are actually lightning channels.

The reason channels are advertized publicly inside the network is because we are using source
routing to ensure payment privacy: when Alice wants to send a payment to Bob, Alice must find a
route through existing channels to reach Bob. If Alice has access to the complete topology of the
graph, she can find the route herself, without revealing to anyone that she intends to send a
payment to Bob.

```text
                  +------+          +-----+
                  | Node |          | Bob |
                  +------+          +-----+
                      |                |
                      |                |
+-------+         +------+         +------+
| Alice |---------| Node |---------| Node |
+-------+         +------+         +------+
    |                                  |
    |             +------+             |
    +-------------| Node |-------------+
                  +------+
```

We can't get rid of this mechanism entirely if we want to preserve payment anonymity, but we can
make it better. Instead of disclosing all the details of our channel, we only need to let the
network know that there exists an on-chain transaction of at least a given amount that created a
channel between two nodes. The details of how we'll do that are not completely fleshed out yet,
but it should be possible with some cryptographic wizardry (e.g. ring signatures or zkps).

### Unannounced Channels

Fortunately, nodes that don't want to route payments (e.g. mobile wallets and merchants) have the
option of not announcing their channels to the network.

```text
                  +------+         +------+
                  | Node |---------| Node |
                  +------+         +------+
                      |                |
                      |                |
+-------+         +------+         +------+         +------+         +-----+
| Alice |ooooooooo| Node |---------| Node |---------| Node |ooooooooo| Bob |
+-------+         +------+         +------+         +------+         +-----+
    o                                  |
    o             +------+             |
    oooooooooooooo| Node |-------------+
                  +------+

NB: unannounced channels are represented with "ooooo"
```

However, whenever nodes want to receive a payment, the sender needs to know how to find them in the
graph, so they have to disclose some information about their unannounced channels.
This is currently done using Bolt 11 invoices, where all the details of some of their unannounced
channels are included. This is bad because invoices are sometimes shared publicly (e.g. on Twitter)
which reveals these unannounced channels to everyone.

We have two upcoming features that will fix this:

* [Offers](https://github.com/lightning/bolts/pull/798) provide a static "address" that can be
  shared publicly without revealing channel details.
* [Route Blinding](https://github.com/lightning/bolts/pull/765) lets recipients completely hide
  their `node_id` and channels from payers.

Note that routing nodes can also partially leverage unannounced channels to preserve some utxo
privacy. Whenever two nodes have more than one channel open between them, they can choose to only
announce one of them to the network and keep the others private. When they receive a payment to
relay, they can use the unannounced channels, and nobody can know that they didn't use the public
channel. See the following example where unannounced channels are represented with `oooo`:

```text
          0.4 btc                                  1.2 btc
   oooooooooooooooooooo                    oooooooooooooooooooooo
   o                  o                    o                    o
+------+   1 btc   +------+   1.5 btc   +------+   0.8 btc   +------+
| Node |-----------| Node |-------------| Node |-------------| Node |
+------+           +------+             +------+             +------+
   o      0.6 btc     o
   oooooooooooooooooooo
```

There is a small drawback though: path-finding heuristics use the capacity between nodes to rank
which channels to use. Since routing nodes will be hiding some of their capacity, they may rank a
bit lower in path-finding scores.

## Off-Chain to Off-Chain privacy

Once you have opened channels, lightning should offer greater anonymity for your payments, as they
don't have any on-chain footprint. We explore some subtleties and pitfalls in the following
sections.

### Payment correlation

Lightning payments use an onion routing scheme called [Sphinx](./sphinx.md), which guarantees that
intermediate nodes only know the previous and the next node in the route, but cannot know whether
there are other hops before or after these nodes.

For example, if Alice uses the following payment route to pay Dave:

```text
+-------+           +-----+           +-------+           +------+
| Alice |---------->| Bob |---------->| Carol |---------->| Dave |
+-------+           +-----+           +-------+           +------+
```

When Bob receives the request to forward the payment to Carol, Bob only learns that:

* the payment could come from Alice or another unknown node before Alice
* the payment goes to Carol or another unknown node after Carol

Similarly, Carol only learns the following facts:

* the payment could come from Bob or another unknown node before Bob
* the payment goes to Dave or another unknown node after Dave

This mechanism provides great privacy for payments. However, we are unfortunately leaking some
data: because of how HTLCs work, all nodes in the route are given the same payment identifier,
the `payment_hash`. If two nodes in the route are controlled by the same entity, they can see
that this is the same payment.

```text
                                                 +---------------------------+
                                                 | Well well well...         |
                                                 | I know that payment hash! |
                                                 +---------------------------+
                                                             |
                                                             |
+-------+           +-----+           +-------+           +------+           +------+
| Alice |---------->| Bob |---------->| Carol |---------->| Bob2 |---------->| Dave |
+-------+     |     +-----+     |     +-------+     |     +------+     |     +------+
              |                 |                   |                  |
       +--------------+  +--------------+    +--------------+   +--------------+
       | payment_hash |  | payment_hash |    | payment_hash |   | payment_hash |
       |   0x123456   |  |   0x123456   |    |   0x123456   |   |   0x123456   |
       +--------------+  +--------------+    +--------------+   +--------------+
```

We will see in the following section how this kind of payment correlation can hurt the privacy
of mobile wallet payments.

The good news is that Taproot lets us switch from HTLCs to PTLCs, and one of the benefits of that
change is that it fixes this payment correlation attack entirely: with PTLCs, every node in the
route sees a different, random payment identifier.

```text
                                                 +-------------------------+
                                                 | Never seen that payment |
                                                 | point before...         |
                                                 +-------------------------+
                                                             |
                                                             |
+-------+           +-----+           +-------+           +------+           +------+
| Alice |---------->| Bob |---------->| Carol |---------->| Bob2 |---------->| Dave |
+-------+     |     +-----+     |     +-------+     |     +------+     |     +------+
              |                 |                   |                  |
      +---------------+  +---------------+  +---------------+  +---------------+
      | payment_point |  | payment_point |  | payment_point |  | payment_point |
      |   0x02123456  |  |   0x03ff0123  |  |   0x03abcdef  |  |   0x026bcad3  |
      +---------------+  +---------------+  +---------------+  +---------------+
```

### Identifying mobile wallets

Most users will make their lightning payments from mobile wallets.
Mobile wallets are fundamentally different from server nodes: it is impossible for a mobile wallet
to hide the fact that it is a mobile wallet to its direct peers.
They do not have a stable IP address, they are offline most of the time, and they are not relaying
payments, which is easy to detect.

One of the consequences of that is that when a mobile wallet asks one of its peers to forward a
payment, that peers learns that the mobile wallet is the sender. Similarly, when a node forwards
a payment to a mobile wallet, it learns that this mobile wallet is the recipient.

Combined with the payment correlation issue described in the previous section, it can let attackers
discover who is paying who:

```text
                                                 +-----------------------+
                                                 | Well well well...     |
                                                 | Alice is paying Dave! |
                                                 +-----------------------+
                                                             |
 mobile                                                      |                mobile
+-------+           +-----+           +-------+           +------+           +------+
| Alice |---------->| Bob |---------->| Carol |---------->| Bob2 |---------->| Dave |
+-------+     |     +-----+     |     +-------+     |     +------+     |     +------+
              |                 |                   |                  |
       +--------------+  +--------------+    +--------------+   +--------------+
       | payment_hash |  | payment_hash |    | payment_hash |   | payment_hash |
       |   0x123456   |  |   0x123456   |    |   0x123456   |   |   0x123456   |
       +--------------+  +--------------+    +--------------+   +--------------+
```

Once lightning moves to PTLCs though, intermediate nodes will only be able to learn who the payer
or the recipient is, but not both. As we've seen, mobile wallets cannot expect to hide this
information anyway. However this isn't too bad, because:

* mobile wallets can use Tor or VPNs to hide their real IP address: the only identity that they
  have on lightning is their `node_id`, which they can change regularly (at the cost of opening
  new channels)
* peers only learn that mobile wallets are making payments or receiving them, but cannot know to
  or from whom
* mobile wallets can make payments to themselves to create synthetic traffic and thwart off-chain
  analysis heuristics
* mobile wallets can have multiple peers so that each of their peers only see a subset of their
  payments

Note that this section only applies to mobile wallets that connect to nodes that are run by other
people. If you run a lightning node yourself, and your mobile wallet only connects to that node,
these issues don't apply: the network will not even see that your mobile wallet exists, since
everything will go through your lightning node.

### Path-finding tricks

More advanced attacks against payment privacy are possible by exploiting path-finding subtleties.
Let's explore two of them:

1. multi-part payment path intersection
2. graph filtering

Let's explore multi-part payment path intersection first.
Multi-part payments improve payment privacy against intermediate nodes (because they only see a
fraction of the total payment amount) but gives more data to the recipient node (because it sees
multiple parts arriving through different channels). Let's consider the following multi-part
payment:

```text
+-------+          +------+          +------+
| Alice |----------| Node |----------| Node |------------------+
+-------+          +------+          +------+                  |
                      |                                        | MPP part #1
   +------------------+                                        |
   |                                                           |
+-----+            +------+          +------+  MPP part #2  +------+
| Bob |------------| Node |----------| Node |---------------| Dave |
+-----+            +------+          +------+               +------+
                      |                                        |
                      |                                        |
                      |                                        | MPP part #3
+-------+          +------+          +------+                  |
| Carol |----------| Node |----------| Node |------------------+
+-------+          +------+          +------+
```

Dave can walk back all possible paths to find where they intersect. A simple analysis of the graph
makes it obvious that the most likely sender is thus Bob. With this kind of analysis, sender
anonymity cannot be guaranteed.

Note however that this is a toy theoretical example, exploiting this on the public graph for real
payments may not be feasible, but I believe it's worth mentioning.

The graph filtering attack is even more complex, and requires more resources, but the general idea
is interesting to explore.

Let's consider an end user (Alice) that is connected to a single node (Eve):

```text
+-------+          +-----+
| Alice |----------| Eve |
+-------+          +-----+
```

Alice thinks her payments are private because she computes the routes herself. However, since her
only peer is Eve, Alice gets updates about the public graph only through Eve. Eve can use this fact
to filter out some channels, and provide Alice with a pruned version of the graph that forces Alice
to go through nodes Eve controls to reach most parts of the network.

```text
                                    +------+          +------+          +------+
                      +-------------| Node |----------| Node |----------| Eve2 |-------------+
                      |             +------+          +------+          +------+             |
                      |                                  |                                   |
                      |                +-----------------+                                   |
                      |                |                                                     |
+-------+          +-----+          +------+          +------+          +------+          +------+
| Alice |----------| Eve |----------| Node |xxxxxxxxxx| Node |----------| Node |----------| Dave |
+-------+          +-----+          +------+          +------+          +------+          +------+
                      |                                   |                                  |
                      |                                   |                                  |
                      |                                   |                                  |
                      |             +------+          +------+          +------+             |
                      +-------------| Node |xxxxxxxxxx| Node |----------| Node |-------------+
                                    +------+          +------+          +------+
```

In the sample graph above, Eve filters out the channels marked with `xxxxx`. If Alice wants to pay
Dave, the only routes that she will find will go through Eve's second node, which allows Eve to
deanonymize the payment.

Again, it's important to emphasize that this kind of attacks only works in very specific cases,
and is probably only of theoretical interest.

### Trampoline

[Trampoline routing](https://github.com/lightning/bolts/pull/829) is a mechanism that lets mobile
wallets partially defer calculation of a payment route to intermediate nodes. It is important to
highlight that wallets don't defer the complete route calculation to intermediate nodes, only a
part of it, which is why it can preserve payment privacy.

Let's consider the following graph:

```text
         Alice's local neighborhood                                                                                      Dave's local neighorhood
+--------------------------------------------------+                                                       +--------------------------------------------------+
|                                                  |                                                       |                                                  |
|                    +------+                      |      +------+          +------+          +------+     |                      +------+                    |
|     +--------------| Node |-----------------------------|  N2  |----------|  N3  |----------|  N4  |-------------+ +------------| Node |                    |
|     |              +------+                      |      +------+          +------+          +------+     |       | |            +------+                    |
|     |                 |                          |          |                 |                          |       | |                                        |
|     |                 +------------------+       |          |                 |                          |       | |                                        |
|     |                                    |       |          |                 |                          |       | |                                        |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
| | Alice |----------| Bob |-----------| Terry |----------|  N1  |----------| Node |----------| Node |----------| Ted  |----------| Carol |----------| Dave | |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
|     |                                    |       |                                                       |                                                  |
|     |                 +------------------+       |                                                       |                                                  |
|     |                 |                          |                                                       |                                                  |
|     |              +------+                      |                                                       |                                                  |
|     +--------------| Node |                      |                                                       |                                                  |
|                    +------+                      |                                                       |                                                  |
|                                                  |                                                       |                                                  |
+--------------------------------------------------+                                                       +--------------------------------------------------+
```

Alice and Dave don't need to sync the whole graph, just their local neighborhood, which saves them
a lot of bandwidth and ensures that whenever they run path-finding algorithms, it's on a very small
graph so it's very fast.

Let's walk through a typical trampoline payment.

The first step is for the recipient (Dave) to create an invoice, that will include some trampoline
nodes that are in his neighborhood, and which will be able to route the payment to Dave. Dave finds
these trampoline nodes by doing a simple graph search in his local neighborhood.
To keep the example simple, Dave includes a single trampoline node (Ted) in his invoice.

Alice scans Dave's invoice, which indicates that she must reach Ted. Alice selects a trampoline
node in her own neighborhood (Terry). Alice then builds a trampoline route:

```text
Alice -----> Terry -----> Ted -----> Dave
```

Alice encrypts this trampoline route in a payment onion, using exactly the same construction as
normal payments, but with a smaller size.

Alice then finds a route to Terry in her local neighborhood:

```text
Alice -----> Bob -----> Terry
```

Alice creates a normal payment onion for that route, and includes the trampoline onion in the
payload for Terry. At a high-level, the onion looks like this:

```text
+---------------------------------+
| encrypted payload for Bob       |
+---------------------------------+
| encrypted payload for Terry     |
| +-----------------------------+ |
| | encrypted payload for Terry | |
| +-----------------------------+ |
| | encrypted payload for Ted   | |
| +-----------------------------+ |
| | encrypted payload for Dave  | |
| +-----------------------------+ |
| | padding                     | |
| +-----------------------------+ |
+---------------------------------+
| padding                         |
+---------------------------------+
```

Alice sends the payment onwards:

```text
         Alice's local neighborhood                                                                                      Dave's local neighorhood
+--------------------------------------------------+                                                       +--------------------------------------------------+
|                                                  |                                                       |                                                  |
|                    +------+                      |      +------+          +------+          +------+     |                      +------+                    |
|     +--------------| Node |-----------------------------|  N2  |----------|  N3  |----------|  N4  |-------------+ +------------| Node |                    |
|     |              +------+                      |      +------+          +------+          +------+     |       | |            +------+                    |
|     |                 |                          |          |                 |                          |       | |                                        |
|     |                 +------------------+       |          |                 |                          |       | |                                        |
|     |                                    |       |          |                 |                          |       | |                                        |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
| | Alice |>>>>>>>>>>| Bob |>>>>>>>>>>>| Terry |----------|  N1  |----------| Node |----------| Node |----------| Ted  |----------| Carol |----------| Dave | |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
|     |                                    |       |                                                       |                                                  |
|     |                 +------------------+       |                                                       |                                                  |
|     |                 |                          |                                                       |                                                  |
|     |              +------+                      |                                                       |                                                  |
|     +--------------| Node |                      |                                                       |                                                  |
|                    +------+                      |                                                       |                                                  |
|                                                  |                                                       |                                                  |
+--------------------------------------------------+                                                       +--------------------------------------------------+
```

Bob is an intermediate node in what looks to him like a normal payment. He only learns that:

* the payment could come from Alice or another unknown node before Alice
* the payment goes to Terry or another unknown node after Terry
* Bob does not know that trampoline is being used

When Terry receives the payment, he discovers that he must find a route to Ted and forward the
payment to him. He only learns that:

* the payment could come from Bob or another unknown node before Bob
* the payment goes to Ted or another unknown node after Ted

Terry finds a route to Ted and creates a normal onion for that route, including the trampoline
onion in the payload for Ted (with the topmost layer of this onion unwrapped).
At a high-level, the onion looks like this:

```text
+--------------------------------+
| encrypted payload for N1       |
+--------------------------------+
| encrypted payload for N2       |
+--------------------------------+
| encrypted payload for N3       |
+--------------------------------+
| encrypted payload for N4       |
+--------------------------------+
| encrypted payload for Ted      |
| +----------------------------+ |
| | encrypted payload for Ted  | |
| +----------------------------+ |
| | encrypted payload for Dave | |
| +----------------------------+ |
| | padding                    | |
| +----------------------------+ |
+--------------------------------+
| padding                        |
+--------------------------------+
```

Terry sends the payment onwards:

```text
         Alice's local neighborhood                                                                                      Dave's local neighorhood
+--------------------------------------------------+                                                       +--------------------------------------------------+
|                                                  |                                                       |                                                  |
|                    +------+                      |      +------+          +------+          +------+     |                      +------+                    |
|     +--------------| Node |-----------------------------|  N2  |>>>>>>>>>>|  N3  |>>>>>>>>>>|  N4  |>>>>>>>>>>>>>> +------------| Node |                    |
|     |              +------+                      |      +------+          +------+          +------+     |       v |            +------+                    |
|     |                 |                          |          ^                 |                          |       v |                                        |
|     |                 +------------------+       |          ^                 |                          |       v |                                        |
|     |                                    |       |          ^                 |                          |       v |                                        |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
| | Alice |>>>>>>>>>>| Bob |>>>>>>>>>>>| Terry |>>>>>>>>>>|  N1  |----------| Node |----------| Node |----------| Ted  |----------| Carol |----------| Dave | |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
|     |                                    |       |                                                       |                                                  |
|     |                 +------------------+       |                                                       |                                                  |
|     |                 |                          |                                                       |                                                  |
|     |              +------+                      |                                                       |                                                  |
|     +--------------| Node |                      |                                                       |                                                  |
|                    +------+                      |                                                       |                                                  |
|                                                  |                                                       |                                                  |
+--------------------------------------------------+                                                       +--------------------------------------------------+
```

N1, N2, N3 and N4 are intermediate nodes in what looks like a normal payment. They only learn that:

* the payment could come from the previous node or another unknown node before the previous node
* the payment goes to the next node or another unknown node after the next node
* they do not know that trampoline is being used

When Ted receives the payment, he discovers that he must find a route to Dave and forward the
payment to him. He only learns that:

* the payment could come from N4 or another unknown node before N4
* the payment goes to Dave or another unknown node after Dave
* if Dave is a mobile wallet though, Ted learns that Dave is the final recipient
* but Dave can use route blinding between him and Ted to hide its identity from Ted!

Ted finds a route to Dave and creates a normal onion for that route, including the trampoline
onion in the payload for Dave (with the topmost layer of this onion unwrapped).
At a high-level, the onion looks like this:

```text
+--------------------------------+
| encrypted payload for Carol    |
+--------------------------------+
| encrypted payload for Dave     |
| +----------------------------+ |
| | encrypted payload for Dave | |
| +----------------------------+ |
| | padding                    | |
| +----------------------------+ |
+--------------------------------+
| padding                        |
+--------------------------------+
```

Ted sends the payment onwards:

```text
         Alice's local neighborhood                                                                                      Dave's local neighorhood
+--------------------------------------------------+                                                       +--------------------------------------------------+
|                                                  |                                                       |                                                  |
|                    +------+                      |      +------+          +------+          +------+     |                      +------+                    |
|     +--------------| Node |-----------------------------|  N2  |>>>>>>>>>>|  N3  |>>>>>>>>>>|  N4  |>>>>>>>>>>>>>> +------------| Node |                    |
|     |              +------+                      |      +------+          +------+          +------+     |       v |            +------+                    |
|     |                 |                          |          ^                 |                          |       v |                                        |
|     |                 +------------------+       |          ^                 |                          |       v |                                        |
|     |                                    |       |          ^                 |                          |       v |                                        |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
| | Alice |>>>>>>>>>>| Bob |>>>>>>>>>>>| Terry |>>>>>>>>>>|  N1  |----------| Node |----------| Node |----------| Ted  |>>>>>>>>>>| Carol |>>>>>>>>>>| Dave | |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
|     |                                    |       |                                                       |                                                  |
|     |                 +------------------+       |                                                       |                                                  |
|     |                 |                          |                                                       |                                                  |
|     |              +------+                      |                                                       |                                                  |
|     +--------------| Node |                      |                                                       |                                                  |
|                    +------+                      |                                                       |                                                  |
|                                                  |                                                       |                                                  |
+--------------------------------------------------+                                                       +--------------------------------------------------+
```

Carol is an intermediate node in what looks to her like a normal payment. She only learns that:

* the payment could come from Ted or another unknown node before Ted
* the payment goes to Dave or another unknown node after Dave
* if Dave is a mobile wallet though, Carol learns that Dave is the final recipient (but it would
  be the same with a non-trampoline payment)
* Carol does not know that trampoline is being used

Since Alice and Dave are blind to most of the graph, there is a strong chance that the complete
route used will be non-optimal. For example, the network could actually look like this:

```text
         Alice's local neighborhood                                                                                      Dave's local neighorhood
+--------------------------------------------------+                                                       +--------------------------------------------------+
|                                                  |                                                       |                                                  |
|                    +------+                      |      +------+          +------+          +------+     |                      +------+                    |
|     +--------------| Node |-----------------------------|  N2  |----------|  N3  |----------|  N4  |-------------+ +------------| Node |                    |
|     |              +------+                      |      +------+          +------+          +------+     |       | |            +------+                    |
|     |                 |                          |          |                 |                          |       | |                                        |
|     |                 +------------------+       |          |                 |                          |       | |                                        |
|     |                                    |       |          |                 |                          |       | |                                        |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
| | Alice |----------| Bob |-----------| Terry |----------|  N1  |----------| Node |----------| Node |----------| Ted  |----------| Carol |----------| Dave | |
| +-------+          +-----+           +-------+   |      +------+          +------+          +------+     |    +------+          +-------+          +------+ |
|     |                                    |       |                                                       |                          |                       |
|     |                 +------------------+       |                                                       |                          |                       |
|     |                 |                          |                                                       |                          |                       |
|     |              +------+                      |                                                       |                          |                       |
|     +--------------| Node |----------------------|-------------------------------------------------------|--------------------------+                       |
|                    +------+                      |                                                       |                                                  |
|                                                  |                                                       |                                                  |
+--------------------------------------------------+                                                       +--------------------------------------------------+
```

Where a very efficient route exists, but Alice and Dave don't know about it.
This is actually a good thing for privacy, because it adds unpredictable randomness to the route
which cannot be reverse-engineered by attackers' heuristics.

In this example, we only used two intermediate trampoline nodes, as it is the most efficient way of
using trampoline, but privacy-aware users may also randomly add another trampoline node in the
middle of the route, which ensures that the final route will be very different from the optimal
route.

On top of that, trampoline combines very well with multi-part payments and makes them more reliable
and private. Each trampoline node can aggregate the incoming multi-part payment and then split the
outgoing payment differently:

```text
       150k sat   +------+  250k sat        300k sat   +------+  200k sat
    +-------------| Node |-------------+ +-------------| Node |--------------+
    |             +------+             | |             +------+              |
    |                                  | |                                   |
    |                                  | |                                   |
    |                                  | |                                   |
+-------+         250k sat          +------+           400k sat          +-------+        500k sat         +------+
| Alice |---------------------------| Bob  |-----------------------------| Carol |-------------------------| Dave |
+-------+                           +------+                             +-------+                         +------+
    |                                  |
    |                                  |
    |                                  |
    |             +------+             |
    +-------------| Node |-------------+
       150k sat   +------+  300k sat
```

Since each trampoline node has knowledge of its local balances that remote nodes don't have, they
are able to more efficiently decide how to split outgoing payments.

Moreover, this also fixes the multi-part payment path intersection we've discussed earlier.
Every trampoline node in the route (including the final recipient) is able to do some path
intersection analysis, but the only thing that they would learn is who the previous trampoline
node may be, which doesn't reveal who the actual payer is.

## Resources

* [SLP 319: Lightning Protocol Privacy Exploration](https://stephanlivera.com/episode/319/)
* [Offers specification](https://github.com/lightning/bolts/pull/798)
* [Route Blinding specification](https://github.com/lightning/bolts/pull/765)
* [Trampoline specification](https://github.com/lightning/bolts/pull/829)
