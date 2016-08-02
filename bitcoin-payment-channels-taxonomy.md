# Bitcoin Payment Channels, A Taxonomy

# Table of Contents

-  1 Introduction
-  2 A brief overview
-  3 Diagram style
    -  3.1 Transaction Graphs
        - Transaction types
    -  3.2 Message Exchange Diagrams
    -  3.3 Payment Channel Balance Diagrams
-  4 Simple Payment Channel
    -  4.1 Opening the channel
    -  4.2 Updating balances in the channel
    -  4.3 Redeeming a Commitment Transaction
    -  4.4 Exercising the Refund Branch
-  5 Two-way Channels
    -  5.1 Revocable Transactions
    -  5.2 Using revocable transactions to construct two-way payment channels
-  6 Everlasting Payment Channels
    -  6.1 Symmetric commitment states
    -  6.2 Opening an everlasting payment channel
    -  6.3 Updating the balance
    -  6.4 Re-anchoring the channel
    -  6.5 Closing the channel co-operatively
    -  6.6 Closing the channel unilaterally
-  7 Dual-funded payment channels
-  8 Routable payment channels
    -  8.1 Hashed contracts
    -  8.2 Hashed Time-locked contracts
    -  8.3 Embedding an HTLC into a payment channel

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

A payment channel is a sequence of valid Bitcoin transactions which are constructed, signed and exchanged off-chain by two counterparties. The transactions are not broadcast to the Bitcoin network during the operation of the channel, and only when one or both of the parties are ready to close the channel is a single closing transaction broadcast. This allows the balances between the two parties to be updated many times while only resulting in two transactions on the blockchain: one to open the channel and one to close the channel.

There are many reasons we'd want to do this:

- Each on-chain transaction requires miner transaction fees. Updating the balances within the payment channel and only settling to the blockchain when the channel closes means the parties don't have to pay Bitcoin transaction fees each time the balances are updated.
- Very small (micro) payments can be made within the channel. The Bitcoin network enforces a lower dust limit on transaction outputs, below which the transaction won't be relayed. Payment channel updates can change the balances in the channel by as little as 1 satoshi.
- Bitcoin transactions must be confirmed in at least one block before the recipient can be confident that the funds won't be double-spent. Blocks are mined on average every 10 minutes, so payments can take many minutes to be confirmed. In a payment channel, the funds are locked into the channel so the balances of both parties can be instantly updated with new commitment transactions within the channel.
- Bitcoin blocks have a size limit, which places a hard limit on the rate that the network can process transactions (currently around 7 transactions per second). Since payment channels only consume two on-chain transactions, using payment channels where there are lots of balance updates between two parties would allow the Bitcoin network to scale to many more transactions per second.

Payment channels exist as a sequence of *commitment states*. For a channel to be in a commitment state:

1. Both parties agree on their balances within the channel. This is the *consensus* property.
2. Either party can close out the channel unilaterally by broadcasting a transaction to claim their full balance (although they may need to wait for a timelock before doing so). This is the *escape* property.
3. As long as the parties monitor the blockchain and act correctly, neither can be denied their full balance. This is the *safeguard* property.

To update the balances in the channel, the channel goes through a *commitment state change* or *commitment transition* and enters a new commitment state with updated balances. A payment channel will always be in a commitment state or a commitment transition between two commitment states.

The *consensus* property guarantees that the channel remains synchronized between both parties and that the channel never enters a state where the parties' balances are confused. The *exit* property guarantees that at all times during channel's existence, both parties have an escape route to claim their balance. The *safeguard* property guarantees that by entering into a payment channel, neither party can be denied their full balance (so long as they act correctly).

Taken together, these properties ensure the trustless nature of the payment channel and mean that neither party takes on risk by entering into the channel.

## 3. Diagram style

#### 3.1 Transaction Graphs

Bitcoin transactions are collections of Transaction Inputs (TXIs) and Transaction Outputs (TXOs). The simplest Bitcoin transaction consists of a single TXI and a single TXO. The value of the TXO is equal to the value of the TXI (less the transaction fee):

![Simple Transaction](./Basic_Transaction1.svg)

A TXI is simply an unspent TXO (a UTXO) from a previous transaction, and the TXO from this transaction will go on to be a TXI in a future transaction. We can draw a chain of transactions as follows:

![Transaction Chain](./Basic_Transaction2.svg)

We're not interested in the transaction that funded our channel, just that it output a TXO which we use as our TXI. We also don't know how the TXO from our transaction will be spent until it's included as a TXI in a future transaction. Our standard transaction ends up looking like this:

