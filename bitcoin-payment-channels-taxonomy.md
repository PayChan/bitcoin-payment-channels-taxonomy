# Bitcoin Payment Channels, A Taxonomy

# Table of Contents

-  1 Introduction
-  2 A brief overview
-  3 Diagram style
    -  3.1 Transaction graphs
        - Transaction types
    -  3.2 Message exchange diagrams
    -  3.3 Payment channel balance diagrams
-  4 Simple payment channel
    -  4.1 Opening the channel
    -  4.2 Updating the balances in the channel
    -  4.3 Closing the channel
        -  4.3.1 Redeeming a commitment transaction
        -  4.3.2 Redeeming the refund branch
    -  4.4 Advantages of simple payment channels
-  5 Two-way channels
    -  5.1 Revocable transactions
    -  5.2 SHA-trees
    -  5.3 Two-way payment channels using revocable transactions
    -  5.4 Closing the channel
        -  5.4.1 Closing the channel unilaterally
        -  5.4.2 Closing the channel co-operatively
        -  5.4.1 Redeeming the refund branch
-  6 Non-expiring payment channels
    -  6.1 Symmetric commitment states
    -  6.2 Opening the channel
    -  6.3 Updating the balance
    -  6.4 Closing the channel unilaterally
    -  6.5 Closing the channel co-operatively
-  7 Acknowledgements

## 1. Introduction

This document is an attempt to describe the various kinds of payment channels that are possible in Bitcoin with today's technology. It is a top-to-bottom description of the payment channel and covers:

- the operation of the channel including the opening (*anchor*) transaction, the commitment states and the channel closing conditions
- the order and exchange of messages for commitment state changes
- the full locking (*scriptPubKey*) and unlocking (*scriptSig*) scripts for all transactions

The reader is assumed to have a knowledge of the format of bitcoin transactions and transaction outputs, the concept of pay-to-[witness-]script-hash and the workings of opcodes and the Script language. No prior knowledge of payment channels is assumed.

A few things this article doesn't cover:

- Applications of payment channels
- Historical constructions of payment channels (which have been obsoleted by newer opcodes)
- Anything outside the transaction layer that is required for a fully functioning lightning network (eg routing, message exchange, protocol for exchanging payment requests, etc)

