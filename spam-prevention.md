# Spamming the Lightning Network

One of the Lightning Network's main goals is to provide good privacy for payers and payees, thanks
to the combination of source-routing and onion encryption ([Sphinx](http://www.cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf)).

Unfortunately, this property can be abused by malicious actors to spam the network: intermediate
routing nodes cannot easily figure out if the payments they are relaying are genuine payments or
spamming attempts.

An evil routing node can use this property to ensure that its competitors' channels are unable to
route payments: this may force payers to route through the evil node's channels instead, earning
him fees and potentially making it economically unsustainable for his competitors. An even more
evil entity could make the whole public lightning network unusable if it has access to a fraction
of the network's capacity.

## Table of Contents

* [Description of the attack](#description-of-the-attack)
* [Mitigation strategies available today](#mitigation-strategies-available-today)
* [Threat model](#threat-model)
* [Proposals](#proposals)
  * [Provable Blaming](#provable-blaming)
  * [Local Reputation Tracking](#local-reputation-tracking)
  * [Naive upfront payment](#naive-upfront-payment)
  * [Reverse upfront payment](#reverse-upfront-payment)
  * [Bidirectional upfront payment](#bidirectional-upfront-payment)
  * [Hold-time-dependent bidirectional upfront payment](#hold-time-dependent-bidirectional-upfront-payment)
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
* they might send long-held HTLCs that are indistinguishable from honest long-held HTLCs

There are important properties of Lightning that we must absolutely preserve:

* payer and payee's anonymity
* trustless payments
* minimal (reasonable) barrier to entry as routing node
* minimal overhead/cost for legitimate payments
* minimal overhead to declare public paths to the network
* incentive to successfully relay payments

And we must avoid creating opportunities for attackers to:

* penalize an honest node's relationship with its own honest peers
* make routing nodes lose non-negligible funds
* steal money (even tiny amounts) from honest senders
* more easily discover their position in a payment path
* introduce third-party channel closure vectors (e.g Alice closing a channel between Bob and Caroll)

## Proposals

Many ideas have been proposed over the years, exploring different trade-offs.
We summarize them here with their pros and cons to help future research progress.

### Provable Blaming

The oldest [proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2015-August/000135.html) discusses
to provide proof of channel closures in case of misbehaving peers not failing/succeeding HTLC
quickly. E.g with Alice sending a HTLC to Caroll through Bob, if Caroll doesn't respond within a
short amount of time, Bob should have close his channel with her and present the closing transaction
as a proof to Alice to clear himself from the routing failure.

This scheme introduces a diverse set of concernes : requirement to understand channel types across
links, privacy breakage, channel frailty, ...

### Local Reputation Tracking

This [proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-May/001232.html) discusses
a reputation system for nodes. A node will keep a real-time accounting of its routing fees earned
thanks to the relayed HTLCs from or to its neighboring peers. After every routing failure, faultive
peer reputation is downgraded until reaching some threshold triggering a channel closure.

This scheme doesn't prevent reputation contamination. From a node viewpoint, failure of your direct
peer or from upstream peer can't be dissociated.

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
`hold_grace_period_delta` and the `hold_grace_period` must be bigger upstream than downstream.

Drawbacks:

* The attacker can still lock HTLCs for the duration of the `hold_grace_period` and repeat the
  attack continuously

Open questions:

* Does the fee need to be based on the time the HTLC is held?
* What happens when a channel closes and HTLC-timeout has to be redeemed on-chain?
* Can we implement this without exposing the route length to intermediate nodes?

### Bidirectional upfront payment

This [proposal](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-October/002862.html)
builds on the two previous proposals and combines them. Nodes pay both a forward and a backwards
upfront fee, but the backwards one is refunded if HTLCs are settled quickly.

```text
A -----> B -----> C -----> D
```

We add a `hold_grace_period_delta` field to `channel_update` (in seconds).
We add three new fields in the tlv extension of `update_add_htlc`:

* `spam_fees` (msat)
* `hold_grace_period` (seconds)
* `hold_fees` (msat)

We add two new fields in the onion per-hop payload:

* `outgoing_hold_grace_period`
* `outgoing_spam_fees`

When nodes receive an `update_add_htlc`, they verify that:

* `hold_fees` is not unreasonable large
* `hold_grace_period` is not unreasonably small or large
* `hold_grace_period` - `outgoing_hold_grace_period` >= `hold_grace_period_delta`
* `spam_fees` - `outgoing_spam_fees` > `0` and not unreasonably small

Otherwise they immediately fail the HTLC instead of relaying it.

For the example we assume all nodes use `hold_grace_period_delta = 10`.

We add a forward upfront payment (`spam_fees`) that is paid unconditionally when offering an HTLC.
We add a backwards upfront payment of `hold_fees` that is paid when receiving an HTLC, but refunded
if the HTLC is settled before the `hold_grace_period` ends (see footnotes about this).

Forward upfront payments strictly decrement at each hop, while backwards upfront payments increment
at each hop (non-strictly).

```text
+---+                          +---+                          +---+                          +---+
| A |                          | B |                          | C |                          | D |
+---+                          +---+                          +---+                          +---+
  |                              |                              |                              |
  | Non-refundable fee: 15 msat  |                              |                              |
  |----------------------------->|                              |                              |
  | Refundable fee: 50 msat      |                              |                              |
  | Refund deadline: 100 seconds |                              |                              |
  |<-----------------------------|                              |                              |
  |                              | Non-refundable fee: 14 msat  |                              |
  |                              |----------------------------->|                              |
  |                              | Refundable fee: 60 msat      |                              |
  |                              | Refund deadline: 90 seconds  |                              |
  |                              |<-----------------------------|                              |
  |                              |                              | Non-refundable fee: 13 msat  |
  |                              |                              |----------------------------->|
  |                              |                              | Refundable fee: 70 msat      |
  |                              |                              | Refund deadline: 80 seconds  |
  |                              |                              |<-----------------------------|
  |                              |                              |                              |
```

* A sends an HTLC to B:
  * `hold_grace_period = 100 sec`
  * `hold_fees = 50 msat`
  * `outgoing_hold_grace_period = 90 sec`
  * `spam_fees = 15 msat`
  * `outgoing_spam_fees = 14 msat`
  * forward upfront payment: 15 msat are deduced from A's main output and added to B's main output
  * backwards upfront payment: 50 msat are deduced from B's main output and added to A's main output
* B forwards the HTLC to C:
  * `hold_grace_period = 90 sec`
  * `hold_fees = 60 msat`
  * `outgoing_hold_grace_period = 80 sec`
  * `spam_fees = 14 msat`
  * `outgoing_spam_fees = 13 msat`
  * forward upfront payment: 14 msat are deduced from B's main output and added to C's main output
  * backwards upfront payment: 60 msat are deduced from C's main output and added to B's main output
* C forwards the HTLC to D:
  * `hold_grace_period = 80 sec`
  * `hold_fees = 70 msat`
  * `spam_fees = 13 msat`
  * forward upfront payment: 13 msat are deduced from C's main output and added to D's main output
  * backwards upfront payment: 70 msat are deduced from D's main output and added to C's main output

* Scenario 1: D settles the HTLC quickly:
  * all backwards upfront payments are refunded (returned to the respective main outputs)
  * only the forward upfront payments have been paid (to protect against `uncontrolled spam` and
    `short-lived controlled spam`)

* Scenario 2: D settles the HTLC after the grace period:
  * D's backwards upfront payment is not refunded
  * If C and B relay the settlement upstream quickly (before `hold_grace_period_delta`) their backwards
    upfront payments are refunded
  * all the forward upfront payments have been paid (to protect against `uncontrolled spam` and
    `short-lived controlled spam`)

* Scenario 3: C delays the HTLC:
  * D settles before its `grace_period`, so its backwards upfront payment is refunded by C
  * C delays before settling upstream: it can ensure B will not get refunded, but C will not get
    refunded either so B gains the difference in backwards upfront payments (which protects against
    `controlled spam`)
  * all the forward upfront payments have been paid (to protect against `uncontrolled spam` and
    `short-lived controlled spam`)

* Scenario 4: the channel B <-> C closes:
  * D settles before its `grace_period`, so its backwards upfront payment is refunded by C
  * for whatever reason (malicious or not) the B <-> C channel closes
  * this ensures that C's backwards upfront payment is paid to B
  * if C publishes an HTLC-fulfill quickly, B may have his backwards upfront payment refunded by A
  * if B is forced to wait for his HTLC-timeout, his backwards upfront payment will not be refunded
    but it's ok because B got C's backwards upfront payment
  * all the forward upfront payments have been paid (to protect against `uncontrolled spam` and
    `short-lived controlled spam`)

The backwards upfront payment is fixed instead of scaled based on the time an HTLC is left pending;
it's slightly less penalizing for spammers, but is less complex and introduces less potential
griefing against honest nodes. With the scaling approach, an honest node that has its channel
unilaterally closed is too heavily penalized (because it has to pay for the maximum hold duration).

Drawbacks:

* If done naively, this mechanism may allow intermediate nodes to deanonymize sender/recipient.
  Randomizing the base `grace_period`, `hold_fees` and `spam_fees` may remove that probing vector.
* Handling the `grace_period` will be a pain:
  * when do you start counting: when you send/receive `commit_sig` or `revoke_and_ack`?
  * what happens if there is a disconnection (how do you account for the delay of reconnecting)?
  * what happens if the remote settles after the `grace_period`, but refunds himself when sending
    his `commit_sig` (making it look like from his point of view he settled before the
    `grace_period`)? In that case the behavior should probably be to give your peers some leeway
    and let them get away with it, but record it. If they're doing it too often, close channels and
    ban them; stealing upfront fees should never be worth losing channels.

### Hold-time-dependent bidirectional upfront payment

One characteristic of bidirectional upfront payments as described above is that
the `hold_fees` are time-independent. If an htlc doesn't resolve within the
`grace_period`, the receiver of the htlc will be forced to pay the full hold
fee. The hold fee should cover the expenses for locking up an htlc for the
maximum duration (could be 2000 blocks), so this can be a significant penalty.
Applications such as atomic onchain/offchain swaps (Lightning Loop and others)
rely on locking funds for some time and could get expensive with a fixed hold
fee.

A different variant of bidirectional upfront payments uses a time-proportional hold
fee rate to address the limitation above. It aims to relate the fees paid more
directly to the actual costs incurred and thereby reduce the number of
parameters.

The complete proposal can be found [here](https://lists.linuxfoundation.org/pipermail/lightning-dev/2021-February/002958.html).

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
per-channel update to avoid a watchtower resource DoS. A malicious counterparty continously
updating a channel may force the victim to exhaust its watchtower credit, thus knocking-out
victim revocation protection.

If a malicious HTLC sender/relayer have to pay a fixed fee to the victim, it creates a higher
bound on victim watchtower budget. Additional watchtower coverage beyond what this fixed fee
afford has to be paid from victim pocket.

Eltoo is likely to solve this issue by restraining watchtower per-update resource cost to a
bandwidth one only.

# Sources

## Mailing List (chronological order)

* [Loop attack](https://lists.linuxfoundation.org/pipermail/lightning-dev/2015-August/000135.html)
* [Analysis: alternative DoS prevention concept](https://lists.linuxfoundation.org/pipermail/lightning-dev/2016-November/000648.html)
* [Mitigations for loop attacks](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-May/001232.html)
* [A proposal for upfront payment](https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-November/002275.html)
* [A proposal for upfront payment (reverse upfront)](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-February/002547.html)
* [Proof-of-closure as griefing attack mitigation](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002608.html)
* [Hold fees: 402 Payment Required for Lightning itself](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-October/002826.html)

## Papers

* [Discharged Payment Channels: Quantifying the Lightning Network's Resilience to Topology-Based Attacks](https://arxiv.org/pdf/1904.10253.pdf)
* [LockDown: Balance Availability Attack Against Lightning Network Channels](https://eprint.iacr.org/2019/1149.pdf)
* [Congestion Attacks in Payment Channel Networks](https://arxiv.org/pdf/2002.06564.pdf)
* [Probing Channel Balances in the Lightning Network](https://arxiv.org/pdf/2004.00333.pdf)