![Bare Transaction](./Basic_Transaction3.svg)

Of course, most Transactions don't have just a single output. Here's an example transaction with two TXOs:

![Transaction with multiple TXOs](./Basic_Transaction4.svg)

Finally, a single TXO can be spent in many ways. We can illustrate that with branching from the TXO:

![TXO with branching child transactions](./Basic_Transaction5.svg)

This is slightly arbitrary, since the TXO could be spent in an infinite number of ways. However, it is instructive to see the different ways that we're expecting the TXO to be spent. In the following payment channels we'll be constructing many commitment transaction, each overriding the previous commitment transactions, so this notation will be useful.

##### Transaction types

If a transaction has been broadcast to the Bitcoin network, we'll colour it green:

![Broadcast Transaction](./Transaction_Types1.svg)

We're not going to consider blockchain confirmations in this document. As with all Bitcoin transactions, users should wait for their transaction to be confirmed before building on top of it, but whether they actually do, and how many confirmations they wait for is an implementation decision. For the rest of the article we'll assume that all broadcast transactions are confirmed and will not be double-spent or invalidated.

If a transaction has been constructed and one party has a valid witness for the transaction, then we colour it yellow:

![Unbroadcast Transaction](./Transaction_Types2.svg)

This transaction can be broadcast to the network (potentially after a locktime or relative locktime) by at least one party to close out the channel.

There may be several unbroadcast transactions using the same TXOs. We'll keep the most recent one coloured yellow, but the older transactions will be a lighter yellow to show that we're not expecting them to be used:

![Overwritten Transaction](./Transaction_Types3.svg)

Once a transaction has a valid witness, that witness is valid forever. However, the transaction can be invalidated by creating and broadcasting a new transaction which double-spends the TXIs for the old transaction. We'll colour transactions that have been invalidated in this way red:

![Invalidated Transaction](./Transaction_Types4.svg)

#### 3.2 Message Exchange Diagrams

Message exchange is depicted using standard ladder diagrams. As well as showing message exchange between the participants in a channel, the ladder diagram can also show one of the participants broadcasting a message to the blockchain:

![Ladder Diagram](./Ladder_Diagram.svg)

In practice, when we show Alice broadcasting a 'signed transaction' to Bob within the channel, Alice may only be broadcasting a signature. As long as Bob knows what the transaction format should be, there's no need for Alice to send the full transaction to Bob.

#### 3.3 Payment Channel Balance Diagrams

Payment Channel Balance Diagrams are used to show the balance within a payment channel.

Alice and Bob start with their own funds of Bitcoin:

![Payment Channel Diagram 1](./Payment_Channel_Diagram1.svg)

Alice funds a new payment channel with 20 BTC. Those funds are hers, but are temporarily locked in the payment channel. The balance of funds within the payment channel can move left and right as the parties create and exchange new commitment transactions:

![Payment Channel Diagram 2](./Payment_Channel_Diagram2.svg)

Alice then pays 10 BTC to Bob within the channel. The channel now contains a balance of 10 BTC for Alice and 10 BTC for Bob:

![Payment Channel Diagram 3](./Payment_Channel_Diagram3.svg)

Alice continues to pay Bob in the channel until her balance is 5 BTC and his balance is 15 BTC:

![Payment Channel Diagram 4](./Payment_Channel_Diagram4.svg)

Finally, Alice closes out the channel, freeing up 5 BTC for herself and 15 BTC for Bob:

![Payment Channel Diagram 5](./Payment_Channel_Diagram5.svg)

## 4. Simple Payment Channel

This is the simplest form of Payment Channel. It is **One-way**, **Simplex** or **Unidirectional**, which means that funds can only flow in one direction from payer to recipient. It is also **Fixed-duration**, which means that the payment channel has to be closed out with a *closing transaction* before a fixed expiry time. If the payer wishes to continue paying the recipient through a channel after this time, she must open a fresh payment channel with a new *anchor transaction*.

As is tradition, Alice is the payer and Bob is the recipient. Alice wishes to pay Bob in increments of 0.01 BTC up to a maximum of 1 BTC.

#### 4.1 Opening the channel

To open the channel, Alice constructs and signs an anchor transaction and then broadcasts it to the Bitcoin network. The anchor transaction has a single TXO which can be spent either:

1. with both Bob and Alice's signatures. This is the *spend* branch; or
2. with Alice's signature after a channel expiration duration (eg 24 hours). This is the *refund* branch.

![Simple Channel - Funded](./Simple_Channel1.svg)