This document is necessarily a work in progress. The rate of innovation in this area is extremely rapid, and new varieties of payment channels will most likely continue to be developed. Please direct any feedback to [@jonnynewbs](http://www.twitter.com/jonnynewbs) or raise a ticket against [the github repo](http://www.github.com/paychan/bitcoin-payment-channels-taxonomy).

With that, let's get started!

## 2. A brief overview

A payment channel is a sequence of valid Bitcoin transactions which are constructed, signed and exchanged off-chain by two counterparties. The transactions are not broadcast to the Bitcoin network during the operation of the channel. Only when one or both of the parties are ready to close the channel is a single closing transaction broadcast. This allows the balances between the two parties to be updated many times while only resulting in two transactions on the blockchain: one to open the channel and one to close the channel.

There are many reasons we'd want to do this:

- **Low or zero fee transactions** Each on-chain transaction requires miner transaction fees. Updating the balances within the payment channel and only settling to the blockchain when the channel closes means the parties don't have to pay Bitcoin transaction fees each time the balances are updated.
- **Micropayments** Very small (micro) payments can be made within the channel. The Bitcoin network enforces a lower dust limit on transaction outputs, below which the transaction won't be relayed. Payment channel updates can change the balances in the channel by as little as 1 satoshi.
- **Instant payments** Bitcoin transactions must be confirmed in at least one block before the recipient can be confident that the funds won't be double-spent. Blocks are mined on average every 10 minutes, so payments can take many minutes or hours to be confirmed. In a payment channel, the funds are locked into the channel so the balances of both parties can be instantly updated with new commitment transactions within the channel.
- **Scalability** Bitcoin blocks have a size limit, which places a hard limit on the rate that the network can process transactions (currently around 7 transactions per second). Payment channels only consume two on-chain transactions, so using payment channels where there are lots of balance updates between two parties would allow the Bitcoin network to scale to many more transactions per second.

Payment channels exist as a sequence of *commitment states*. For a channel to be in a commitment state:

1. Both parties agree on their balances within the channel. This is the *consistency* property.
2. Either party can close the channel unilaterally by broadcasting a transaction to claim their full balance (although they may need to wait for a relative or absolute timelock before doing so). This is the *escape* property.
3. As long as the parties monitor the blockchain and act correctly, neither can be denied their full balance. This is the *safeguard* property.

To update the balances in the channel, the channel goes through a *commitment state change* or *commitment transition* and enters a new commitment state with updated balances. A payment channel will always be in a commitment state or a commitment transition between two commitment states.

The *consistency* property guarantees that the channel remains synchronized between both parties and that the channel never enters a state where the parties' balances are inconsistent. The *escape* property guarantees that at all times during channel's existence, both parties have an escape route to claim their balance. The *safeguard* property guarantees that by entering into a payment channel, neither party can be denied their full balance (so long as they act correctly).

Taken together, these properties ensure that payment channels do not require any trust between counterparies and mean that neither party takes on risk by entering into the channel.

#### A note on segregated witness

For more advanced types of payment channel, we need to avoid first or third party malleability. For this reason, all of the transactions described below are assumed to be segregated witness transactions (either P2WSH or P2WPK).

The format for P2WSH TXOs are defined in [BIP 141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki). For simplicity, we refer only to 'scripts' and 'signatures' in this article, and ignore the technical aspects of including those scripts and signatures in the spending transaction's witness program. Readers can just assume that the script locks the TXO and the signatures provide the requirements for unlocking the TXO.

Simple one-way payment channels *can* be implemented using standard P2SH without worrying about malleability. Again, we refer only to 'scripts' and 'signatures' for the locking and unlocking conditions.

## 3. Diagram style

#### 3.1 Transaction graphs

Bitcoin transactions are collections of Transaction Inputs (TXIs) and Transaction Outputs (TXOs). The simplest Bitcoin transaction consists of a single TXI and a single TXO. The value of the TXO is equal to the value of the TXI (less the transaction fee):

![Simple Transaction](./Basic_Transaction1.svg)

A TXI is simply an unspent TXO (a UTXO) from a previous transaction, and the TXO from this transaction will go on to be a TXI in a future transaction. We can draw a chain of transactions as follows:

![Transaction Chain](./Basic_Transaction2.svg)

Often we're not interested in the transaction that funded our channel, just that there's a UTXO which we use as our TXI. We also don't know how the TXO from our transaction will be spent until it's included as a TXI in a future transaction. Our standard transaction ends up looking like this:

![Bare Transaction](./Basic_Transaction3.svg)

Of course, most Transactions don't have just a single output. Here's an example transaction with two TXOs:

![Transaction with multiple TXOs](./Basic_Transaction4.svg)

Finally, a single TXO can be spent in many ways. We can illustrate that with branching from the TXO:

![TXO with branching child transactions](./Basic_Transaction5.svg)

This is slightly arbitrary since the TXO could be spent in an infinite number of ways. However, it is instructive to see the different ways that we're expecting the TXO to be spent. In the following payment channels we'll be constructing many commitment transactions, each of which is spending the same TXO, so this notation will be useful.

##### Transaction types

If a transaction has been broadcast to the Bitcoin network, we'll colour it green:

![Broadcast Transaction](./Transaction_Types1.svg)

We're not going to consider blockchain confirmations in this document. As with all Bitcoin transactions, users should wait for a transaction to be confirmed before building on top of it, but whether they actually do, and how many confirmations they wait for is an implementation decision. For the rest of the article we'll assume that all broadcast transactions are confirmed and will not be double-spent or invalidated.

If a transaction has been constructed and one party has a valid witness for the transaction, then we colour it yellow:

![Unbroadcast Transaction](./Transaction_Types2.svg)

This transaction can be broadcast to the network (potentially after a relative or absolute locktime) by at least one party to close the channel.

There may be several unbroadcast transactions using the same TXOs. We'll keep the most recent one coloured yellow, but the older transactions will be a lighter yellow to show that we're not expecting them to be used:

![Overwritten Transaction](./Transaction_Types3.svg)

Once a transaction has a valid witness, that witness is valid forever. However, the transaction can be invalidated by creating and broadcasting a different transaction which spends the same TXO that our original transaction was spending. We'll colour transactions that have been invalidated in this way red:

![Invalidated Transaction](./Transaction_Types4.svg)

#### 3.2 Message exchange diagrams

Message exchange is depicted using ladder diagrams. As well as showing message exchange between the participants in a channel, the ladder diagram can also show one of the participants broadcasting a message to the blockchain:

![Ladder Diagram](./Ladder_Diagram.svg)

In practice, when we show Alice sending a 'signed transaction' to Bob within the channel, Alice may only be sending a signature. As long as Bob knows what the transaction format should be, there's no need for Alice to send the full transaction to Bob.

#### 3.3 Payment Channel Balance Diagrams

Payment Channel Balance Diagrams are used to show the balance within a payment channel.

Alice and Bob start with their own stock of Bitcoin:

![Payment Channel Diagram 1](./Payment_Channel_Diagram1.svg)

Alice funds a new payment channel with 20 BTC. Those funds are hers, but are temporarily locked in the payment channel. The balance of funds within the payment channel (indicated by the downward-pointing arrow) can move left and right as the parties create and exchange new commitment transactions:

![Payment Channel Diagram 2](./Payment_Channel_Diagram2.svg)

Alice then pays 10 BTC to Bob within the channel. The channel now contains a balance of 10 BTC for Alice and 10 BTC for Bob:

![Payment Channel Diagram 3](./Payment_Channel_Diagram3.svg)

Alice continues to pay Bob in the channel until her balance is 5 BTC and Bob's balance is 15 BTC:

![Payment Channel Diagram 4](./Payment_Channel_Diagram4.svg)

Finally, Alice closes out the channel, freeing up 5 BTC for herself and 15 BTC for Bob:

![Payment Channel Diagram 5](./Payment_Channel_Diagram5.svg)

## 4. Simple payment channel

This is the simplest form of Payment Channel. It is *One-way* (also called *Simplex* or *Unidirectional*), which means that funds can only flow in one direction from payer to recipient. It is also *Fixed-duration*, which means that the payment channel has to be closed with a *closing transaction* before a fixed expiry time. If the payer wishes to continue paying the recipient through a channel after this time, she must open a fresh payment channel with a new *anchor transaction*.

As is tradition, Alice is the payer and Bob is the recipient. Alice wishes to pay Bob in increments of 0.01 BTC up to a maximum of 1 BTC.

#### 4.1 Opening the channel

To open the channel, Alice constructs and signs an anchor transaction and then broadcasts it to the Bitcoin network. The anchor transaction has a single TXO which can be spent either:

1. with both Bob and Alice's signatures. This is the *spend* branch; or
2. with Alice's signature after a channel expiration duration (eg 24 hours). This is the *refund* branch.

![Simple Channel - Funded](./Simple_Channel1.svg)

Alice needs a refund branch to protect her funds from being stranded in the channel. This is required for the *escape* property of channels - both parties must have an escape route to reclaim their funds at all times in the channel's existence. If the anchor transaction didn't have a refund branch and was just a 2-of-2 multisig, then Alice wouldn't have an escape route.

Without an escape route Alice's funds could become stranded or held to ransom inside the channel. If Bob stops responding to Alice's messages (either inadvertently or maliciously), Alice would have no way to get her funds back. A malicious Bob might hold Alice's funds to ransom and only agree to unlock the multisig TXO in return for a ransom fee.

The script is as follows:

```
OP_IF
  <Bob's public key> OP_CHECKSIGVERIFY # Spend branch - requires both signatures
OP_ELSE
  <channel expiry duration> OP_CHECKSEQUENCEVERIFY OP_DROP # Refund branch - after the channel expiry duration only Alice's signature is required
OP_ENDIF
<Alice's public key> OP_CHECKSIG # Both branches require Alice's signature
```

To finish opening the channel, Alice constructs and signs the first *commitment transaction* (*CTx1*) and sends it directly to Bob. This transaction has the anchor transaction TXO as its only TXI, and produces two TXOs:

- TXO1 is for 0.01 BTC and can be spent with Bob's signature
- TXO2 is for 0.99 BTC and can be spent with Alice's signature

Both of those TXOs can be standard P2PKHs.

We're now in commitment state 1:

![Simple Channel - first commitment](./Simple_Channel2.svg)

Let's check the commitment state properties:

1. **Consistency:** Both parties agree that Alice's balance is 0.99 BTC and Bob's balance is 0.01 BTC.
2. **Escape:** Both parties have an escape route: Alice can escape by broadcasting her refund transaction and claiming 1 BTC (after the channel expiry duration), and Bob can escape by signing and brodcasting CTx1 and claiming his 0.01 BTC.
3. **Safeguard:** Both parties are guaranteed their full balance: Alice gets at least 0.99 BTC in both cases. Bob can guarantee he'll get 0.01 BTC by broadcasting CTx1 before Alice broadcasts the refund transaction. To make sure this happens, he simply needs to broadcast the CTx well before the channel expiry duration.

#### 4.2 Updating the balances in the channel

Let's assume that Bob hasn't closed out the channel and Alice wants to pay a further 0.01 BTC to Bob. She constructs and signs a second commitment transaction *CTx2* and sends it to Bob. This second transaction has exactly the same anchor transaction TXO as its TXI, but now produces the following TXOs:

- TXO1 is for 0.02 BTC and can be spent with Bob's signature
- TXO2 is for 0.98 BTC and can be spent with Alice's signature

Commitment state 2 looks like this:

![Simple Channel - second commitment](./Simple_Channel3.svg)

In effect, this is a double-spend of the anchor transaction TXO, so only one of those transactions can be included in the blockchain.

Let's check the commitment state properties again:

1. **Consistencyt:** Both parties now agree that Alice's balance is 0.98 BTC and Bob's balance is 0.02 BTC.
2. **Escape:** Both parties have an escape route: Alice can escape by broadcasting her refund transaction and claiming 1 BTC (after the channel expiry duration), and Bob can escape by signing and brodcasting one of the CTxs. He'll almost certainly prefer the transaction which gives him 0.02 BTC, so he can throw away CTx1 as soon as he receives CTx2.
3. **Safeguard:** Both parties are guaranteed their full balance: Alice gets at least 0.98 BTC in both cases. Bob can guarantee he'll get 0.02 BTC by broadcasting CTx2 (and not broadcasting CTx1!), and making sure CTx2 is broadcast before Alice broadcasts the refund transaction. Alice doesn't have Bob's signature for any of the CTxs, so can't broadcast an old CTx that pays Bob a lower balance.

Alice can continue paying Bob through the payment channel in this fashion. Each time she wants to pay another 0.01 BTC to Bob, she constructs a new CTx from the same anchor transaction output and sends it to Bob.

![Simple Channel - multiple commitments](./Simple_Channel4.svg)

#### 4.3 Closing the payment channel

There are two ways that the channel can be closed:

1. Bob signs and broadcasts a CTx.
2. The channel expires and Alice broadcasts a *Refund Transaction* or *RTx*.

##### 4.3.1 Redeeming a commitment transaction

Bob may choose to sign and broadcast his latest CTx to close the channel for a couple of reasons:

1. Alice has paid exhausted her funds in the channel (ie the most recent CTx sends 1 BTC to Bob and 0 to Alice).
2. Bob simply wishes to close out the channel and collect his balance from the most recent CTx.

In either case, he should make sure he signs and broadcasts the most recent CTx well before Alice does, to make sure the RTx doesn't steal the TXIs for his CTx.

Here's a diagram of Bob closing the channel after 4 commitment transactions:

![Simple Channel - ladder diagram](./Simple_Channel5.svg)

In this example:

1. Alice broadcasts the Anchor transaction to the network
2. Alice then constructs and signs a sequence of four CTxs, which she sends to Bob
3. Bob closes the channel by signing and broadcasting the most recent transaction CTx4

Bob broadcasts CTx4 transaction, which has the following signature:

```
<Alice's sig> <Bob's sig> 1
```

##### 4.3.2 Redeeming the refund branch

Bob should never let the channel expiry time pass without broadcasting a CTx. Once the channel is expired, Alice is able to broadcast the RTx and reclaim all the funds in the channel.

However, if Bob stops responding, Alice needs a way to reclaim the funds from the channel. She does this by constructing and broadcasting a refund transaction.

Here's a diagram of Bob disappearing after 2 CTxs, and Alice reclaiming the funds with an RTx:

![Simple Channel - ladder diagram 2](./Simple_Channel6.svg)

The refund transaction takes the anchor transaction TXO as its TXI and has the following signature:

```
<Alice's sig> 0
```

#### 4.4 Advantages of simple payment channels

An advantage of this style of payment channel is that it is extremely simple. The anchor TXO script is essentially either a 2-of-2 multisig in the spend branch, or a P2PKH with relative timelock in the refund branch. Commitment transitions are achieved simply by Alice constructing and signing new CTXs and sending them to Bob.

The channel is also almost entirely passive from Bob's point of view. He simply needs to keep hold of the most recent CTx, and then sign and broadcast it when he's ready to close the channel. Simple channels should be very straightforward for wallets and applications to implement.

## 5. Two-way channels

One of the most obvious limitations of the simple payment channel is that it is one-way. Bob's balance in the channel can only ever increase (and Alice's balance can only ever decrease). This is because every commitment transaction that Alice signs and sends to Bob is valid forever, or at least until one of them is broadcast and confirmed. Even if Alice constructs and signs a new CTx with a smaller balance for Bob, Bob will always be able to broadcast the CTx which assigns him the greatest balance. Alice has no way to stop Bob from doing this, and no way to invalidate the old commitment trasactions.

