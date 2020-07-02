# Pinning Attacks

This article summarizes the discussions about pinning attacks (discussed
[here](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002639.html) and
[here](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-June/002758.html)) and ideas
that have been proposed to mitigate this class of attacks.

## Table of Contents

* [Threat Model](#threat-model)
* [Attacks on Lightning](#attacks-on-lightning)
  * [Attacking individual HTLCs](#attacking-individual-htlcs)
  * [Attacking commitment transactions](#attacking-commitment-transactions)
* [Anchor outputs](#anchor-outputs)
* [Mitigations](#mitigations)
  * [Update fee must stay](#update-fee-must-stay)
  * [Spamming the bitcoin network](#spamming-the-bitcoin-network)
  * [Pay for preimage](#pay-for-preimage)
  * [Out of band package relay](#out-of-band-package-relay)
  * [Insert your ideas here](#insert-your-ideas-here)

## Threat Model

In Bitcoin, "There Ain't No Such Thing As A Global Mempool" (TANSTAAGM - credits to Z-man).
Bitcoin will not try to enforce mempool convergence nor try to resolve mempool [split-brains](https://en.wikipedia.org/wiki/Split-brain_(computing)).
These split-brain occurences are slowly and partially resolved by getting some of the mempool
transactions confirmed in blocks: that is and will always be the only consensus-enforced data
structure available.

A clever and well-funded attacker is able to manipulate mempools and create arbitrary split-brains,
that last an arbitrary amount of time.
We must work under the assumption that such attackers are able to accurately split the network's
mempools, ensuring for example that N% of miners' mempool contain a transaction txA while the rest
of the network's mempools all contain a conflicting transaction txB.

Lightning, as well as other off-chain contracts, must ensure that users don't suffer a loss of
funds from such mempool partitions.

## Attacks on Lightning

In this threat model, there are two types of attacks that can be carried out: attacks on individual
HTLC transactions and attacks on commitment transactions. The goal of both of these attacks is to
steal pending HTLCs (HTLC outputs in the broadcast commitment transaction), exploiting a routing
node by timing out HTLCs upstream while collecting them on-chain downstream.

```ascii
  Attacker -----------------------> Victim ----------------------------> Attacker
 (upstream)         ^                                    ^             (downstream)
                    |                                    |
                    |                                    |
     +---------------------------+          +------------------------+
     | HTLCs timed out off-chain |          | HTLCs claimed on-chain |
     +---------------------------+          +------------------------+

```

Note that none of these attacks are able to steal participants' main outputs, only HTLCs.
A general mitigation is thus to limit the number of pending HTLCs in the commitment transaction,
and limit the amount pending across those HTLCs. Existing Lightning implementations already offer
ways to configure these limits.

These attacks cannot be used to steal from wallets either, since they don't relay payments.

### Attacking individual HTLCs

This attack works at the HTLC transaction level. The attacker creates the following mempool
split-brain:

* all miners have a success transaction in their mempool, revealing the preimage to claim the HTLC
* the rest of the network has a timeout transaction in their mempool

The attacker ensures the preimage transaction has a low enough feerate to be kept in the mempool
without confirming before the upstream channel can timeout them off-chain.

The attacker can also ensure that the transaction in the miners' mempool cannot be replaced, by
attaching a long enough chain of child transactions (BIP 125 rule 5).

This can be done:

* With the current commitment format (no anchor outputs), by forcing the victim to broadcast their
  commitment since the htlc preimage branch has no restriction on how it's spent.
* With anchor outputs, either with the same solution as above, or by adding outputs to the
  HTLC-success transaction (thanks to `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`).

There are two ways the victim can defend against this attack:

* learn the preimage before the upstream HTLC times out
* or get his timeout transaction confirmed

### Attacking commitment transactions

This attack works at the commitment transaction level, and thus impacts all HTLCs outputs in that
transaction. The attacker creates the following mempool split-brain:

* all miners have one version of the commitment transaction
* the rest of the network has another version of the commitment transaction

Note that there are variants of this attack where the network can be arbitrarily split into N
clusters, each with a different version of the commitment transaction. At a high level, this
achieves the same result and mitigations should address this whole class of attacks.

The attacker ensures the commitment transaction in the miners' mempools has a low enough feerate to
be kept without confirming before the upstream channel can timeout HTLCs off-chain.

The attacker can also ensure that the transaction in the miners' mempool cannot be replaced:

* With the current commitment format (no anchor outputs), this can be done by using a commitment
  transaction with a feerate at least equal to the victim's latest commitment, since the victim
  cannot bump the fees of his own commitment transaction.
* With anchor outputs, this can be done by attaching a long enough chain of child transactions via
  the attacker's anchor (BIP 125 rule 5).

The only way the victim can defend against this attack is by somehow getting a commitment
transaction confirmed. It doesn't matter which one is confirmed: if it's a revoked commitment, the
victim can claim all the channels funds (payback time, b*tches), if it's the latest commitment the
victim has to claim or timeout HTLCs afterwards.

## Anchor outputs

The [anchor outputs proposal](https://github.com/lightningnetwork/lightning-rfc/pull/688) doesn't
fix this class of attacks, but it doesn't make the situation worse. See the two previous sections
for details.

It solves another class of attacks, so I believe it makes sense to implement today (regardless of
potential future commitment changes once bitcoin offers some kind of package relay).

## Mitigations

First of all, let's clarify that CPFP/carve-out cannot save us in that case. Without package relay,
CPFP will not help our valid transaction propagate all the way to miners' mempools. And even if you
knew what transaction is in the miners' mempools somehow and wanted to use CPFP/carve-out to get
that transaction to confirm (which unblocks the situation) you cannot propagate your carve-out to
miners because your area of the network doesn't have the transaction from which you're carving out
in their mempools, so they won't relay anything.

A long-term mitigation is to deploy some form of package relay on the bitcoin network that would
allow us to send a package "our valid tx + a child that bumps fee" that would always be able to
replace the attacker's pinned transaction (as long as our package pays enough fees). This is not
an easy task and it will take time, so we will now explore some short-term mitigations.

Disclaimer: most of these (clever) ideas are not mine. I'm merely trying to summarize them to help
the discussion move forward.

### Update fee must stay

The current `update_fee` mechanism is clearly imperfect; it's impossible to predict the future.
But it does raise the bar for the commitment transaction pinning attack. If you ensure commitment
transactions you sign always pay a high fee, the attacker can only keep it pinned in miners'
mempools during high mempool congestion. If your `cltv_expiry_delta` is high enough, the attacker's
commitment transaction is likely to get confirmed, which thwarts the attack (especially if it's a
revoked commitment transaction that confirms).

This doesn't mitigate the attack on HTLC preimage transactions though, because those have a feerate
unilaterally set by the attacker.

### Spamming the bitcoin network

Since the root of the issue is that we're missing information that some mempools have, the most
obvious solution is to connect to everyone available (ie every bitcoin node that accepts incoming
connections). It allows a node to discover conflicting transactions and either learn a preimage
or figure out what CPFP transaction it needs to send to speed up confirmation of a pinned
commitment transaction.

Obviously this solution doesn't scale and may potentially not work; you have no guarantee that you
will be able to connect to enough nodes to probe mempools quickly enough (before upstream HTLCs
timeout).

NB: this solution would also require a small change to `bitcoind`. LN nodes will need their
`bitcoind` to send them conflicting transactions that match a given format. When the LN node
receives such transactions, it can then act on them (extract preimage / commitment transaction).

### Pay for preimage

When we detect that our HTLC-timeout doesn't confirm in time, we can incentivize random nodes to
discover and share preimages by giving them a reward.

This can come into free forms:

* Broadcast pay-for-preimage transactions on-chain hoping they'll be claimed and you learn the
  preimage in a block (`OP_SHA256 <hash_whose_preimage_bob_wants> OP_EQUAL`)
* Send probe HTLCs off-chain to random nodes; if these nodes know the preimage, they will claim
  the HTLC even though they didn't create a matching invoice
* Rely on altruistic off-chain gossip of discovered preimages (we can rate-limit that gossip based
  on the channel capacity our peer opened to us to avoid making this a DoS vector)

These solutions all assume that nodes watch for mempool conflicts and analyze the conflicting
transactions to discover preimages (see previous section). These solutions only mitigate the attack
on individual HTLCs, not the one on commitment transactions.

### Out of band package relay

Another mitigation that could work for both attacks is to implement a limited package relay outside
of the bitcoin p2p relay. If (some) miners pick up these packages and manually (to be defined)
replace the conflicting transactions with the high-fee package received, that allows the honest
participant to get his transactions confirmed in time. As long as the fees are higher, miners have
an economic incentive to select these transactions.

There are many out-of-band medium that can be used to propagate these packages, for example:

* Using LN gossip (where we can rate-limit based on our peer's channel capacity). This assumes that
  (some) miners will run a lightning node and receive that gossip.
* Using centralized APIs that miners would connect to over Tor. Lightning nodes with many funds at
  risk can expose web servers that serve these high-fee packages when they need to broadcast them.
  Miners could regularly poll these servers and pick transactions they like.
* Using publicly advertised bitcoin nodes that miners could connect to. The drawback is that it
  attracts DoS attacks (same for the centralized APIs).

This would require a few changes on Bitcoin Core. There is currently no RPC that lets the user
evict transactions from the mempool (and replace them with a different package); this could be
useful addition. Bitcoin Core could also provide a whitelist-only flag that would let miners
blindly accept conflicting transactions from chosen nodes (this assumes the miners trust these
nodes).

### Insert your ideas here

<insert your own ideas here>

## Resources

Thanks to David Harding, Matt Corallo, Antoine Riard and ZmnSCPxj for their patience, their
explanations and lengthy discussions and brainstorms!

The following links may be useful to dig further:

* [Opt-in RBF](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki)
* [OG RBF pinning thread](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002639.html)
* [Summary RBF pinning thread](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-June/002758.html)