Alice needs a refund branch to protect her funds from being stranded in the channel. This is required for the fundamental *escape* property of channels - both parties must have an escape route to reclaim their funds at all times in the channel's existence. If the anchor transaction didn't have a refund branch and was just a 2-of-2 multisig, then Alice wouldn't have an escape route.

Without an escape route Alice's funds could become stranded or held to ransom inside the channel. If Bob stops responding to Alice's messages (either inadvertently or maliciously), Alice would have no way to get her funds back. A malicious Bob might hold Alice's funds to ransom and only agree to unlock the multisig TXO in return for a ransom fee.

The locking script is as follows:

```
OP_IF
  <Bob's public key> OP_CHECKSIGVERIFY # Spend branch - requires both signatures
OP_ELSE
  <channel expiry duration> OP_CHECKSEQUENCEVERIFY OP_DROP # Refund branch - after the channel expiry duration only Alice's signature is required
OP_ENDIF
<Alice's public key> OP_CHECKSIG # Both branches require Alice's signature
```

To finish opening the channel, Alice constructs the first *commitment transaction* (*CTx1*) and sends it directly to Bob. This transaction has the anchor transaction TXO as its only TXI, and produces two TXOs:

- TXO1 is for 0.01 BTC and can be spent with Bob's signature
- TXO2 is for 0.99 BTC and can be spent with Alice's signature

Both of those TXOs can be standard P2PKHs.

We're now in commitment state 1:

![Simple Channel - first commitment](./Simple_Channel2.svg)

Let's check the commitment state properties:

1. Both parties agree that Alice's balance is 0.99 BTC and Bob's balance is 0.01 BTC.
2. Both parties have an escape route: Alice can escape by broadcasting her refund transaction and claiming 1 BTC (after the channel expiry duration), and Bob can escape by signing and brodcasting CTx1 and claiming his 0.01 BTC.
3. Both parties are guaranteed their full balance: Alice gets at least 0.99 BTC in both escape paths. Bob can guarantee he'll get 0.01 BTC by broadcasting CTx1 before Alice broadcasts the refund transaction. To make sure this happens, he just needs to broadcast the CTx well before the channel expiry duration.

#### 4.2 Updating balances in the channel

Let's assume that Bob hasn't closed out the channel and Alice wants to pay a further 0.01 BTC to Bob. She constructs and signs a second commitment transaction CTx2 and sends it to Bob. This second transaction has exactly the same anchor transaction TXO as its TXI, but now produces the following TXOs:

- TXO1 is for 0.02 BTC and can be spent with Bob's signature
- TXO2 is for 0.98 BTC and can be spent with Alice's signature

Commitment state 2 looks like this:

![Simple Channel - second commitment](./Simple_Channel3.svg)

In effect, this is a double-spend of the anchor transaction TXO, so only one of those transactions can be included in the blockchain.

Let's check the commitment state properties again:

1. Both parties now agree that Alice's balance is 0.98 BTC and Bob's balance is 0.02 BTC.
2. Both parties have an escape route: Alice can escape by broadcasting her refund transaction and claiming 1 BTC (after the channel expiry duration), and Bob can escape by signing and brodcasting one of the CTxs. He'll almost certainly prefer the transaction which gives him 0.02 BTC, so he can throw away CTx1 as soon as he receives CTx2.
3. Both parties are guaranteed their full balance: Alice gets at least 0.98 BTC in both escape paths. Bob can guarantee he'll get 0.02 BTC by broadcasting CTx2 (and not broadcasting CTx1!), and making sure CTx2 is broadcast before Alice broadcasts the refund transaction. Alice doesn't have Bob's signature for any of the CTxs, so can't broadcast an old CTx that pays Bob a lower balance.

Alice can continue paying Bob through the payment channel in this fashion. Each time she wants to pay another 0.01 BTC to Bob, she constructs a new CTx from the same anchor transaction output and sends it to Bob.

![Simple Channel - multiple commitments](./Simple_Channel4.svg)

This continues until one of the following happens:

1. Bob wishes to close out the channel, and so signs and broadcasts the most recent CTx.
2. the funds in the channel are exhausted and the most recent CTx sends 1 BTC to Bob and 0 to Alice. At this point, Bob can just sign and broadcast that CTx and collect the 1 BTC.
3. The payment channel expiry duration is reached. At this point Alice can reclaim all of the funds in the channel. Bob should never let this happen, so should sign and broadcast the most recent CTx well before the expiry duration.

Here's a diagram of Bob closing the channel after 4 commitment transactions:

![Simple Channel - ladder diagram](./Simple_Channel5.svg)

In this example:

1. Alice broadcasts the Anchor transaction to the network
2. Alice then constructs and signs a sequence of four CTxs, which she sends to Bob
3. Bob closes the channel by signing and broadcasting the most recent transaction CTx4

The advantage of this style of payment channel is that it is extremely simple. The anchor TXO locking script is essentially either a 2-of-2 multisig in the spend branch, or a P2PKH with relative timelock in the refund branch. Commitment transitions are achieved simply by Alice constructing and signing new CTXs and sending them to Bob.

The channel is also almost entirely passive from Bob's point of view. He simply needs to keep hold of the most recent CTx, and then sign and broadcast it when he's ready to close the channel. Simple channels should be very straightforward for wallets and applications to implement.

#### 4.3 Redeeming a Commitment Transaction

To redeem a CTx, Bob broadcasts the transaction with the following unlocking script:

```
<Alice's sig> <Bob's sig> 1
```

#### 4.4 Exercising the Refund Branch

In mainline payment channel operation, the Refund Branch should never be exercised since doing so denies Bob his full balance. However, if Bob stops responding, Alice can reclaim the full funds in the channel by constructing and broadcasting a refund transaction after the channel expiry duration. This refund transaction takes the anchor transaction TXO as its TXI. The unlocking script for the transation is:

```
<Alice's sig> 0
```

## 5. Two-way Channels

One of the most obvious limitations of the simple payment channel is that it is one-way. Bob's balance in the channel can only ever increase, and Alice's balance can only ever decrease. This is because every commitment transaction that Alice signs and sends to Bob is valid forever (at least until one of them is broadcast and confirmed). Even if Alice constructs and signs a new transaction with a smaller balance for Bob, Bob will always be able to broadcast the commitment transaction which assigns him the greatest balance. Alice has no way to stop Bob from doing this, and no way to invalidate the old commitment trasactions.

However, there is a trick that allows Alice to ensure that Bob can't use a previous commitment transaction to claim an old balance. This trick uses hash pre-images and timelocks to construct a *revocable* transaction. That's what we'll look at next.

#### 5.1 Revocable Transactions

The trick to creating revocable transactions is to construct one of the TXOs such that it is either encumbered by:

- Bob's signature and a relative timelock (Bob's *spend branch*); or
- Alice's signature and a secret revocation hash provided by Bob (Alice's *revocation branch*).

To revoke the transaction, Bob reveals the pre-image of his secret revocation hash to Alice. If Bob ever broadcasts the revoked transaction, then Alice will have the chance to spend before him (since his spend branch is encumbered by a timelock and Alice's isn't). Bob is effectively blocked from being able to use the revoked transaction.

We'll call this type of TXO a *revocable TXO* or *rTXO* for short. The secret revocation hash is *h(rev)* and its pre-image is *rev*.

![Revocable Transaction](./Revocable_Transaction1.svg)

Transactions will normally have multiple TXOs. Within a payment channel, there will be one TXO for Alice's balance and an rTXO for Bob's balance (which can be spent by Alice when the rTXO is revoked):

![Full Revocable Transaction](./Revocable_Transaction2.svg)

Once Bob has revealed the revocation secret, he's no longer able to broadcast the revocable transaction since Alice would be able to collect her spend TXO as well as the rTXO. The transaction has been revoked:

![Revoked Transcation](./Revocable_Transaction3.svg)

We're going to use rTXOs *a lot* for more advanced channels, so it makes sense to have a special notation for them:

![Revocable Transaction - Notation](./Revocable_Transaction4.svg)

Conceptually, the combined up/down facing arrow indicates that the rTXO can be reversed.

The locking script for a revocable transaction is:

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

For Bob to spend the TXO, he needs to wait for the revocation timeout duration and then provide the following unlocking script:

```
<Bob's sig> 1
```

If Bob broadcasts a revoked transaction, Alice can claim the revocation TXO by providing the following unlocking script:

```
<Alice's sig> <rev> 0
```

#### 5.2 Using revocable transactions to construct two-way payment channels

To create a two-way payment channel, Alice constructs the anchor transaction exactly as before (where the TXO can be spent either by a 2-of-2 multi-sig or by just herself after the channel expiry duration). The difference from the one-way payment channel is in the construction of the commitment transactions: instead of including a standard P2PKH for Bob's TXO, she uses a rTXO with a revocation hash provided by Bob.

Commitment state 1 is as follows:

![Two-way Channel - First commitment](./two-way-channel1.svg)