However, there is a trick that allows Alice to ensure that Bob can't use a previous commitment transaction to claim an old balance. This trick uses hash pre-images and timelocks to construct a *revocable* transaction. That's what we'll look at next.

#### 5.1 Revocable transactions

The trick to creating revocable transactions is to construct one of the TXOs such that it is either encumbered by:

- Bob's signature and a relative timelock (Bob's *spend branch*); or
- Alice's signature and a secret revocation hash provided by Bob (Alice's *revocation branch*).

To revoke the transaction, Bob reveals the pre-image of his secret revocation hash to Alice. If Bob ever broadcasts the revoked transaction, then Alice will have the chance to spend before him, since his spend branch is encumbered by a timelock and Alice's isn't. Bob is effectively blocked from being able to use the revoked transaction.

We'll call this type of TXO a *revocable TXO* or *rTXO* for short. The secret revocation hash is *h(rev)* and its pre-image is *rev*.

![Revocable Transaction](./Revocable_Transaction1.svg)

Transactions will normally have multiple TXOs. Within a payment channel, there will be one TXO for Alice's balance and an rTXO for Bob's balance (which can be spent by Alice when the rTXO is revoked):

![Full Revocable Transaction](./Revocable_Transaction2.svg)

Once Bob has revealed the revocation secret, he's no longer able to broadcast the revocable transaction since Alice would be able to collect her spend TXO as well as the rTXO. The transaction has been revoked:

![Revoked Transcation](./Revocable_Transaction3.svg)

We're going to use rTXOs *a lot* for more advanced channels, so it makes sense to have a special notation for them:

![Revocable Transaction - Notation](./Revocable_Transaction4.svg)

Conceptually, the combined up/down facing chevron indicates that the rTXO can be revoked.

The script for a revocable transaction is:

```
OP_IF # Bob's spend branch - after the revocation timeout duration, Bob can spend with just his signature
  <TXO revocation timeout duration> OP_CHECKSEQUENCEVERIFY OP_DROP
  <Bob's public key>
OP_ELSE # Revocation branch - once the revocation pre-image is revealed, Alice can spend immediately with her signature
  OP_HASH160 <h(rev)> OP_EQUALVERIFY OP_DROP
  <Alice's public key>
OP_ENDIF
OP_CHECKSIG
```

For Bob to spend the TXO, he needs to wait for the revocation timeout duration and then provide the following signature:

```
<Bob's sig> 1
```

If Bob broadcasts a revoked transaction, Alice can claim the revocation TXO by providing the following signature:

```
<Alice's sig> <rev> 0
```
#### 5.2 SHA-trees

The revocation hash is the 160 bit image of hashing a secret revocation pre-image first with SHA256 and then with RIPEMD-160. Bob needs to make sure that Alice can't guess the pre-images for the revocation hashes. If she could do that then she'd be able to steal the funds from the rTXO before Bob had revoked it.

