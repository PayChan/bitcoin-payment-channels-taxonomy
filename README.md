# Bitcoin Payment Channels, A Taxonomy

The concept of a payment channel in Bitcoin has been around for several years, but recent changes to the protocol (**OP_CHECKLOCKTIMEVERIFY**, **OP_CHECKSEQUENCEVERIFY**, Segregated Witness) have made much more powerful constructions of payment channels possible.

As far as I'm aware, there haven't been any resources that describe all the different varieties of payment channels possible today. Much of the available material is now obsolete and contains overly complex constructions to achieve what is possible today with far more elegant solutions.

This document is an attempt to describe the various kinds of payment channels that are possible in Bitcoin. It covers:

- the operation of the channel including the opening (*anchor*) transaction, the commitment states and the channel closing conditions
- the order and exchange of transactions for commitment state changes
- the full locking (*scriptPubKey*) and unlocking (*scriptSig*) scripts for all tranactions.

The reader is assumed to have a knowledge of the format of bitcoin transactions and transaction outputs, the concept of pay-to-[witness-]script-hash and the workings of opcodes and the Script language. No prior knowledge of payment channels is assumed.

A few things the article doesn't cover:

- Applications of payment channels
- Historical constructions of payment channels (which have been obsoleted by the new opcodes)
- Anything outside the transaction layer that is required for a fully functioning lightning network (eg routing, message exchange, payment protocol for exchanging payment requests, etc)

This document is necessarily a work in progress. The rate of innovation in this area is extremely rapid, and new varieties of payment channels will most likely continue to be developed. Please direct any feedback to [@jonnynewbs](http://www.twitter.com/jonnynewbs) or raise a ticket against [the github repo](http://www.github.com/paychan/bitcoin-payment-channels-taxonomy).

The article as available at (http://paychan.github.io/bitcoin-payment-channels-taxonomy).

#### TODO

- Run through the script execution for simple payment channels
- Message exchange order for simple payment channels
- Non-Expiring Payment Channels
    - Symmetric Commitment Transactions
    - recharching a payment channel
- Routable Channels (Lightning Network)
    - Hashed Contracts
    - Hashed Time-Lock Contracts