If Alice wants to increase Bob's balance in the channel to 0.02 BTC, she continues in exactly the same way as for the one-way payment channel. She constructs a new commitment transaction which sends 0.98 BTC to herself and 0.02 BTC to Bob. Again, the only difference is that the TXO for Bob in the CTx is an rTXO.

Commitment state 2 is:

![Two-way Channel - Second commitment](./two-way-channel2.svg)

You'll notice that the rTXO in commitment transaction uses the same revocation secret as for CTx1. That's because Alice only needs to revoke an old CTx if her balance in the new CTx has increased. She doesn't need to revoke CTx1 (which gives her 0.99 BTC) since she'd be perfectly happy for Bob to broadcast CTx1 instead of CTx2 (which gives her 0.98 BTC).

This is an important point which will be relied upon later. Revocation secrets only need to be revealed when the balance in a rTXO decreases. If the amount encumbered with the revocation secret increases from one commitment state to the next, the same revocation secret can be used.

If Bob now wants to pay Alice 0.01 BTC and reduce his balance back to 0.01 BTC, he provides her with a new revocation hash (*hash(rev2)*), which she uses to construct CTx3. Once she's sent CTx3 to Bob, Bob sends her rev1, which revokes CTx1 and CTx2:

![Two-way Channel - Third commitment](./two-way-channel3.svg)

Finally, Alice wants to increase Bob's balance to 0.02 BTC again. She constructs CTx4 using h(rev2):

![Two-way Channel - Fourth commitment](./two-way-channel4.svg)

Bob can close the channel as soon as the rTXO timeout duration has elapsed by signing and broadcasting the most recent CTx.

## 6. Everlasting Payment Channels

So far, we've seen how to construct one-way and two-way payment channels. However, we're still limited by the channel expiry duration, which is determined by the relative locktime on Alice's refund branch. This means that our payment channels can only be open for a certain period of time before the channel has to be closed by Bob.

Next we'll look at how to construct a payment channel that can stay open indefinitely.

#### 6.1 Symmetric commitment states

Payment channels have two branches. So far, the payment channels we've seen have a *spend* branch for Bob and a *refund* branch for Alice. The refund branch is required to prevent Alice's funds from being stranded (Alice's *escape* clause), and needs to have a relative timelock to prevent Alice from stealing all the funds in the channel (Bob's *safeguard* clause).

With revocable transactions, we have a new method of preventing funds from being stranded inside the transaction. Instead of Alice's branch being a refund transaction, we can create a mirror image of Bob's branch, with the revocable transaction ensuring that the funds don't get stranded. Either party can close out the channel with their most recent commitment transaction, so there's no longer any need for a refund branch.

#### 6.2 Opening an everlasting payment channel

The anchor transaction for a symmetric payment channel is simply a 2-of-2 multisig transaction. Alice's branch is no longer a refund branch, but a mirror of Bob's spend branch, so the anchor transaction TXO doesn't need a relative locktime refund branch for Alice.

We do need to be a bit careful in opening the payment channel. If Alice just pays into a multisig address, then her funds could be stranded if Bob disappears. Therefore, the sequence for opening a symmetric transaction is as follows:

1. Alice constructs an anchor transaction to a 2-of-2 multisig address for Alice and Bob, but she doesn't broadcast or share it.
2. Alice sends the txid (the hash of the transaction) to Bob, along with her first revocation hash h(revA1).
3. Bob constructs his first commitment transaction CTxB1 using h(revA1) and sends it to Alice, along with his first revocation hash h(revB1).
4. Alice constructs her first commitment transaction CTxA1 using h(revB1) and sends it to Bob.
5. Alice signs and broadcasts the anchor transaction.

![Everlasting Channel - Commitment State 1](./Everlasting_Channel1.svg)

We're now in a commitment state. Either party is able to close the channel and claim their full balance (after the revocation timeout).

#### 6.3 Updating the balance

If Alice wants to pay Bob in the channel, she needs to transition the channel to a new commitment state with an increased balance for Bob. She does this as follows:

1. Alice constructs a new commitment transaction CTxA2 using the new balances and h(revB1) and sends it to Bob, along with a new revocation hash h(revA2).
2. Bob constructs a new commitment transaction CTxB2 using the new balances and h(revA2) and sends it to Alice
3. Alice sends revA1 to Bob, which revokes CtxB1.

![Everlasting Channel - Commitment State 2](./Everlasting_Channel2.svg)

If Bob wants to pay Alice in the channel, the protocol proceeds exactly as above, except that the roles are reversed (ie Bob starts by sending a new revocation hash to Alice).

