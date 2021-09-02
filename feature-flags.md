# Lightning feature flags: from Zero to Hero

This article describes optional lightning features that nodes can turn on and off.
Nodes actively tell the whole network about the features they support, to help peers looking for specific features connect to them.

## `data_loss_protect`

This flag allows a node to detect upon channel reestablishment that it has lost state (i.e. thinks a state is 'current' while it has been superseded and maybe even revoked). This works because the commitment point (that is added with this flag) can be verified to be authentically derived from the secret, so the 'forgetful' node must have sent it to them before it lost its commitment point counter state.

The node that receives the `channel_reestablish` could also just _pretend_ that it has lost state, and e.g. drop the connection after receiving `channel_reestablish`, tempting the peer to close with a revoked state that it thinks the pretender can't punish. But tempting a cheating node is better for an honest node, than entering into a Lightning peer connection, which would eventually get forced closed because of a commitment mismatch. At that point, a cheater knows with certainty that the 'forgetful' node is not bluffing, so the attack odds are better.

Accepted in November 2017: [meeting log](https://docs.google.com/document/d/1x2Qr2VgqauFfPgs-uBv7B4GegifAzRmfA_LpebvsETM/edit)

This option is required by most implementations, e.g. LND: [brontide.go:2749](https://github.com/lightningnetwork/lnd/blob/cc0321d1881ed23c9608cf898af2e2a7b347304a/peer/brontide.go#L2749).

## `upfront_shutdown_script`

This makes a channel party commit to a `scriptpubkey` (output locking script) upon opening of a channel. In a cooperative close, without this flag, the funds are sent to an address chosen upon closing. This flag can increase security somewhat, since even an attacker with the channel keys may not have the keys of the address controlling the chosen `scriptpubkey`.

### Implementations

- Electrum: since [this PR from January 2021](https://github.com/spesmilo/electrum/pull/6875#issuecomment-757900498), most likely part of v4.1.
- C-Lightning: since [v0.7.3 from November 2019](https://medium.com/blockstream/the-latest-c-lightning-0-7-3-3efc107f092b)
- LND: since [this PR merged in December 2019](https://github.com/lightningnetwork/lnd/pull/3655), part of [v0.9.0 from January 2020 according to release notes](https://github.com/lightningnetwork/lnd/releases/tag/v0.9.0-beta).
- Eclair: since [this PR merged in July 2121](https://github.com/ACINQ/eclair/pull/1846)

## `gossip_queries` and `gossip_queries_ex`

`gossip_queries` enables:
* a `query_short_channel_ids` message such that a node can request gossip for given list of short channel IDs.
* a `query_channel_range` message such that a node can request channel updates from a specified block range.
* a `gossip_timestamp_filter` message that a node must use to signal the range of the channel updates it is interested in, filtered by their timestamp.

`gossip_queries_ex` adds timestamps and checksums to `query_channel_range` replies.

### Implementations of `gossip_queries_ex`

* Eclair: since [this PR merged in October 2019](https://github.com/ACINQ/eclair/pull/1165), part of [v0.3.2 according to branch tags on Github](https://github.com/ACINQ/eclair/commit/4300e7b651b934a5e0605d8a0319308894164936).
* C-Lightning: since [this PR merged in September 2019](https://github.com/ElementsProject/lightning/pull/3071/commits/b553da3910daab41d5ef62281d92bb3499a146b2), part of [v0.7.3 according to branch tags on GitHub](https://github.com/ElementsProject/lightning/commit/27790832a5ee60fb595b322386d02883e77cd71a).
* LND: `gossip_queries` supported since [this PR merged in May 2018](https://github.com/lightningnetwork/lnd/pull/1106), part of v0.5.

## `static_remote_key`

This removes key-rotation from the `to_remote` output. This means that the `to_remote` output of every commitment transaction can be spent by the same key. If you lose channel state, this will allow you to claim your balance when the counterparty closes. Without this option, you'd have to grind and see which key matches. Watchtowers are also easier to implement with this.

### Implementations

- C-Lightning: since [this PR from September 2019](https://github.com/ElementsProject/lightning/pull/3104), part of v0.7.3 according to the milestone there.
- Eclair: since [this PR merged in June 2020](https://github.com/ACINQ/eclair/pull/1141), part of v0.5.0 according to [branch information](https://github.com/ACINQ/eclair/commit/dc364a1996548e405199fe38ad861a693baf4787)
- LND: since [v0.8.0 from October 2019](https://github.com/lightningnetwork/lnd/releases/tag/v0.8.0-beta). LND requires this option for new channels: [brontide.go:2790](https://github.com/lightningnetwork/lnd/blob/cc0321d1881ed23c9608cf898af2e2a7b347304a/peer/brontide.go#L2790)
- Electrum requires this for all peers since [4.0.1 from July 2020](https://github.com/spesmilo/electrum/blob/922a48f2b74bd6c58682f9d5b6163c4fbd45981a/RELEASE-NOTES#L107)

## `payment_secret`

Merged in July 2019 as part of MPP: https://github.com/lightningnetwork/lightning-rfc/pull/643

The feature prevents probing attacks as noted in BOLT-11.

You can use payment_secret without necessarily using MPP, though it is a requirement for MPP.

### Implementations

- C-Lightning: since at least [v0.8.0 from December 2019](https://github.com/ElementsProject/lightning/releases/tag/v0.8.0), since that version supports receiving MPP
- Eclair: since at least [this PR from October 2019](https://github.com/ACINQ/eclair/pull/1153), released with MPP (see below)
- Electrum: sent/accepted since [4.0.1 from July 2020](https://github.com/spesmilo/electrum/blob/922a48f2b74bd6c58682f9d5b6163c4fbd45981a/RELEASE-NOTES#L107)
- LND: supported since v0.10.0 (see below), **required** in incoming payments since [v0.12.0](https://github.com/lightningnetwork/lnd/pull/4752) (unreleased as of January 7th, 2020):  All generated invoices will include the `s` field.

## `basic_mpp` (multi-part payments)

This splits a payment into multiple pieces, that all have the same payment hash (so it is [vulnerable to a straddling attack](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-February/000993.html)). This means that the theoretical maximum payment amount can be higher. A receiver doesn't know how many parts a payment is split into. To avoid an attack where one part the MPP payment will never arrive and the HTLC consumes resources over too long, receivers must fail incoming HTLCs after a reasonable timeout, usually 60 seconds.

### Implementations

All listed implementations that have send support, also have receive support.

- C-Lightning: Send support since [v0.9.0 from July 2020](https://github.com/ElementsProject/lightning/releases/tag/v0.9.0)
- Eclair: Send support since [v0.3.3 from January 2020](https://github.com/ACINQ/eclair/releases/tag/v0.3.3), see also [wiki page](https://github.com/ACINQ/eclair/wiki/Multipart-Payments).
- LND: Send support since [v0.10.0 from April 2020](https://github.com/lightningnetwork/lnd/releases/tag/v0.10.0-beta)
