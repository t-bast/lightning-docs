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
  * [Reverse upfront payment](#reverse-upfront-payment)
  * [Web of trust HTLC hold fees](#web-of-trust-htlc-hold-fees)

## Description of the attack

The attacker leverages the HTLC-timeout mechanism to lock up liquidity in the network.
This attack doesn't directly cost money to routing nodes, but it wastes capital allocation and may
prevent legitimate payments from going through.

The attacker controls two nodes: `A1` and `A2` (we'll refer to this attack as `controlled spam`).
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

Also note that the attacker doesn't necessarily need to hold the HTLCs for a very long time; he can
release them and repeat the same process instantly, or keep a constant stream of HTLCs to flood the
network; we'll call this attack `short-lived controlled spam`.

There is another variant of this attack that is worth considering. Instead of sending to a node he
controls (`A2`), the attacker sends HTLCs to random nodes he does *not* control. These final nodes
will instantly fail the HTLC (because it doesn't match any invoice in their DB) but the HTLCs will
spend some time locked in channels commitments due to forwarding delays. The attacker can flood the
network with a constant stream of such HTLCs to disrupt legitimate payments. We'll call this attack
`uncontrolled spam`.

## Mitigation strategies available today

It is not possible today to fully prevent this type of attacks, but we can make the attacker's job
harder by properly configuring channels:

* the attacker needs to lock at least `htlc_minimum_msat * max_accepted_htlcs` of his own funds to
  completely fill a channel, so you should use a reasonable value for `htlc_minimum_msat` (1 sat is
  **not** a reasonable value for channels with a big capacity; it may be ok for smaller channels
  though)
* open redundant unannounced channels to your most profitable peers
* implement relaying policies to avoid filling up channels: always keep X% of your HTLC slots
  available, reserved for high-value HTLCs

Long-lived controlled spams might also be mitigated by a relay policy rejecting too far in the
future CLTV locktime or requiring a lower `cltv_expiry_delta`. This later mitigation may downgrade
relay node security.

## Threat model

We want to defend against attackers that have the following capabilities:

* they are able to quickly open channels to any node in the network
* they have up-to-date knowledge of the network's *public* topology
* they are running modified (malicious) versions of LN node implementations
* they are able to quickly create many seemingly unrelated nodes
* they may already have long-lived channels (good reputation)
* they might probe in real-time channel balances to adjust their spams
* they might send long-held HTLCs, those ones unobservable from the set of honest long-held HTLCs

There are important properties of Lightning that we must absolutely preserve:

* payer and payee's anonymity
* trustless payments
* minimal (reasonable) barrier to entry as routing node
* minimal overhead/cost for legitimate payments
* minimal overhead to declare public paths to the network

And we must avoid creating opportunities for attackers to:

* penalize an honest node's relationship with its own honest peers
* make routing nodes lose non-negligible funds
* steal money (even tiny amounts) from honest nodes
* more easily discover their position in a payment path
* introduce third-party channel closure vectors (e.g Alice closing a channel between Bob and Caroll)

## Proposals

Many ideas have been proposed over the years, exploring different trade-offs.
We summarize them here with their pros and cons to help future research progress.

### Naive upfront payment

The most obvious proposal is to require nodes to unconditionally pay a *fixed* tiny amount to the
next node when they want to relay an HTLC. Let's explore why this proposal does **not** work:

* the attacker will pay that fee at the first hop, but will receive it back at the last hop: it
  this doesn't cost him anything
* this fee applies to every payment attempt: when a legitimate user is unlucky and tries multiple
  routes without success (potentially because of valid reasons such as liquidity issues downstream)
  he will have to pay that fee multiple times

### Reverse upfront payment

This [proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-February/002547.html) builds on the previous one, but reverses the flow. Nodes pay a fee for *receiving*
HTLCs instead of *sending* them.

```text
A -----> B -----> C -----> D

B pays A to receive the HTLC.
Then C pays B to receive the forwarded HTLC.
Then D pays C to receive the forwarded HTLC.
```

There must be a grace period during which no fees are paid; otherwise the `uncontrolled spam` attack
allows the attacker to force all nodes in the route to pay fees while he's not paying anything.

The fee cannot be the same at each hop, otherwise it's free for the attacker when he is at both
ends of the payment route.

This fee must increase as the HTLC travels downstream: this ensures that nodes that hold HTLCs
longer are penalized more than nodes that fail them fast, and if a node has to hold an HTLC for a
long time because it's stuck downstream, they will receive more fees than what they have to pay.

The grace period cannot be the same at each hop either, otherwise the attacker can force Bob to be
the only one to pay fees. Similarly to how we have `cltv_expiry_delta`, nodes must have a
`grace_period_delta` and the `grace_period` must be bigger upstream than downstream.

Drawbacks:

* The attacker can still lock HTLCs for the duration of the `grace_period` and repeat the attack
  continuously

Open questions:

* Does the fee need to be based on the time the HTLC is held?
* What happens when a channel closes and HTLC-timeout has to be redeemed on-chain?
* Can we implement this without exposing the route length to intermediate nodes?

### Web of trust HTLC hold fees

This [proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-October/002826.html) introduces fees depending on the amount of time HTLCs are kept pending.
Nodes pay when *offering* HTLCs based on the following formula:

```text
hold_fee = lock_time * (fee_base + fee_rate * htlc_value)
```

The `fee_base` and `fee_rate` depend on the trust relationship between the two peers:

* Nodes charge a high rate to nodes they don't know/trust
* Over time, nodes observe their peers and may lower the fees if they have behaved correctly
  for a long enough period of time
* This ensures spamming is more costly to attackers, who have to either:
  * spend sats to spam
  * or spend time to build a reputation
* Hold fees can be implemented on the protocol-level, but it is also possible to
  enforce the policy externally. For example: stop forwarding payments when the
  hold fee budget is exhausted and require the peer to top up via keysend.

Drawbacks:

* Barrier to entry (for new routing nodes) potentially leading to centralization
* Correctly rating peers is hard: most of the metrics can be gamed by remote nodes to lower a
  peer's score
* Small griefing is possible: peers have an incentive to hold HTLCs longer to collect more fees:
  this is true of all proposals that are based on pay-per-time held where the sender pays the fees
* "Exit scam" type of attacks: malicious nodes behave correctly long enough, then launch an attack

# Adjacent Issues

Solving channel spamming might help in other corner cases of LN.

## Costless channel probing

A node continuously probing channels across the network may discover the payment traffic of routing
nodes and thus globally track LN payment traffic.

## Watchtower Credit Exhaustion

Considering the upcoming deployment of public watchtowers, a LN node may have to pay a cost
per-channel update to avoid a watchtower ressource DoS. A malicious counterparty continously
updating a channel may force the victim to exhaust its watchtower credit, thus knocking-out
victim revocation protection.

If a malicious HTLC sender/relayer have to pay a fixed fee to the victim, it creates a higher
bound on victim watchtower budget. Additional watchtower coverage beyond what this fixed fee
afford has to be paid from victim pocket.

Eltoo is likely to solve this issue by restraining watchtower per-update resource cost to a
bandwidth one only.

# Sources

* [https://arxiv.org/pdf/1904.10253.pdf Discharged Payment Channels: Quantifying the Lightning Network's Resilience to Topology-Based Attacks]
* [https://eprint.iacr.org/2019/1149.pdf LockDown: Balance Availability Attack Against Lightning Network Channels]
* [https://arxiv.org/pdf/2002.06564.pdf Congestion Attacks in Payment Channel Networks]
* [https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002608.html Proof-of-closure as griefing attack mitigation]
* [https://arxiv.org/pdf/2004.00333.pdf Probing Channel Balances in the Lightning Network]