The TXOs in a symmetric transaction are exactly the same is in the two-way transaction described earlier. The only difference is that the anchor transaction is a simple 2-of-2 multisig, and the CTxs are constructed as a symmetric pair and exchanged using protocol described above.

#### 6.4 Re-anchoring the channel

Since there is no refund transaction in this type of channel, it's possible for the parties to co-operate to add fresh funds to the payment channel or withdraw funds from the payment channel without closing it.

If one party runs out of funds in the channel, they may want to recharge the payment channel with additional funds (one blockchain transaction) rather than close the channel and open a new channel (two transactions).

Let's assume Alice wants to add funds to the channel. The process is as follows:

1. Alice constructs and signs a second anchor transaction ATx2 to the 2-of-2 multisig address for Alice and Bob, but she doesn't broadcast or share it. The TXIs for ATx2 are the TXO from ATx1 and a new TXI from Alice for the recharge amount.
2. Alice sends the txid (the hash of the transaction) to Bob, along with a new revocation hash h(revA2-1).
3. Bob constructs a new commitment transaction CTxB2-1 as follows:
    - The TXI is the TXO from ATx2
    - Bob's balance is the same is it was in the previous CTx
    - Alice's balance is her balance from the previous CTx plus the additional funds she's just paid into the channel with ATx2
    - the revocation hash is h(revA2-1)
4. Bob sends CTxB2-1 to Alice, along with a new revocation hash h(revB2-1).
5. Alice constructs her first commitment transaction CTxA2-1. This is the mirror of Bob's commitment transaction CTxB2-1.
6. Alice broadcasts ATx2 to the Bitcoin network
7. Alice sends the revocation pre-image from the previous CTxA to Bob.
8. Bob sends the revocation pre-image from the previous CTxB to Alice.

Withdrawals can be made in a similar way. If Alice wants to withdraw funds from the channel:

1. Alice constructs and signs a second anchor transaction ATx2, but she doesn't broadcast or share it. The TXI for ATx2 is the TXO from ATx1 and a new TXI from Alice for the recharge amount. There are two TXOs: one which pays into the 2-of-2 multisig (for the channel), and one which pays to Alice.
2. Alice sends the txid (the hash of the transaction) to Bob, along with a new revocation hash h(revA2-1).
3. Bob constructs a new commitment transaction CTxB2-1 as follows:
    - The TXI is the first TXO from ATx2
    - Bob's balance is the same is it was in the previous CTx
    - Alice's balance is her balance from the previous CTx minus the funds she's just withdrawn from ATx1
    - the revocation hash is h(revA2-1)
4. Bob sends CTxB2-1 to Alice, along with a new revocation hash h(revB2-1).
5. Alice constructs her first commitment transaction CTxA2-1. This is the mirror of Bob's commitment transaction CTxB2-1.
6. Alice broadcasts ATx2 to the Bitcoin network
7. Alice sends the revocation pre-image from the previous CTxA to Bob.
8. Bob sends the revocation pre-image from the previous CTxB to Alice.

Paying into the payment channel (from one side) and withdrawing from the payment channel (to the other side) is a quick way to rebalance a payment channel that has become unbalanced.

![Everlasting Channel - Re-anchoring the channel](./Everlasting_Channel3.svg)

#### 6.5 Closing the channel co-operatively

If both parties agree that they want to close the channel, they can close it immediately without having to wait for the revocation timeout delay. If Alice wants to close the channel co-operatively, she constructs a *closing transaction* as follows:

1. the TXI is the TXO from the anchor transactions
2. there are two TXOs:
    - a P2PKH to Alice for her balance (from the current commitment state)
    - a P2PKH to Bob for his balance (from the current commitment state)

she signs the transaction and sends it to Bob, who also signs it and broadcasts it to the Bitcoin network:

![Symmetric Channel - Closing the channel co-operatively](./Everlasting_Channel4.svg)

Bob can close the channel co-operatively in exactly the same way, since the channel is entirely symmetrical.

If Alice constructs and sends the closing transaction to close the channel co-operatively but Bob doesn't broadcast it to the network, Alice **must** close the channel unilaterally and should not sign any more commitment transactions. This is because Alice cannot revoke the closing transaction, so Bob might keep hold of it and broadcast it after his balance has decreased.

#### 6.6 Closing the channel unilaterally

In any commitment state, both parties hold a valid CTx. Either party can close out the channel by broadcasting the CTx after waiting for the revocation timeout duration.

