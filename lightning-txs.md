# Lightning transactions: from Zero to Hero

This article contains everything you need to understand Bitcoin/Lightning transactions.
We ignore segwit subtleties and fields that aren't necessary for our global understanding.

## Table of Contents

* [Bitcoin transactions](#bitcoin-transactions)
  * [Transaction format](#transaction-format)
  * [Sighash flags](#sighash-flags)
  * [Absolute locktime](#absolute-locktime)
  * [Relative locktime](#relative-locktime)
* [Lightning transactions](#lightning-transactions)
  * [Overview](#overview)
  * [Funding transaction](#funding-transaction)
  * [Commitment transaction](#commitment-transaction)
  * [HTLC transactions](#htlc-transactions)
  * [Closing transaction](#closing-transaction)
* [Future work](#future-work)
  * [Replace by fee](#replace-by-fee)
  * [Child pays for parent](#child-pays-for-parent)
  * [Bumping Lightning transactions](#bumping-lightning-transactions)
  * [Anchor outputs](#anchor-outputs)
  * [RBF Pinning](#rbf-pinning)
  * [Alternative proposal](#alternative-proposal)
* [Resources](#resources)

## Bitcoin transactions

### Transaction format

A Bitcoin transaction contains the following fields:

```json
{
  "version": 2,
  "locktime": 634346,
  "vin": [
    {
      "txid": "652cd2cc2b12ecc86e77ed067ae11815d6e6347e2ae40b8970a063880798787c",
      "vout": 2,
      "scriptSig": "<some satisfying data>",
      "sequence": 10
    }
  ],
  "vout": [
    {
      "value": 0.1893,
      "scriptPubKey": "OP_HASH160 69ef88bab45ccfaa49f8421fdb4ae39efb23ffb2 OP_EQUAL"
    }
  ]
}
```

* `locktime` can be used to impose an *absolute* timelock.
* `vin` is the list of inputs that this transaction is spending: they match unspent outputs from
  previous transactions that have been included in blocks, and data that satisfies the spending
  conditions (`scriptPubKey`).
  * `txid` references the transaction containing the output we want to spend and `vout` is the index
  of that output in the transaction. It is computed as `SHA256(SHA256(raw_tx))`.
  * `scriptSig` contains data that satisfies the input's `scriptPubKey`.
  * `sequence` can be used to impose a *relative* timelock.
* `vout` is the list of outputs that this transaction creates (unspent outputs until someone spends
  them by including them in a transaction's `vin`). `scriptPubKey` is a Bitcoin script that must be
  satisfied by the spender (usually a signature from a given public key, knowledge of the preimage
  of a hash, etc).

### Sighash flags

One important and very useful subtlety of Bitcoin scripts is the `sighash` flags that can be used
with the `OP_CHECKSIG` opcode (and variants of `OP_CHECKSIG`).

Let's imagine that we have a UTXO with the following `scriptPubKey`: `<alice_pubkey> OP_CHECKSIG`,
indicating that Alice can spend that UTXO. Alice can create a new transaction that includes that
UTXO in `vin`. In the input's `scriptSig`, she needs to provide a valid signature of this *new*
transaction. In addition to the signature bytes, she appends one byte that specifies what parts of
the transaction were signed (that's the `sighash` flags).

These `sighash` flags provide some flexibility for complex, multi-participants transactions. When
some parts of the transaction are *not* signed, that means others participants can slightly update
the transaction while keeping it valid, without requiring the signer to re-sign the changes.

The possible values for that `sighash` byte are:

* SIGHASH_ALL (default): the whole transaction is signed, except input scripts (otherwise there's a
  chicken-and-egg problem). That means the signer doesn't want anyone to be able to modify the
  transaction after it has been signed.
* SIGHASH_NONE: the outputs are not signed. That means the signer agrees to the chosen inputs (where
  bitcoins come from), but lets other participants freely change the outputs (where the bitcoins go).
* SIGHASH_SINGLE: only the output that has the same index as the input containing the signature is
  signed. There is also a small subtlety on the inputs: their `sequence` field isn't signed. This
  means the signer agrees to the chosen inputs and a single output of his choice, but lets other
  participants modify the other outputs freely.
* SIGHASH_ANYONECANPAY: only the input containing the signature is signed, the other inputs are
  ignored and can be modified freely by other participants.

Note that obviously, SIGHASH_ALL, SIGHASH_NONE and SIGHASH_SINGLE are mutually exclusive. However
SIGHASH_ANYONECANPAY can be used in combination with SIGHASH_ALL, SIGHASH_NONE or SIGHASH_SINGLE.

Note that the combination `SIGHASH_NONE | SIGHASH_ANYONECANPAY` is a dangerous footgun: you give
away a UTXO but have no control whatsoever on where it will be sent.

There are proposals to add a flag that doesn't sign any of the inputs, relying only on script
compatibility to ensure transaction validity. There are currently two variations around that idea:

* [SIGHASH_NOINPUT](https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki)
* [SIGHASH_ANYPREVOUT](https://github.com/ajtowns/bips/blob/bip-anyprevout/bip-anyprevout.mediawiki)

### Absolute locktime

Bitcoin makes it possible to restrict the future time at which a transaction becomes valid.
Such timelocks can be expressed in block height or timestamp, but we'll only consider the block
height version here.

A first possibility is to restrict this at the transaction level, by setting the `locktime` field.
The transaction will then be considered invalid until the block height becomes greater than this
`locktime` value. This may be useful if you want to share a signed transaction with someone, but
don't want it to be included immediately in a block. Note that there is one exception to this rule:
if all the inputs' `sequence` fields are set to `0xffffffff`, the `locktime` check is disabled (for
historical reasons).

Another possibility is to restrict this at the script level, by using the following script:
`<block_height> OP_CHECKLOCKTIMEVERIFY`. This is a more powerful mechanism: you can create a UTXO
that can be redeemed by someone else, while preventing the recipient from redeeming it before the
specified `block_height`.

### Relative locktime

Bitcoin also makes it possible to restrict when a UTXO can be spent, relatively to when that UTXO
was created. This is especially useful when you exchange signed transactions off-chain, but don't
know when these transactions will be broadcast on-chain (and thus have no idea what block height
would make sense to put in the `locktime` field).

This is done by using the `sequence` field on a transaction's inputs. When correctly set (according
to the rules defined in BIP68), it ensures that the transaction becomes valid only `n` blocks after
the output it spends has been included in a block. This may be useful if you want to share a signed
transaction with someone, and that transaction spends from an unconfirmed transaction, and you want
to ensure there is a specific delay between these two transactions confirm.

Another possibility is to restrict this at the script level, by using the following script:
`<n> OP_CHECKSEQUENCEVERIFY`. This is a more powerful mechanism: you can create a UTXO that can be
redeemed by someone else, while ensuring the recipient can only redeem it at least `n` blocks after
the first transaction was confirmed.

## Lightning transactions

Lightning transactions leverage the mechanisms explained in the above sections, so make sure you
understood them all before reading on.

[Bolt 3](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md) also
contains all the details about the transactions, but it's quite compact and can be hard to read for
non-implementers. Hopefully this document helps fill in the gaps in that specification.

### Overview

Here is the high-level view of all transactions used for a channel between A and B (note that we
are B in this example, and when we say "us" in the following section we refer to B):

```ascii
+------------+
| funding tx |
+------------+
      |
      |        +-------------+
      +------->| commit tx B |
               +-------------+
                  |  |  |  |  
                  |  |  |  | A's main output
                  |  |  |  +-----------------> to A
                  |  |  |
                  |  |  |                 +---> to B after relative delay
                  |  |  | B's main output |
                  |  |  +-----------------+
                  |  |                    |
                  |  |                    +---> to A with revocation key
                  |  |
                  |  |                                              +---> to B after relative delay
                  |  |                        +-----------------+   |
                  |  |                   +--->| HTLC-timeout tx |---+
                  |  | HTLC offered by B |    +-----------------+   |
                  |  +-------------------+      (after timeout)     +---> to A with revocation key
                  |                      |
                  |                      +---> to A with payment preimage
                  |                      |
                  |                      +---> to A with revocation key
                  |
                  |                                                     +---> to B after relative delay
                  |                            +-----------------+      |
                  |                    +------>| HTLC-success tx |------+
                  | HTLC received by B |       +-----------------+      |
                  +--------------------+     (with payment preimage)    +---> to A with revocation key
                                       |
                                       +---> to A after timeout (absolute delay)
                                       |
                                       +---> to A with revocation key
```

Notation:

* transactions are enclosed in boxes
* incoming arrows represent inputs
* outgoing arrows represent outputs
* branching in an output (with a `+`) represents a branching condition in script (`OP_IF`)

A has a symmetric set of transactions, where A and B have been swapped (and amounts appropriately
swapped as well). Participants are always able to generate the transaction that the *other* side
holds (this is necessary to send them their signature).

NB: in the sample transactions below, we directly write the spending scripts in `scriptPubKey` for
clarity (in real transactions they are P2WSH scripts).

### Funding transaction

A channel always starts with a funding transaction. This one is very simple. It only has to create
an output with the following `scriptPubKey`:

```script
2 <pubkey1> <pubkey2> 2 OP_CHECKMULTISIG
```

We don't care about the inputs that are used, and whether other outputs are created.
The structure of the funding transaction thus looks like:

```json
{
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "...": "..."
    },
    ...
  ],
  "vout": [
    {
      "value": 1.0,
      "scriptPubKey": "2 <pubkey1> <pubkey2> 2 OP_CHECKMULTISIG"
    },
    ...
  ]
}
```

### Commitment transaction

The commitment transaction contains the channel state, which consists of two parts: the balance of
each participant and all the pending payments (HTLCs).

It has a single input, which is the funding transaction and uses `SIGHASH_ALL`. This ensures both
participants agree on the exact details of the commitment transaction when they exchange signatures.

It sets `locktime` and `sequence` to values that allow spending it without delays (these fields are
also used to hide some data in a very clever way we don't care about for this document).

When anything changes in the channel (new payments are added, pending payments are failed or
fulfilled):

* a new commitment transaction is created that replaces the previous one
* participants exchange signatures for that new commitment transaction
* participants reveal the revocation secret for the previous commitment transaction

The commitment transaction for a channel with capacity 1 BTC may look like:

```json
{
  "version": 2,
  "locktime": 543210000,
  "vin": [
    {
      "txid": "...",
      "vout": ...,
      "scriptSig": "0 <signature_for_pubkey1> <signature_for_pubkey2>",
      "sequence": 2500123456
    }
  ],
  "vout": [
    {
      "value": 0.5,
      "scriptPubKey": "
        # to_local output
        OP_IF
          # funds go to anyone who has the revocation key
          <revocationpubkey>
        OP_ELSE
          # or back to us after a relative delay (<to_self_delay>)
          <to_self_delay>
          OP_CHECKSEQUENCEVERIFY
          OP_DROP
          <local_delayedpubkey>
        OP_ENDIF
        OP_CHECKSIG
      "
    },
    {
      "value": 0.3,
      "scriptPubKey": "
        # to_remote output: goes back to the other channel participant immediately
        P2WKH(<remote_pubkey>)
      "
    },
    {
      "value": 0.05,
      "scriptPubKey": "
        # A pending HTLC (payment) sent by us
        OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
        OP_IF
          # funds go to anyone who has the revocation key
          OP_CHECKSIG
        OP_ELSE
          <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
          OP_NOTIF
            # funds go back to us via a second-stage HTLC-timeout transaction (which contains an absolute delay)
            # NB: we also need the remote signature, which prevents us from unilaterally changing the HTLC-timeout transaction
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
          OP_ELSE
            # funds go to the remote node if it has the payment preimage.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
          OP_ENDIF
        OP_ENDIF
      "
    },
    {
      "value": 0.03,
      "scriptPubKey": "
        # Another pending HTLC (payment) sent by us
        OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
        OP_IF
          # funds go to anyone who has the revocation key
          OP_CHECKSIG
        OP_ELSE
          <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
          OP_NOTIF
            # funds go back to us via a second-stage HTLC-timeout transaction (which contains an absolute delay)
            # NB: we also need the remote signature, which prevents us from unilaterally changing the HTLC-timeout transaction
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
          OP_ELSE
            # funds go to the remote node if it has the payment preimage.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
          OP_ENDIF
        OP_ENDIF
      "
    },
    {
      "value": 0.08,
      "scriptPubKey": "
        # A pending HTLC (payment) sent to us
        OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
        OP_IF
          # funds go to anyone who has the revocation key
          OP_CHECKSIG
        OP_ELSE
          <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
          OP_IF
            # funds go to us via a second-stage HTLC-success transaction once we have the payment preimage
            # NB: we also need the remote signature, which prevents us from unilaterally changing the HTLC-success transaction
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
          OP_ELSE
            # funds go to the remote node after an absolute delay (timeout)
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
          OP_ENDIF
        OP_ENDIF
      "
    },
    {
      "value": 0.04,
      "scriptPubKey": "
        # Another pending HTLC (payment) sent to us
        OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
        OP_IF
          # funds go to anyone who has the revocation key
          OP_CHECKSIG
        OP_ELSE
          <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
          OP_IF
            # funds go to us via a second-stage HTLC-success transaction once we have the payment preimage
            # NB: we also need the remote signature, which prevents us from unilaterally changing the HTLC-success transaction
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
          OP_ELSE
            # funds go to the remote node after an absolute delay (timeout)
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
          OP_ENDIF
        OP_ENDIF
      "
    },
  ]
}
```

A few important points to note:

* there can be many incoming and outgoing pending payments (HTLCs) inside the commitment transaction
* knowledge of the revocation key allows a participant to claim all the funds in the channel, so
  it's very important to make sure you don't publish an outdated commitment transaction
* it's always better if it's the other side that publishes its commitment transaction: HTLCs can be
  directly fulfilled or timed out without the need for a second-stage transaction
* these incentives ensure that you cooperatively close channels instead of publishing the commitment
  transaction: the only case where you should publish your commitment transaction is if the remote
  has become unresponsive or behaves maliciously

### HTLC transactions

The HTLC transactions are second-stage transactions that spend HTLC outputs from your commitment
transaction. They require `SIGHASH_ALL` signatures from both participants. This second stage is
necessary to decouple the HTLC timeout (an absolute block height) from the relative timeout applied
to local funds (`to_self_delay`); otherwise we wouldn't be able to fulfill or fail HTLCs before the
`to_self_delay` period.

An HTLC-success transaction looks like:

```json
{
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "txid": "...",
      "vout": 42,
      "scriptSig": "0 <remotehtlcsig> <localhtlcsig> <payment_preimage>",
      "sequence": 0
    }
  ],
  "vout": [
    {
      "value": 0.04,
      "scriptPubKey": "
        OP_IF
          # funds go to anyone who has the revocation key
          <revocationpubkey>
        OP_ELSE
          # or back to us after a relative delay (<to_self_delay>)
          `to_self_delay`
          OP_CHECKSEQUENCEVERIFY
          OP_DROP
          <local_delayedpubkey>
        OP_ENDIF
        OP_CHECKSIG
      "
    }
  ]
}
```

An HTLC-timeout transaction looks like:

```json
{
  "version": 2,
  "locktime": <cltv_expiry>,
  "vin": [
    {
      "txid": "...",
      "vout": 42,
      "scriptSig": "0 <remotehtlcsig> <localhtlcsig> <>",
      "sequence": 0
    }
  ],
  "vout": [
    {
      "value": 0.04,
      "scriptPubKey": "
        OP_IF
          # funds go to anyone who has the revocation key
          <revocationpubkey>
        OP_ELSE
          # or back to us after a relative delay (<to_self_delay>)
          `to_self_delay`
          OP_CHECKSEQUENCEVERIFY
          OP_DROP
          <local_delayedpubkey>
        OP_ENDIF
        OP_CHECKSIG
      "
    }
  ]
}
```

Once HTLC transactions are confirmed, their outputs can be spent simply with your (unilateral)
signature, which allows you to send the funds back to your wallet whenever is convenient.

### Closing transaction

You have probably noticed that using the commitment transaction and then HTLC transactions to
transfer off-chain funds back on-chain is complex and expensive (this will cost quite a lot of
on-chain transaction fees). This is meant to be the exceptional case where your peer isn't
collaborating. The recommended way of closing a channel to transfer funds back on-chain is via a
closing transaction that is built collaboratively.

When one of the channel participants initiates a channel close, the channel stops accepting new
payments. Both sides wait and settle pending payments (either fulfill or fail) to remove all HTLC
outputs from the commitment transaction. Once there are only 2 outputs remaining (one for each
participant, representing its latest balance in the channel), participants exchange adresses to
send the funds to (which removes the relative `to_self_delay`, allowing channel funds to be spent
immediately and unilaterally on-chain).

The resulting closing transaction looks like:

```json
{
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "txid": "...",
      "vout": 42,
      "scriptSig": "0 <signature_for_pubkey1> <signature_for_pubkey2>",
      "sequence": 0xFFFFFFFF
    }
  ],
  "vout": [
    {
      "value": 0.4,
      "scriptPubKey": "an address provided by A"
    },
    {
      "value": 0.6,
      "scriptPubKey": "an address provided by B"
    }
  ]
}
```

The closing transaction spends directly from the funding transaction, hiding all the complexity of
the commitment transaction and HTLC transactions. The resulting on-chain footprint of the channel
looks like:

```ascii
+------------+
| funding tx |
+------------+
       |
       |          +------------+
       +--------->| closing tx |
                  +------------+
                        | |
                        | |
                        | +-----> A's output
                        |
                        +-------> B's output
```

NB: the `sequence` is set to `0xFFFFFFFF` to ensure all timelocks are disabled.

## Future work

Now that you know what Lightning transactions look like today, let's explore what they might look
like in the near future. The current transactions format works great *if the following assumptions
hold*:

1. nodes are receiving (on-chain) blocks without too much delay
2. published transactions are quickly included in blocks

But what happens if your peer becomes unresponsive and suddenly on-chain fees rise tremendously?
Because the commitment transaction and HTLC transactions require signatures from both participants,
you are unable to raise the fee; your transaction will only be confirmed once on-chain fees go back
to lower levels. Our second condition doesn't hold true anymore and this can be a real issue since
HTLCs timeout at a specific, absolute block height.

You need to be able to increase the fee of your commitment transaction and HTLC transactions to
incentivize miners to include them in a block. Bitcoin provides two mechanisms for that, that we
will explain below: replace-by-fee (RBF) and child-pays-for-parent (CPFP).

RBF allows the *spender* to increase fees; CPFP allows the *recipient* to increase fees.

### Replace by fee

RBF lets you replace a mempool transaction. But replacing transactions has a cost for the network
because the new version of the transaction needs to propagate to the whole network. In order to
avoid creating an easy DoS attack vector, RBF imposes strict restrictions on the new version of the
transaction, detailed in [BIP 125](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki).

These rules have quite subtle consequences so it's worth reading the BIP in details. The gist of it
is that the new transaction must pay strictly more fees than the whole unconfirmed "package" it
replaces (otherwise miners would have no incentive to favor this new transaction).

RBF can also be explicitly disabled at the transaction level by setting the `sequence` of all
inputs to `0xFFFFFFFF`.

### Child pays for parent

CPFP lets you attach a child transaction to an unconfirmed transaction. By paying a big fee on the
child transaction, you can incentivize miners to include both the parent and the child transactions
in a block, speeding up the confirmation of the parent transaction.

If the child transaction still doesn't pay enough fees to get the transactions confirmed quickly
(for example because the fees kept rising), the combination of RBF and CPFP can help you unblock
the situation. You can use RBF on your child transaction (or on the parent if you're able to, but
usually you won't be able to produce the signatures to replace the parent transaction), or use CPFP
by appending yet another child to the chain of unconfirmed transactions.

```ascii
 mempool
+------------------------------------------------------------+
|                     +---------------+                      |
|   +---------+       | some other tx |                      |
|   | some tx |       +---------------+                      |
|   +---------+                                              |
|                                                            |
| +---------+            +-----------+                       |
| | tx with |----------->| child tx1 |                       |
| | low fee |----+       +-----------+                       |
| +---------+    |       +-----------+       +-----------+   |
|                +------>| child tx2 |------>| child tx3 |   |
|                        +-----------+       +-----------+   |
+------------------------------------------------------------+
```

There is however a limit on the chain of unconfirmed transactions you'll be able to add to the
mempool. Once that limit is reached, the only options to speed up confirmation are:

* wait for some of the transactions to be evicted from the mempool (then you can re-submit them
  with a higher fee): this can take a while
* RBF one of the transactions: because of the RBF rules, this may be very costly (the fee increase
  you will need to provide may be non-negligible)
* leverage the [CPFP carve-out rule](https://bitcoinops.org/en/topics/cpfp-carve-out/): this is a
  special rule that was added especially to unblock that scenario, which allows you to add a last
  additional child transaction, but only if its only parent is the first unconfirmed transaction.
  This means the first unconfirmed transaction needs to have an output available for that purpose.

```ascii
 mempool: the chain txA -> txB -> txC has the maximum allowed size
+------------------------------------------------------------+
|                     +---------------+                      |
|   +---------+       | some other tx |                      |
|   | some tx |       +---------------+                      |
|   +---------+                                              |
|                                                            |
| +---------+      +-----------+      +-----------+          |
| |   txA   |----->| child txB |----->| child txC |          |
| | low fee |      +-----------+      +-----------+          |
| +---------+                                                |
+------------------------------------------------------------+

 the carve-out rule allows us to bypass the size limit and add the child txD
+------------------------------------------------------------+
|                     +---------------+                      |
|   +---------+       | some other tx |                      |
|   | some tx |       +---------------+                      |
|   +---------+                                              |
|                                                            |
| +---------+      +-----------+      +-----------+          |
| |   txA   |----->| child txB |----->| child txC |          |
| | low fee |--+   +-----------+      +-----------+          |
| +---------+  |   +-----------+                             |
|              +-->| child txD |                             |
|                  +-----------+                             |
+------------------------------------------------------------+
```

NB: when using CPFP, the child transaction you are adding must be valid (miners must be able to
potentially include it in the next block). If your child transaction has an absolute or relative
timelock set in the future, it will be rejected. In particular, if your child transaction has a
relative timelock on a parent transaction that is still in the mempool, you cannot use it for CPFP.

### Bumping Lightning transactions

As we've seen previously, there are 5 types of transactions in Lightning:

* funding transaction
* commitment transaction
* htlc-success
* htlc-timeout
* closing transaction

At a high-level, this table summarizes which can be RBF-ed or CPFP-ed *unilaterally*:

| Transaction  | RBF                 | CPFP                |
| ------------ | ------------------- | ------------------- |
| funding      | :heavy_check_mark:* | :heavy_check_mark:  |
| commitment   | :x:                 | :heavy_check_mark:* |
| htlc-success | :x:                 | :x:                 |
| htlc-timeout | :x:                 | :x:                 |
| closing      | :x:                 | :heavy_check_mark:  |

The funding transaction can be RBF-ed (since it's created completely unilaterally), but this will
invalidate its transaction id so you'll need to start a new channel creation flow, which is quite
cumbersome. It's most of the time a better idea to use CPFP if you want to make your funding tx
confirm faster.

The commitment transaction can be CPFP-ed, but only by the remote node (via its main output, the
revocation key or an HTLC output for which it has the preimage). All other outputs use relative
timelocks, so they cannot be used for CPFP.

None of the transactions can be *unilaterally* RBF-ed because they need signatures from both sides.

This is quite inconvenient if you've been forced to broadcast your commitment transaction because
the remote node was unresponsive and it's stuck in the mempool. It would be nice to be able to make
it confirm faster, and this has been an area of active research.

### Anchor outputs

The [anchor outputs proposal](https://github.com/lightningnetwork/lightning-rfc/pull/688) tries to
address these shortcomings by leveraging the CPFP carve-out rule and a better use of sighash flags.

At a high-level, it consists of three main changes:

1. Update HTLC transactions to use `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY`. This lets you RBF your
  HTLC transactions by adding new inputs/outputs to raise the fees.
2. Add new outputs (called `anchor outputs`) to the commitment transaction specifically for CPFP
  carve-out. This lets you do CPFP on a commitment transaction that's stuck in the mempool.
3. Add a relative timelock of 1 block to all outputs of the commitment transaction except the new
  `anchor outputs`. This ensures only these two outputs can be used for the CPFP carve-out; all
  other outputs cannot be used for CPFP, they have to wait for the commitment transaction to be
  confirmed before they can be spent.

Let's focus on the first point. An HTLC-success transaction will look like:

```json
{
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "txid": "...",
      "vout": 42,
      "scriptSig": "0 <remotehtlcsig> <localhtlcsig> <payment_preimage>",
      "sequence": 1
    }
  ],
  "vout": [
    {
      "value": 0.04,
      "scriptPubKey": "
        OP_IF
          # funds go to anyone who has the revocation key
          <revocationpubkey>
        OP_ELSE
          # or back to us after a relative delay (<to_self_delay>)
          `to_self_delay`
          OP_CHECKSEQUENCEVERIFY
          OP_DROP
          <local_delayedpubkey>
        OP_ENDIF
        OP_CHECKSIG
      "
    }
  ]
}
```

The only difference with the current format is the `sequence` set to `1` (instead of `0`) and that
the remote signature uses `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` (but the local signature MUST use
`SIGHASH_ALL` to prevent others from malleating the transaction after you broadcast it). This lets
you replace the transaction with the following, without invalidating the signatures:

```json
{
  "version": 2,
  "locktime": 0,
  "vin": [
    {
      "txid": "...",
      "vout": 42,
      "scriptSig": "0 <remotehtlcsig> <localhtlcsig> <payment_preimage>",
      "sequence": 1
    },
    {
      "txid": "...",
      "vout": ...,
      "scriptSig": "...",
      "sequence": ...
    }
  ],
  "vout": [
    {
      "value": 0.04,
      "scriptPubKey": "
        OP_IF
          # funds go to anyone who has the revocation key
          <revocationpubkey>
        OP_ELSE
          # or back to us after a relative delay (<to_self_delay>)
          `to_self_delay`
          OP_CHECKSEQUENCEVERIFY
          OP_DROP
          <local_delayedpubkey>
        OP_ENDIF
        OP_CHECKSIG
      "
    },
    {
      "value": ...,
      "scriptPubKey": "..."
    }
  ]
}
```

By attaching an input of amount `N` and an output of amount `M`, you are adding `N-M` fees to the
transaction (you can of course attach multiple inputs/outputs). This lets you arbitrarily raise the
fees and ensure the HTLC transaction gets confirmed in time.

The second and third points change the format of the commitment transaction to:

```json
{
  "version": 2,
  "locktime": 543210000,
  "vin": [
    {
      "txid": "...",
      "vout": ...,
      "scriptSig": "0 <signature_for_pubkey1> <signature_for_pubkey2>",
      "sequence": 2500123456
    }
  ],
  "vout": [
    {
      "value": 0.5,
      "scriptPubKey": "
        # to_local output
        OP_IF
          # funds go to anyone who has the revocation key
          <revocationpubkey>
        OP_ELSE
          # or back to us after a relative delay (<to_self_delay>)
          <to_self_delay>
          OP_CHECKSEQUENCEVERIFY
          OP_DROP
          <local_delayedpubkey>
        OP_ENDIF
        OP_CHECKSIG
      "
    },
    {
      "value": 0.3,
      "scriptPubKey": "
        # to_remote output: goes back to the other channel participant after 1 block
        <remote_pubkey>
        OP_CHECKSIGVERIFY
        1 OP_CHECKSEQUENCEVERIFY
      "
    },
    {
      "value": 0.00000330,
      "scriptPubKey": "
        # our anchor output: used by us to bump the fee with CPFP
        <local_funding_pubkey>
        OP_CHECKSIG
        OP_IFDUP
        OP_NOTIF
          # after a relative timelock of 16 blocks, anyone can claim this tiny amount
          OP_16 OP_CHECKSEQUENCEVERIFY
        OP_ENDIF
      "
    },
    {
      "value": 0.00000330,
      "scriptPubKey": "
        # remote anchor output: used by them to bump the fee with CPFP
        <remote_funding_pubkey>
        OP_CHECKSIG
        OP_IFDUP
        OP_NOTIF
          # after a relative timelock of 16 blocks, anyone can claim this tiny amount
          OP_16 OP_CHECKSEQUENCEVERIFY
        OP_ENDIF
      "
    },
    {
      "value": 0.05,
      "scriptPubKey": "
        # A pending HTLC (payment) sent by us
        OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
        OP_IF
          # funds go to anyone who has the revocation key
          OP_CHECKSIG
        OP_ELSE
          <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
          OP_NOTIF
            # funds go back to us via a second-stage HTLC-timeout transaction (which contains an absolute delay)
            # NB: we also need the remote signature, which prevents us from unilaterally changing the HTLC-timeout transaction
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
          OP_ELSE
            # funds go to the remote node if it has the payment preimage.
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
          OP_ENDIF
          # add a 1-block delay to prevent this output from being used for CPFP
          1 OP_CHECKSEQUENCEVERIFY OP_DROP
        OP_ENDIF
      "
    },
    {
      "value": 0.08,
      "scriptPubKey": "
        # A pending HTLC (payment) sent to us
        OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
        OP_IF
          # funds go to anyone who has the revocation key
          OP_CHECKSIG
        OP_ELSE
          <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
          OP_IF
            # funds go to us via a second-stage HTLC-success transaction once we have the payment preimage
            # NB: we also need the remote signature, which prevents us from unilaterally changing the HTLC-success transaction
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
          OP_ELSE
            # funds go to the remote node after an absolute delay (timeout)
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
          OP_ENDIF
          # add a 1-block delay to prevent this output from being used for CPFP
          1 OP_CHECKSEQUENCEVERIFY OP_DROP
        OP_ENDIF
      "
    },
  ]
}
```

If you squint really hard, you'll see that the change in the output scripts is only the addition
of `1 OP_CHECKSEQUENCEVERIFY` to the outputs that were previously spendable without any delay.

This ensures that there are **only two outputs** that can be used for CPFP: the newly added anchors.
One of them can be used by us, the other by the remote participant. This ensures that whatever a
malicious participant may do to prevent the commitment transaction from confirming in time, the
honest participant always has an opportunity to leverage the CPFP carve-out rule to bump the fee
at least once.

The anchor outputs have a relative timelock branch that allows anyone to spend them. The reason for
this is that we want to be nice to the Bitcoin layer and avoid creating many tiny UTXOs that would
bloat the UTXO set. By letting anyone spend them, there's a chance they will be consolidated when
fees are low.

The overall transaction graph now looks like:

```ascii
+------------+
| funding tx |
+------------+
      |
      |        +-------------------+
      +------->|    commit tx B    |
               +-------------------+
                  |  |  |  |  |  |  
                  |  |  |  |  |  | A's main output
                  |  |  |  |  |  +-----------------> to A after a 1-block relative delay
                  |  |  |  |  |
                  |  |  |  |  |                 +---> to B after relative delay
                  |  |  |  |  | B's main output |
                  |  |  |  |  +-----------------+
                  |  |  |  |                    |
                  |  |  |  |                    +---> to A with revocation key
                  |  |  |  |
                  |  |  |  | A's anchor output
                  |  |  |  +--------------------> to A immediately (or anyone after 16-block relative delay)
                  |  |  |
                  |  |  | B's anchor output
                  |  |  +-----------------------> to B immediately (or anyone after 16-block relative delay)
                  |  |
                  |  |     (B's RBF inputs) ---+
                  |  |                         |                                    +---> to B after relative delay
                  |  |                         +---->+-----------------+            |
                  |  |                   +---------->| HTLC-timeout tx |------------+
                  |  | HTLC offered by B |           +-----------------+            |
                  |  +-------------------+      (after timeout + 1-block delay)     +---> to A with revocation key
                  |                      |
                  |                      +---> to A with payment preimage after a 1-block relative delay
                  |                      |
                  |                      +---> to A with revocation key
                  |
                  |        (B's RBF inputs) ---+
                  |                            |                                        +---> to B after relative delay
                  |                            +---->+-----------------+                |
                  |                    +------------>| HTLC-success tx |----------------+
                  | HTLC received by B |             +-----------------+                |
                  +--------------------+     (with payment preimage + 1-block delay)    +---> to A with revocation key
                                       |
                                       +---> to A after timeout (absolute delay + 1-block relative delay)
                                       |
                                       +---> to A with revocation key
```

### RBF Pinning

While the anchor outputs proposal solves the issue of a slow-to-confirm commitment transaction, it
doesn't solve other issues related to transaction pinning and may even provide more attack surface
to steal HTLC outputs.

This [mail thread](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002639.html)
details these issues and starts the discussion around alternative proposals.

The simplest incarnation of the attack works as follows:

* The attacker (Mallory) sets up two channels with her target (Alice): `Mallory -> Alice -> Mallory`
* Mallory then proceeds to send a payment to herself via these two channels
* When she receives the HTLC, she stops responding and waits for the timeout, forcing Alice to
  publish her commitment transaction that contains an htlc-offered output
* Mallory will then publish a transaction to claim that htlc output (revealing the preimage), with
  a low fee and RBF disabled
* If Mallory's transaction gets into miners mempools before Alice's htlc-timeout transaction, Alice
  won't be able to claim the funds back because she can't replace Mallory's transaction
* If Alice isn't able to retrieve the preimage (because her own mempool contains her htlc-timeout
  transaction), she will have to let the upstream htlc expire. When Mallory's transaction
  eventually gets mined, Alice will have paid the htlc amount downstream but not received the
  corresponding amount upstream

Note that this attack is hard to carry out in practice: ensuring your transaction gets in more
mempools than your target's transaction without revealing the preimage to your target is a non
trivial task. But the anchor outputs proposal makes it slightly simpler than before. Without the
1-block relative delay, Alice was able to broadcast the commitment transaction and her htlc-timeout
transaction together: this gave her a head-start to get her htlc-timeout in nodes' mempools before
Mallory. With anchor outputs she has to wait for the commitment transaction to be confirmed before
she is able to broadcast the htlc-timeout; she doesn't have a head-start anymore.

The main reason this attack is possible is because Mallory has no constraints on how she can spend
htlc outputs in Alice's commitment transaction. The linked email thread proposes to make this more
symmetric, by only allowing Mallory to spend htlc outputs via a pre-signed transaction that requires
signatures from both participants, and adding anchor outputs to HTLC transactions.

### Alternative proposal

Let's summarize our requirements and detail a proposal that combines ideas from the two previous
sections.

Let's define the following constants (configurable by implementations):

```ascii
received htlc expiry                                offered htlc expiry
        |                                                     |
        |                                                     |
        v                                                     v
        +-----------------------------------------------------+
                           cltv_expiry_delta
        <----------------------------------------------------->
            fulfill_safety_delta                 N
        <---------------------------><------------------------>
```

When we broadcast our commitment transaction, we need to handle the following cases:

* Offered HTLC times out: in that case, we want to ensure that either:
  * our HTLC-timeout tx confirms in less than `N` blocks
  * or we learn the HTLC preimage before `N` blocks
* Received HTLC should be fulfilled, but times out in less than `fulfill_safety_delta`. We want to
  ensure that our HTLC-success tx confirms (prevent remote from spending it via the timeout branch)

In case the remote participant broadcasts their commitment transaction, we want to ensure that:

* If we have the preimage for a received HTLC (offered in their commitment) before the timeout,
  we're able to claim the output (prevent remote from spending it via their HTLC-timeout tx)
* If an offered HTLC times out (received in their commitment), we must either:
  * claim the HTLC output in less than `N` blocks
  * or learn the HTLC preimage before `N` blocks

The current transaction format lets us easily handle the second case (when the remote broadcasts
their commitment transaction), but not the first one.

In order to leverage the CPFP carve-out rule on HTLC transactions, we need to be able to confirm
the commitment transaction quickly. That means we need the commitment transaction to have two
anchors and all other outputs with a 1-block relative timelock.

The HTLC-timeout transaction can use `SIGHASH_SINGLE | SIGHASH_ANYONECANPAY` for the remote
signature, which lets us RBF it. The HTLC-success transaction however must use `SIGHASH_ALL` and
leverage anchor outputs to bump the fees with CPFP. A new transaction is necessary when the remote
claims an HTLC output by providing the preimage: we call it Remote-HTLC-success transaction.
It must use `SIGHASH_ALL` and have an output for each participant to allow CPFP carve-out.
No additional transaction seems required for claiming the timeout branch of a received HTLC.

The trick to protect against a malicious participant that broadcasts a low-fee HTLC-success or
Remote-HTLC-success transaction is that we can always blindly do a CPFP carve-out on them; we know
their txid (because they use `SIGHASH_ALL`) so if we observe that our HTLC-timeout (or our claim of
the timeout branch of a received HTLC) doesn't confirm, we can try to broadcast a transaction that
spends our (potential) anchor output on the success transaction. The goal is to make it confirm
quickly if it exists so that we learn the preimage and can adapt our upstream behavior in time.

```ascii
+------------+
| funding tx |
+------------+
      |
      |        +-------------------+
      +------->|    commit tx B    |
               +-------------------+
                  |  |  |  |  |  |  
                  |  |  |  |  |  | A's main output
                  |  |  |  |  |  +-----------------> to A after a 1-block relative delay
                  |  |  |  |  |
                  |  |  |  |  |                 +---> to B after relative delay
                  |  |  |  |  | B's main output |
                  |  |  |  |  +-----------------+
                  |  |  |  |                    |
                  |  |  |  |                    +---> to A with revocation key
                  |  |  |  |
                  |  |  |  | A's anchor output
                  |  |  |  +--------------------> to A immediately (or anyone after 16-block relative delay)
                  |  |  |
                  |  |  | B's anchor output
                  |  |  +-----------------------> to B immediately (or anyone after 16-block relative delay)
                  |  |
                  |  |     (B's RBF inputs) ---+
                  |  |                         |                                    +---> to B after relative delay
                  |  |                         +---->+-----------------+            |
                  |  |                   +---------->| HTLC-timeout tx |------------+
                  |  | HTLC offered by B |           +-----------------+            |
                  |  +-------------------+      (after timeout + 1-block delay)     +---> to A with revocation key
                  |                      |
                  |                      |           +------------------------+
                  |                      +---------->| Remote-HTLC-success tx |
                  |                      |           +------------------------+
                  |                      |     (with payment preimage + 1-block delay)
                  |                      |                      |  |
                  |                      |                      |  | A's output
                  |                      |                      |  +-----------------> to A immediately (or anyone after 16-block relative delay)
                  |                      |                      |
                  |                      |                      | B's anchor output
                  |                      |                      +--------------------> to B immediately (or anyone after 16-block relative delay)
                  |                      |
                  |                      +---> to A with revocation key
                  |
                  |                                  +-----------------+
                  |                    +------------>| HTLC-success tx |
                  | HTLC received by B |             +-----------------+
                  +--------------------+     (with payment preimage + 1-block delay)
                                       |                   |  |  |                     +---> to A with revocation key
                                       |                   |  |  | B's output          |
                                       |                   |  |  +---------------------+---> to B after relative delay
                                       |                   |  |
                                       |                   |  | A's anchor output
                                       |                   |  +--------------------> to A immediately (or anyone after 16-block relative delay)
                                       |                   |
                                       |                   | B's anchor output
                                       |                   +-----------------------> to B immediately (or anyone after 16-block relative delay)
                                       |
                                       +---> to A after timeout (absolute delay + 1-block relative delay)
                                       |
                                       +---> to A with revocation key
```

Notes:

* This raises quite significantly the minimum HTLC value that is economical to enforce on-chain,
  which is quite sad, and it makes the transactions more complex/costly
* The blind CPFP carve-out is a one shot, so you'll likely need to pay a lot of fees for it to work
  which still makes you lose money in case an attacker targets you (but the money goes to miners,
  not to the attacker - unless he is the miner). It's potentially hard to estimate what fee you
  should put into that blind CPFP carve-out because you have no idea what the current fee of the
  success transaction package is (if the attacker created a long chain of unconfirmed transactions)
* If we take a step back, the only attack we need to protect against is an attacker pinning a
  preimage transaction while preventing us from learning that preimage for at least `N` blocks.
  If we have:
  * a high enough `cltv_expiry_delta` (and thus a high enough `N` value)
  * [off-chain preimage broadcast](https://github.com/lightningnetwork/lightning-rfc/issues/783)
  * LN hubs (or any party commercially investing in running a lightning node) participating in
    various mining pools to help discover preimages
  * decent mitigations for eclipse attacks
  * then the official anchor outputs proposal should be safe enough?

## Resources

* [Bolt 3](https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md)
* [SIGHASH_NOINPUT](https://github.com/bitcoin/bips/blob/master/bip-0118.mediawiki)
* [SIGHASH_ANYPREVOUT](https://github.com/ajtowns/bips/blob/bip-anyprevout/bip-anyprevout.mediawiki)
* [OP_CHECKLOCKTIMEVERIFY](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki)
* [nSequence](https://github.com/bitcoin/bips/blob/master/bip-0068.mediawiki)
* [OP_CHECKSEQUENCEVERIFY](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki)
* [Opt-in RBF](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki)
* [CPFP carve-out](https://bitcoinops.org/en/topics/cpfp-carve-out/)
* [Anchor Outputs](https://github.com/lightningnetwork/lightning-rfc/pull/688)
* [Transaction Pinning](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-April/002639.html)
