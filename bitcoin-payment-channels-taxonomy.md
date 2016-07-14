# Bitcoin Payment Channels, A Taxonomy

## Introduction

The concept of a payment channel in Bitcoin has been around for several years, but recent changes to the protocol (**OP_CHECKLOCKTIMEVERIFY**, **OP_CHECKSEQUENCEVERIFY**, Segregated Witness) have made much more powerful constructions of payment channels possible. I haven't been able to find a resource that describes the different varieties of payment channels in full detail, and much of the available material is now obsolete.

This document is an attempt to fully describe the various kinds of payment channels that are possible in Bitcoin today. It is a top-to-bottom description of the operation of the channel including the order and exchange of transactions and the the full locking (*scriptPubKey*) and unlocking (*scriptSig*) scripts. The reader is assumed to have a knowledge of the format of bitcoin transactions and transaction outputs, the concept of pay-to-witness-script-hash and the workings of opcodes and the Script language. No prior knowledge of payment channels is assumed.

A few things this article doesn't cover:

- Applications of payment channels
- Historical constructions of payment channels (which have been obsoleted by the new opcodes)
- anything outside the transaction layer that is required for a fully functioning lighning network (eg routing, message exchange, payment protocol for exchanging payment requests, etc)

This document is necessarily a work in progress. The rate of innovation in this area is extremely rapid, and new varieties of payment channels will most likely continue to be developed. Please direct any feedback to [@jonnynewbs](http://www.twitter.com/jonnynewbs).

With that, let's get started!

## A brief overview

A payment channel is a series of Bitcoin transactions which are constructed, signed and exchanged off-chain by two counterparties, and only brodcast to the Bitcoin network (and therefore included in the blockchain) once the parties are ready to close the channel. This allows the balances between the two parties to be changed many times, while only resulting in two transactions on the blockchain - one to open the channel and one to close the channel. There are many reasons we'd want to do this:

- each on-chain transaction requires miner transaction fees. Updating the balances in the payment channel and only settling to the chain when the channel closes means the parties don't have to pay Bitcoin transaction fees many times over.
- Very small (micro) payments can be made within the channel. The Bitcoin network enforces a lower dust limit on transaction outputs, below which the transaction won't be relayed. Payment channel updates can change the balances in the channel by as little as 1 satoshi.
- Bitcoin transactions must be confirmed in at least one block before the recipient can be confident that the funds won't be double-spent. Blocks are mined on average every 10 minutes, so payments can take many minutes to be confirmed. In a payment channel, the funds are locked into the channel so the balances of both parties can be instantly updated with new commitment transactions within the channel.
- Bitcoin blocks have a size limit, which places a hard limit on the number of transactions the network can process per second (currently around 7 transactions per second). Since payment channels only consume two on-chain transactions, using payment channels where there are lots of balance changes between two parties would allow the Bitcoin network to scale to many more transactions per second.

All payment channel transactions are valid Bitcion transactions and can be dumped into the Bitcoin network at any time. Neither party holds any risk since she can close the channel out at any time and be guaranteed to receive her funds.

## Diagram style

Bitcoin transactions are collections of Transaction Inputs (TXIs) and Transaction Outputs (TXOs). The simplest Bitcoin transaction consists of a single TXI and a single TXO. The value of the TXO is equal to the value of the TXI (less the transaction fee):

![Simple Transaction](./Transaction_Diagram1.svg)

Of course, a TXI is simply an unspent TXO (a UTXO) from a previous transaction, the the TXO from this transaction will go on to be a TXI in a future transaction. We can draw a chain of transactions as follows:

![Transaction Chain](./Transaction_Diagram2.svg)

We're not interested in the transaction that funded our channel, just that it output a TXO which we use as our TXI. We also don't know how the TXO from our transaction will be used until it's included as the TXI in a future transaction. Our standard transaction ends up looking like this:

![Bare Transaction](./Transaction_Diagram3.svg)

Of course, most Transactions don't have just a single output. Here's an example transaction with two TXOs:

![Transaction with multiple TXOs](./Transaction_Diagram4.svg)

Finally, a single TXO can be spent in many ways. We can illustrate that with branching from the TXO:

![TXO with branching child transactions](./Transaction_Diagram5.svg)

This is slightly arbitrary of course, since the TXO could be spent in an infinite number of ways. However, it is instructive to see the different ways that we're expecting the TXO to be spent. This is important when we're constructing many transactions in a payment channel, each of which is overriding the previous transaction.

## Unidirectional, Fixed-duration Payment Channel

This is the simplest form of Payment Channel. It is **One-way**, **Simplex** or **Unidirectional**, which means that funds can only flow in one direction from payer to recipient. It is also **Fixed-duration**, which means that the payment channel has to be closed out with a *closing transaction* after a fixed time period. If the payer wishes to continue paying the recipient through a channel after this time, she must open a fresh payment channel with a new *anchor transaction*.

As is tradition, Alice is the payer and Bob is the recipient. Alice wishes to pay Bob in increments of 0.01 BTC up to a maximum of 1 BTC.

### Opening the channel

The Channel is opened with an anchor transaction, which is constructed and signed by Alice, and broadcast to the Bitcoin network. The anchor transaction has a single transaction output which can be spent either:

(1) with both Bob and Alice's signatures. This is the *spend* branch; or
(2) with Alice's signature after a channel expiration locktime (eg 24 hours). This is the *refund* branch.

Alice is providing all of the funds for the channel. The refund branch protects Alice from having those funds stranded or held to ransom inside the channel. The spend branch requires Bob's signature, so if there was no refund branch Bob could stop responding and Alice would have no way of reclaiming her funds. If Bob were malicious, he might hold Alice's funds to ransom and only agree to sign a transaction from the anchor transaction if it assigned the majority of funds to him.

The locking script is as follows:

```
OP_IF
  <Bob's public key> OP_CHECKSIGVERIFY # Spend branch - requires both signatures
OP_ELSE
  <channel timeout> OP_CHECKLOCKTIMEVERIFY OP_DROP # Refund branch - requires Alice's signature only after the channel expiration timeout
OP_ENDIF
<Alice's public key> OP_CHECKSIG # Both branches require Alice's signature
```

Before using the channel, the anchor transaction must be confirmed in the blockchain. If it isn't confirmed and Bob starts accepting payments through the channel, then Alice could double-spend the input to the anchor transaction and Bob would be left with nothing.

### Paying into the channel

To start paying into the channel, Alice constructs and signs a *commitment transaction* and sends it directly to Bob. This transaction has the anchor transaction output as its only input, and produces two outputs:

1. is for 0.01 BTC and can be spent with Bob's signature
2. is for 0.99 BTC and can be spent with Alice's signature

Both of those outputs can be standard P2PKHs. This is a valid Bitcoin transaction, and has been signed by Alice. Remember that the anchor transaction output required both Alice's and Bob's signatures, and Bob hasn't signed this transaction, so Alice don't have a full witness. The transaction can't be brodcast to the Bitcoin network until Bob has added his signature to the witness. Bob can close out the channel at any time by adding his signature to the witness for this commitment transaction and broadcasting it to the network.

Let's assume Bob doesn't close out the channel, and Alice wants to pay a further 0.01 BTC to Bob. She constructs and signs a second commitment transaction and sends it to Bob. This second transaction has exactly the same anchor transaction output as its input, but now produces the following outputs:

1. is for 0.02 BTC and can be spent with Bob's signature
2. is for 0.98 BTC and can be spent with Alice's signature

In effect, this is a double-spend of the anchor transaction output. Only one of those transactions can be included in the blockchain. Which will it be?

Alice would probably prefer for it to be commitment transaction 1 (or even better a refund transaction). However, the commitment transactions' witnesses require Bob's signature, which Alice doesn't have. The refund transaction can't be included until the channel has expired, so Alice can't broadcast that either. Alice therefore isn't able to broadcast anything.

How about Bob? He has commitment transaction 1 and commitment transaction 2, as well as Alice's signatures for both. He could add his own signature to either of those transactions and have a complete witness, which he could then broadcast to the network. He'd almost certainly prefer the transaction which gives him 0.02 BTC over the transaction which gives him 0.02, so he'll likely never need commitment transaction 1 and can throw it away. If he wants to close the channel at any time, he can now broadcast commitment transaction 2 instead.

Alice can continue paying Bob through the payment channel in this fashion. Each time she wants to pay another 0.01 BTC to Bob, she constructs a new commitment transaction from the same anchor transaction output and sends it to Bob. This continues until one of the following happens:

1. Bob wishes to close out the channel, and so signs and broadcasts the most recent transaction
2. the funds in the channel are exhausted and the most recent commitment transaction sends 1 BTC to Bob and 0 to Alice. At this point, Bob should just sign and broadcast that commitment transaction and collect the 1 BTC.
3. The payment channel expiry timeout is reached. At this point Alice can reclaim all of the funds in the channel. Bob should never let this happen, so should sign and broadcast the latest commitment transaction well before the expiry time.

### Redeeming a Commitment transaction

To redeem a commitment transaction, Bob broadcasts the transaction with the following witness:

```
<Alice's sig> <Bob's sig> 1
```


### Claiming the refund

After the payment channel expiry time, Alice an get a refund for the contents of the channel by constructing a refund transaction. This takes the anchor transaction output as its input and sends all the funds to her public key. The witness for the transation is:

```
<Alice's sig> 0
```

## Bidirectional, Fixed-duration Payment Channel