![Symmetric Channel - Closing the channel unilaterally](./Everlasting_Channel5.svg)

## 7. Dual-funded payment channels

So far, all of the payment channels that we've seen have been wholly funded by Alice. We now have a method of opening a payment channel that's funded by both parties:

1. Alice opens the payment channel as normal with her funds.
2. Alice and Bob co-operate to re-anchor the payment channel using Bob's funds as an additional input.

![Dual-funded Payment Channel](./Dual_Funded_Channel.svg)

## 8. Routable payment channels

Payment channels exist between two parties. With the payment channels we've looked at so far, if Alice wants to pay or be paid by a new counterparty, she would need to open a new payment channel with that party. That limits the scalability of payment channels since for every new channel we need to broadcast two transactions to the blockchain. In the worst case (where the parties only want to execute a single payment) this is actually *worse* than simply broadcasting a single transaction to the blockchain.

However, if we could somehow link payment channels together and route payments through them, we wouldn't need to open a new channel for each new recipient. For example, imagine:

- Alice and Bob have an open payment channel
- Bob and Carol have an open payment channel
- Alice does not have a payment channel with Carol

If Alice wants to pay Carol, she would route a payment through Alice ---> Bob ---> Carol instead of having to open a new channel to Carol.

The naive way to do this would be for Alice to pay Bob and then ask Bob to pay Carol. That requires Alice to trust Bob with her payment and would allow a malicious Bob to steal Alice's money instead of delivering it to Carol.

However, using a trick similar to the revocation hashes used earlier, Alice can pay Carol through Bob without having any trust in Bob. The trick uses *Hashed Time-lock Contracts* (*HTLCs*) to link the payment channels together. Payment channels that are linked into a network in this way are called a *lightning network*.

The next two sections introduce the concepts of hashed contracts and hashed time-locked contracts independently from payment channels. Finally, we'll see how to embed those HTLCs within the everlasting payment channel described above to create a network of routable payment channels.

#### 8.1 Hashed contracts

Imagine Alice wants to pay Carol, but Alice only has AliceCoins and Carol only accepts CarolCoins. Luckily, they both know Bob, and Bob is happy to buy and sell in AliceCoins *or* CarolCoins. Alice wants to find a way to pay Bob in AliceCoins and make sure that he pays onwards to Carol.

Alice doesn't trust Bob enough to pay him and know that he'll pay onwards to Carol, and Bob doesn't trust Alice enough to pay Carol and know that he'll be able to collect payment from Alice.

Alice needs a contract that says "*I promise to pay Bob if he proves he's paying Alice*". Simultaneously, Bob needs a contract that says "*I promise to pay Carol if Alice proves she's paying me*".

We can construct those contracts using a very similar trick to the revocation secrets used in the revocable transactions above. Here's how it works:

1. Carol generates a contract secret *con* and sends the hash of that secret *h(con)* to Alice through a secure channel.
2. Alice constructs a payment to Bob encumbered with *h(con)* and broadcasts it to the network
3. Bob constructs a payment to Carol *encumbered with the same secret contract hash h(con)* and broadcasts it to the network
4. Carol redeems Bob's payment by revealing *con*
5. Bob can redeem Alice's payment by revealing *con*

![Hash Contracts - 2 hops](./hash-contract1.svg)

There's no reason that this method can't be extended further. Say Alice wants to pay Erica. Alice knows Bob, who knows Carol, who knows David, who knows Erica. Alice can route a payment to Erica through Bob, Carol and David:

![Hash Contracts - 4 hops](./hash-contract2.svg)

The locking script for Alice's hashed contract to Bob is:

```
OP_HASH160 <h(con)> OP_EQUALVERIFY OP_DROP #Check that Bob has the contract hash pre-image
<Bob's public key> OP_CHECKSIG #Check Bob's signature
```

Bob's unlocking script is simply:

```
<Bob's sig> <con>
```

Bob can issue exactly the same contract to Carol, just swapping his public key in the locking script for Carol's.

#### 8.2 Hashed Time-locked contracts

If you've been reading carefully, you'll notice that there's a problem with the hashed contract protocol. It's exactly the same problem as when Alice opened her first simple payment channel with Bob. The problem is that as soon as Alice broadcast her first transaction at stage (1), if Bob stops responding (either inavertently or maliciously), her funds are stranded in a hashed contract and she can't get them back. Even worse, if Carol stops responding after Bob broadcasts his transaction in step (2), his funds are stranded in a hashed contract *and* Alice's funds are stranded in a hashed contract. Obviously, the longer the chain of hashed contracts, the more risk there is that one of the parties in the chain will disappear and strand everyone's funds.

