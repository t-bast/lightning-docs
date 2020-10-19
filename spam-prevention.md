# Spamming the Lightning Network

One of the Lightning Network's main goals is to provide good privacy for payers and payees, thanks
to the combination of source-routing and onion encryption ([Sphinx](http://www.cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf)).

Unfortunately, this property can be abused by malicious actors to spam the network: intermediate
routing nodes cannot easily figure out if the payments they are relaying are genuine payments or
spamming attempts.

## Table of Contents

* [Description of the attack](#description-of-the-attack)
* [Mitigation strategies available today](#mitigation-strategies-available-today)
* [Threat model](#threat-model)
* [Proposals](#proposals)
  * [Naive upfront payment](#naive-upfront-payment)

## Description of the attack

The attacker leverages the HTLC-timeout mechanism to lock up liquidity in the network.
This attack doesn't directly cost money to routing nodes, but it wastes capital allocation and may
prevent legitimate payments from going through.

The attacker controls two nodes: `A1` and `A2`.
The attackers finds a long route between `A1` and `A2`, sends HTLCs through these routes and goes
silent on the recipient's end (simulates a stuck HTLC):

```text
A1 ---htlc1---> Some node ---> ... ---> Some node ---> A2
A1 ---htlc2---> Some node ---> ... ---> Some node ---> A2
...
A1 ---htlcN---> Some node ---> ... ---> Some node ---> A2
```

The intermediate nodes notice that the HTLCs seem stuck *somewhere downstream*, but:

* their only choice is to wait for the HTLCs to timeout, otherwise they risk losing funds
* they cannot know the final destination of the payment, nor its origin
* they cannot blame their direct peers, they may or may not be the attacker
* they cannot know that these HTLCs are related

When the HTLCs are close to timing out, the attacker fails them from `A2` and repeats the same
process. The only cost to the attacker is that he needs to lock the HTLC amounts, but he gets the
funds back immediately when he fails the HTLCs from `A2`.

Since payment routes can be at most 20 hops, it looks like the attacker can lock 20 times the funds
he's allocating to the attack. But in reality it's worse: there is a limit to the number of pending
HTLCs a channel can have (by default 483 HTLCs). By completely filling a channel with tiny HTLCs
(just above the dust limit) the attacker is able to lock the whole channel down at a very small cost.

Note that the attacker may use the same node on both ends (`A1 = A2`).

## Mitigation strategies available today

It is not possible today to fully prevent this type of attacks, but properly configuring channels
can help partially mitigate them:

* use a reasonable value for `htlc_minimum_msat` (1 sat is **not** a reasonable value for channels
  with a big capacity; it may be ok for small channels though)
* open redundant unannounced channels to your most profitable peers
* implement relaying policies to avoid filling up channels: always keep X% of your HTLC slots
  available, reserved for high-value HTLCs

## Threat model

We want to defend against attackers that have the following capabilities:

* they are able to quickly open channels to any node in the network
* they have up-to-date knowledge of the network's *public* topology
* they are running modified (malicious) versions of LN node implementations
* they are able to quickly create many seemingly unrelated nodes
* they may already have long-lived channels (good reputation)

There are important properties of Lightning that we must absolutely preserve:

* payer and payee's anonymity
* trustless payments
* decentralization
* minimal (reasonable) barrier to entry
* minimal overhead/cost for legitimate payments

And we must avoid creating opportunities for attackers to:

* penalize an honest node's relationship with its own honest peers
* make routing nodes lose non-negligible funds

## Proposals

Many ideas have been proposed over the years, exploring different trade-offs.
We summarize them here with their pros and cons to help future research progress.

### Naive upfront payment

The most obvious proposal is to require nodes to unconditionally pay a tiny amount to the next node
when they want to relay an HTLC. Let's explore why this proposal does **not** work:

* the attacker will pay that fee at the first hop, but will receive it back at the last hop: it
  this doesn't cost him anything
* this fee applies to every payment attempt: when a legitimate user is unlucky and tries multiple
  routes without success (potentially because of valid reasons such as liquidity issues downstream)
  he will have to pay that fee multiple times