The simplest way for Bob to create revocation secrets would be to use a pseudo-random number generator every time he needed to provide Alice with a new revocation hash. However, we're going to use revocation secrets in channels where we expect a large number of CTxs. Alice needs to store all the revocation secrets to ensure Bob doesn't broadcast an old revoked transaction, and the space required for Alice to store those pre-images would grow linearly with the number of commitment transactions in the channel.

Much better is if Bob uses a seed to generate the revocation secrets, and then shares information with Alice that would allow her to generate all the revocation pre-images that have been revealed so far.

One way to do this is to use a hash chain. Take a random seen and run SHA256 on it 1,000,000 times. Use the 1,000,000th hash as the first revocation pre-image, the 999,999th hash as the second pre-image and so on. Alice only needs to store the most recent pre-image, since she can derive all the other pre-images by hashing it repeatedly. The downside to this is that Bob needs to hash a value 1,000,000 times when the channel opens, and then either:

- store all 1,000,000 values (which is what we're trying to avoid!); or
- hash the same seed 999,999 times for the next revocation pre-image, and then hash the seed 999,998 times for the next, and so on. That's slow and computationally expensive.

A better way is to use a seed to construct an efficient binary tree of revocation secrets. The method for constructing and storing the sha-tree is described in https://github.com/rustyrussell/ccan/blob/master/ccan/crypto/shachain/design.txt.

Using this method:

- Bob can generate up to 2^64 revocation pre-images from a single 256-bit seed
- Bob has to do a maximum of 63 hashes, and an average of 1 hash, to generate a new revocation pre-image
- The number of revocation pre-images that Alice needs to store grow O(log n) with the number of transactions

#### 5.3 Two-way payment channels with revocable transactions

To create a two-way payment channel, Bob first needs to provide two revocation hashes *h(rev1)* and *h(rev2)* to Alice. Alice then constructs the anchor transaction exactly as before. The ATx's TXO can be spent either by a 2-of-2 multi-sig or by just herself after the channel expiry duration.

Alice completes the channel opening by constructing and signing CTx1 and sending it to Bob. The difference from the one-way payment channel is in the construction of the CTxs: instead of including a standard P2PKH for Bob's TXO, she includes an rTXO with a revocation hash *h(rev1)* provided by Bob.

The message exchange for channel opening is:

![Two-way Channel - Channel Opening](./Two_Way_Channel_Messages1.svg)

Commitment state 1 is as follows:

![Two-way Channel - First commitment](./Two_Way_Channel_Txs1.svg)

If Alice then wants to increase Bob's balance in the channel to 0.02 BTC, she constructs a new CTx which sends 0.98 BTC to herself and 0.02 BTC to Bob. The only difference from a simple channel is that the TXO for Bob in the CTx is an rTXO. Alice uses the second revocation hash *h(rev2)* to construct CTx2.

To acknowledge and commit the new CTx, Bob sends the revocation secret for CTx1 *rev1* along with a new revocation hash *hash(rev3)* so Alice can construct CTx3.

The message exchange for the first channel payment from Alice to Bob is:

![Two-way Channel - Alice pays Bob](./Two_Way_Channel_Messages2.svg)

Commitment state 2 is:

![Two-way Channel - Second commitment](./Two_Way_Channel_Txs2.svg)

Strictly speaking, Alice could use re-use the previous revocation hash to construct CTx2. That's because Alice only needs Bob to revoke an old CTx if her balance in the new CTx has increased. She doesn't need to revoke CTx1 (which gives her 0.99 BTC) since she'd be perfectly happy for Bob to broadcast CTx1 instead of CTx2 (which gives her 0.98 BTC).

However, we're not going to use that optimization. It makes things simpler just to use a new revocation secret for each new CTx. The sha-tree described in section 5.2 means that additional revocation secrets are cheap to generate and store, so the slight additional overhead in using unnecessary revocation hashes is acceptable for keeping the protocol simple.

Bob now wants to pay Alice 0.01 BTC and reduce his balance in the channel back to 0.01 BTC. Bob sends a request to Alice to create a new CTx with the new channel balances. Alice then constructs and signs CTx3 using *h(rev3)* and sends it to Bob. As before, Bob revokes the previous transaction and provides Alice with a new revocation hash.

The message exchange for Bob's channel payment to Alice is:

![Two-way Channel - Bob pays Alice](./Two_Way_Channel_Messages3.svg)

Commitment state 3 is:

![Two-way Channel - Third commitment](./Two_Way_Channel_Txs3.svg)

#### 5.4 Closing the channel

Two-way payment channel can be closed either unilaterally by Bob or co-operatively. As with a simple payment channel, Alice also has the option to close the channel using the refund branch if Bob disappears.

###### 5.4.1 Closing the channel unilaterally

Bob can close the channel at any point by signing and broadcasting the most recent CTx. Once the CTx is broadcast, he must then wait for the rTXO timeout duration before he's able to spend his output from the CTx.

The message exchange diagram below shows Bob unilaterally closing the channel after commitment state 3. Note that Alice has access to her funds as soon as Bob broadcasts the CTx, but Bob must wait for the revocation timeout delay.

![Two-way Channel - Bob closes unilaterally](./Two_Way_Channel_Messages4.svg)

###### 5.4.2 Closing the channel co-operatively

Bob may not want to wait for an rTXO timeout duration before spending his funds. If Alice is acting co-operatively, they can work together so he can get his funds as soon as the channel closes. The protocol for doing that is as follows:

1. Bob sends Alice a request to close the channel.
2. Alice constructs, signs and sends to Bob an unrevocable CTx with the same balances for herself and for Bob as the previous CTx. The difference is that Bob's TXO is now a simple P2PKHs instead of an rTXO. This is exactly the same as a cTXO in the simple payment channel.
3. Bob signs and broadcasts the unrevocable CTx and has instant access to his funds.

The message exchange diagram below shows Alice and Bob acting co-operatively to close the channel. Note that Alice and Bob both have access to the funds from the CTx immediately.

![Two-way Channel - Co-operative closing](./Two_Way_Channel_Messages5.svg)

Once Alice has constructed and signed an unrevocable CTx, she should consider the channel closed and not sign any further CTxs. If Bob doesn't broadcast it to the network, Alice **must** close the channel unilaterally and should not sign any more commitment transactions. That's because if she continues to use the channel and constructs a future CTx with a lower balance for Bob, she'd have no way to revoke the unrevocable CTx and Bob could steal funds by broadcasting an old CTx.

###### 5.4.3 Redeeming the refund branch

If the channel expiry time passes, Alice can close the channel using a refund transaction, in exactly the same way as for a simple channel.

## 6. Non-expiring payment channels

So far we've seen how to construct one-way and two-way payment channels. However, we're still limited by the channel expiry duration, which is determined by the relative locktime on Alice's refund branch. This means that our payment channels can only be open for a limited period of time before the channel has to be closed by Bob.

Next we'll look at how to construct a payment channel that can stay open indefinitely.

#### 6.1 Symmetric commitment states

Payment channels have two branches. So far, the payment channels we've seen have a *spend* branch for Bob and a *refund* branch for Alice. The refund branch is required to prevent Alice's funds from being stranded (Alice's *escape* clause), and needs to have a relative timelock to prevent Alice from prematurely claiming (ie stealing) all the funds in the channel (Bob's *safeguard* clause).

If we contstruct the channel in such a way that instead of having a refund branch, Alice has her own spend branch, then the channel doesn't need an expiry time. Either party can unilaterally close the channel at any time, which means they both have an *escape* route. Alice's spend branch is an exact mirror of Bob's.

These mirrored symmetric commitment transactions require the balance in one commitement transaction to go down when the balance in its mirror commitment transaction goes up. For that reason, we can't have symmetric commitment transactions for a simple channel since balances in simple channels can only go one way. However, using revocable transactions we *can* create symmetric commitment transactions where the balance in one commitment transaction decreases as the balance in the other commitment transaction increases.

Under this scheme Alice signs CTxA and sends it to Bob, and Bob signs CTxB and sends it to Alice.

#### 6.2 Opening a non-expiring payment channel

The anchor transaction for a symmetric payment channel is simply a 2-of-2 multisig transaction. Alice's branch is no longer a refund branch, but a mirror of Bob's spend branch, so the anchor transaction TXO doesn't need a refund branch for Alice with a relative locktime. All CTxs in the channel are the same as for the two-way channel, but are constructed as pairs - CTxA is constructed and signed by Alice and CTxB is constructed and signed by Bob.

Alice needs to be a bit careful in opening the payment channel. If she just pays into a multisig address and Bob disappears, then her funds would be stranded. Therefore, the sequence for opening a symmetric transaction is as follows:

1. Alice constructs an anchor transaction to a 2-of-2 multisig address for Alice and Bob, but she doesn't broadcast or share it.
2. Alice sends the unsigned ATx to Bob, along with her first two revocation hashes h(revA1) and h(revA2).
3. Bob constructs his first commitment transaction CTxB1 using h(revA1) and sends it to Alice, along with his first two revocation hashes h(revB1) and h(revB2).
4. Alice verifies that CTxB1 is correct, then constructs her first commitment transaction CTxA1 using h(revB1) and sends it to Bob.
5. Alice signs and broadcasts the anchor transaction.

![Non-expiring Channel - Opening The Channel](./Non_Expiring_Channel_Messages1.svg)

Commitment state 1 is as follows:

![Non-expiring Channel - Commitment State 1](./Non_Expiring_Channel1.svg)

We're now in a commitment state. Either party is able to close the channel unilaterally and (after the revocation timeout) claim their full balance.

#### 6.3 Updating the balances in the channel

If Alice wants to pay Bob in the channel, she needs to transition the channel to a new commitment state with an increased balance for Bob. She does this as follows:

1. Alice constructs a new commitment transaction CTxA2 using h(revB2) with the new balances, signs it and sends it to Bob.
2. Bob constructs a new commitment transaction CTxB2 using h(revA2) with the new balances, signs it and sends it to Alice. He also sends revB1 to revoke the previous CTxA and hash(revB3) so Alice can construct the next CTxA.
3. Alice sends revA1 to Bob to revoke CtxB1, and hash(revA3) so Bob can construct the next CTxB.

![Non-expiring Channel - Alice pays Bob](./Non_Expiring_Channel_Messages2.svg)

Commitment state 2 is as follows:

![Non-expiring Channel - Commitment State 2](./Non_Expiring_Channel2.svg)

If Bob wants to pay Alice in the channel, the protocol proceeds exactly as above, except that the roles are reversed (ie Bob starts by constructing a new CTxB and sending it to Alice).

#### 6.4 Closing the channel unilaterally

In any commitment state, both parties hold a valid CTx. Either party can close out the channel by broadcasting the CTx, and then claim their funds after waiting for the revocation timeout duration.

This is exactly the same message exchange and protocol as closing a two-way channel unilaterally. The only difference is in a two-way channel only Bob could close the channel unilaterally, whereas in a symmetrical channel either party can close the channel unilaterally.

#### 6.5 Closing the channel co-operatively

If both parties agree that they want to close the channel, they can do so co-operatively and have access to their funds immediately without having to wait for the revocation timeout delay. The message exchange protocol for closing the channel co-operatively is exactly the same as in the earlier two-way channel case. Again, the only difference is that both Alice can propose to close the channel co-operatively since the channel is entirely symmetrical.

As in the two-way channel, if one of the parties constructs and sends an unrevocable closing transaction but the other doesn't broadcast it to the network, the party that has constructed the closing transaction **must** close the channel unilaterally and should not sign any more commitment transactions.

## 7. Acknowledgements and further reading

Many people have contributed ideas to the concept of bitcoin payment channels. The list below aims to identify ideas that have been particularly important in that development. There's no doubt that I'll fail to acknowledge everyone. Please contact [@jonnynewbs](http://www.twitter.com/jonnynewbs) or raise a ticket against [the github repo](http://www.github.com/paychan/bitcoin-payment-channels-taxonomy) if you believe there to be any egregious omissions.

- Simple payment channels were suggested for bitcoin as long ago as 2012. There's [a page on the bitcoin wiki](https://en.bitcoin.it/wiki/Contract#Example_7:_Rapidly-adjusted_.28micro.29payments_to_a_pre-determined_party) which describes a (pre-OP_CHECKSEQUENCEVERIFY) method for creating a payment channel. The page was written by Mike Hearn, with credit for ideas given to Nick Szabó in his paper [Formalizing and Securing Relationships on Public Networks](http://szabo.best.vwh.net/formalize.html).
- Mike Hearn and Matt Corallo implemented payment channels in [bitcoinj](https://bitcoinj.github.io/).
- The ideas of revocable transactions, symmetric commitment transactions and hashed time-locked contracts were introduced by Joseph Poon and Thaddeus Dryja in their paper [The Bitcoin Lightning Network](https://lightning.network/lightning-network-paper.pdf).
- Rusty Russell wrote [an illuminating set of blog posts](http://rusty.ozlabs.org/?p=450) describing the concepts behind the lightning network. Note that many of the specific techniques described in the posts have been obsoleted by the newer OPCODEs and segwit.
- Rusty has also written the first two [BOLTs (Basis of Lightning Technology)](https://github.com/rustyrussell/lightning-rfc/tree/master/bolts) and the [spec for sha-trees](https://github.com/rustyrussell/ccan/blob/master/ccan/crypto/shachain/design.txt).
- The lightning-dev [mailing list](https://lists.linuxfoundation.org/pipermail/lightning-dev/) and [IRC channel](https://botbot.me/freenode/lightning-dev/) are great sources of information.
- There are a number of lightning implementations in development:
    - [lightningd by Blockstream](https://github.com/ElementsProject/lightning/)
    - [lnd by lightningnetwork](https://github.com/LightningNetwork/lnd)
    - [Thunder by Blockchain.info](https://github.com/blockchain/thunder)
    - [Eclair by ACINQ](https://github.com/ACINQ/eclair)
    - [Amiko-pay](https://github.com/cornwarecjp/amiko-pay)