The solution to this is basically the same as the refund branch in the simple payment channel. Each hashed contract contains its own refund branch that means the contract payer can reclaim the funds if the recipient doesn't redeem the contract using the *con* within a certain timelock duration. A hashed contract with a timelocked refund branch is called a Hashed Time-locked Contract or HTLC.

There are a couple of subtleties here:

- because there's a chain of hashed contracts, each timelock should be slightly lower than the timelock in the previous contract. Bob doesn't want to end up in a situation where Carol has redeemed his HTLC, but Alice's HTLC to him has timed out and Alice has claimed it back. For that reason, each party in the HTLC decreases the contract timelock in his onward contract so he has plenty of time to redeem his inward contract. Alice's contract to Bob would have a four day timelock, Bob's contract to Carol would have a three day timelock, Carol's contract to David would have a two day timelock, and so on.
- The timelocks in these contracts are *absolute* timelocks rather than *relative* timelocks. That's because the timelock should be relative to the timelock in the previous HTLC in the chain, not relative to when this HTLC happens to hit the blockchain.

The chain of HTLCs now looks like this:

![HTLCs - 4 hops](./htlc1.svg)

The locking script for Alice's HTLC to Bob is:

```
OP_IF # Spend branch - Bob can spend the HTLC if he has the contract pre-image
    OP_HASH160 <h(con)> OP_EQUALVERIFY OP_DROP
    <Bob's public key>
OP_ELSE # Renfund branch - Alice can spend if the HTLC has timed out
    <HTLC timelock> OP_CHECKLOCKTIMEVERIFY OP_DROP
    <Alice's public key>
OP_ENDIF
OP_CHECKSIG
```

Bob's spend unlocking script is:

```
<Bob's sig> <con> 1
```

Alice's refund unlocking script is:

```
<Alice's sig> 0
```

#### 8.3 Embedding an HTLC into a payment channel

We can embed the HTLC in a payment channel by adding the HTLC as a TXO to the commitment script. To add a routed payment, we transition from a commitment state without the HTLC to a new commitment state where the commitment transactions contain an HTLC:

![Lightning Channel](./Lightning_Channel1.svg)

There's one very small detail that needs to change in the HTLC before we can do that. The parties in the channel need to be able to revoke HTLCs in previous commitment states, just like they're able to revoke the rTXOs. That's so that when they revoke a previous transaction state, all the TXOs from it are revoked.

The HTLC is redeemable in three ways:

- by the recipient by using the contract pre-image
- by the sender if the HTLC timelock has expired
- (if one of the parties in the channel tried to cheat and redeem an old commitment state) by the cheated party by using the revocation pre-image

The locking script for the HTLC is slightly different in the two branches. If Alice is adding an HTLC to Bob, the locking script in her branch is:

```
OP_HASH160 OP_DUP
   h(rev) OP_EQUAL OP_SWAP​
   h(con) OP_EQUAL
OP_ADD
OP_IF
   <Bob's public key>
OP_ELSE
   <HTLC timelock> OP_CHECKTIMELOCKVERIFY OP_DROP
   <Alice's public key>
​OP_ENDIF​
OP_CHECKSIG
```

The unlocking scripts are:

1. Bob spends by using the contract pre-image:

```
<Bob's sig> <con>
```

2. Alice spends after the HTLC timelock:

```
<Alice's sig> 0
```

3. Alice tries to cheat by broadcasting this old commitment state, so Bob claims the revocation branch and HTLCs using the revocation pre-image:

```
<Bob's sig> <rev>
```

The locking script in Bob's branch is:

```
OP_HASH160 OP_DUP
<h(con)> OP_EQUAL
OP_IF
    OP_DROP
    <revocation timeout duration> OP_CHECKSEQUENCEVERIFY OP_DROP
    <Bob's public key>
OP_ELSE
    <h(rev)> OP_EQUAL
    OP_IF OP_ELSE
        <HTLC timelock> OP_CHECKTIMELOCKVERIFY OP_DROP
    OP_ENDIF
    <Alice's public key>
OP_ENDIF
OP_CHECKSIG
```

The unlocking scripts are:

1. Bob spends by using the contract pre-image after the revocation timelock

```
<Bob's sig> <con>
```

2. Alice spends after the HTLC timelock:

```
<Alice's sig> 0
```

3. Bob tries to cheat by broadcasting this old commitment state, so Alice claims the revocation branch and HTLCs using the revocation pre-image:

```
<Alice's sig> <rev>
```