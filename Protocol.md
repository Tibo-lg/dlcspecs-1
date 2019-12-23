# DLC Protocol

This document describes a protocol for two parties to set up a Discreet Log Contract (DLC).

## Preliminaries

The following protocol assumes that both parties have already agreed on the contract terms (the possible outputs of the contract).
Negotiation of the contract, or decisions on the underlying asset are thus not part of the exchanged information.

## Description

The two parties exchange a set of three messages, and terminates when the non-initiating party broadcasts the Fund Transaction to the Bitcoin network.
The following figure gives a brief overview of the protocol:

    +-------+                    +-------+
    |       |                    |       |
    |       |---- (1) offer  --->|       |
    |       |                    |       |
    |       |<--- (2) accept ----|       |
    |   A   |                    |   B   |
    |       |---- (3) sign   --->|       |
    |       |                    |       |
    |       |                    |  (4) broadcast fund-tx
    |       |                    |       |
    +-------+                    +-------+

### Offer

The initiating party starts the protocol by sending an `offer` message to the other party.
This message includes the following information:
1. Contract information (an array of triples defined below)
1. Oracle information (optional)
1. A's fund amount (satoshi)
1. A's public key
1. A's inputs
1. A's change outputs (optional)
1. Estimatesmartfee (satoshi/kbyte)
1. CET CSV delay (optional)
1. Refund locktime

#### Contract information
Contract information consists of a list of triples to be used to create CETs.
The triple consists of:
`message` - `a_output` (satoshi) - `b_output` (satoshi)
where `message` is the data on which the oracle will be providing a (set of) signatures, and `a_output` and `b_output` represent the amount that A and B will receive for this outcome respectively.
The interpretation of the message semantics is left to the application.

#### Oracle information
The oracle R point to use to build the CETs as well as meta information to identify the oracle to be used in the contract:
`R_point` - `meta data`
The meta data can either be human readable for a user to be able to interpret it, or machine readable so that an application can interpret it and present it to the user.
If both parties already have this information, transmission is unnecessary.

#### A's fund amount (satoshi)
How much A inputs into the contract.

#### A's public key
The public key to be used in the Fund transaction, refund transaction and CETs.

#### A's inputs
The set of UTXOs to be used as input to the fund transaction.

#### A's change output (optional)
If the sum of A's inputs is greater than the fund amount plus the fee for the fund transaction, A can specify a change output to be added to the fund transaction.

#### Estimatesmartfee (satoshi/kbyte)
The fee rate to be used when computing fees for the different transactions.

#### CET CSV Delay (optional)
The number to use as input to the CSV in the CET transactions.
This is optional, and a default of 144 (roughly one day) can be use which should give enough time for a party to broadcast its closing transaction and get it included in the blockchain in most circumstances.

#### Refund locktime
The locktime to be used for the refund transaction.
It should be set at a later date than the maturity date of the contract.

### Accept
After receiving the `offer` message from A, and after [validating](#offer-validation) the provided information, B creates and sends back an `accept` message containing:
1. B's fund amount (satoshi)
1. B's public key
1. B's inputs
1. B's change output (optional)
1. CET signatures
1. Refund signature

#### B's fund amount (satoshi)
How much B inputs into the contract.

#### B's public key
The public key to be used in the Fund transaction, refund transaction and CETs.

#### B's inputs
The set of UTXOs to be used as input to the fund transaction.

#### B's change output (optional)
If the sum of B's inputs is greater than the fund amount plus the fee for the fund transaction, B can specify a change output to be added to the fund transaction.

#### CET signatures
A set of signatures from B, one for each CET input.

#### Refund signature
A signature from B for the refund transaction input.

### Sign

After receiving and [validating](#sign-validation) the content of the `offer` message, A replies with a `sign` message containing:
1. Fund transaction signatures
1. CET signatures
1. Refund signature

#### Fund transaction signatures
A set of signatures from A, one (in case of P2WPKH) or several (in case of multisig P2WSH) for each of A's input to the fund transaction.

#### CET signatures
A set of signatures from A, one for each CET input.

#### Refund signature
A signature from A for the refund transaction input.

### Broadcast
Upon receiving the `sign` message from A, B [validates](#sign-validation) its content, signs the fund transaction and broadcasts it to the Bitcoin network.

## Validation

This section describes how each party needs to validate the messages received.
We distinguish between user validation, performed by the user (through a user interface), and machine validation, that can be done without user interaction.

### Offer Validation

#### User validation
* Contract Information: confirm that the contract information matches what was agreed upon.
* Oracle information: confirm that the oracle specified in the message matches what was agreed upon.
* A fund amount: confirm that the input amount matches what was agreed upon.
* Estimatesmartfee: confirm that it corresponds to what was agreed upon.
* CET CSV delay: confirm that it corresponds to what was agreed upon.
* Refund locktime: confirm that it corresponds to what was agreed upon.

#### Machine validation

* A fund amount: given this and B fund amount, need to validate that the amount incorporates the required fees for the CET or refund transaction as well as the closing transaction, and that the total input amount is greater than the payout amounts plus the fees to be paid.
* A public key: validate that the public key is well formed.
* Alice inputs: validate that the referenced outputs are in fact unspent and are either P2WSH or P2WPKH.

### Accept validation

#### User validation
* B fund amount: confirm that the fund amount matches what was agreed upon.

#### Machine validation
* Bob public key: validate that the public key is well formed.
* Bob inputs: validate that the referenced outputs are in fact unspent and are either P2WSH or P2WPKH.
* CET signatures: validate that the signatures are valid for each of the CETs.
* Refund signature: validate that the signature is valid.

### Sign validation

#### Machine validation
* Fund transaction signatures: validate that the signatures are valid for each of the provided inputs.
* CET signatures: validate that the signatures are valid for each of the CETs input.
* Refund signature: validate that the signature is valid for the refund transaction input.

## Contract cancellation

At any point in the protocol up until the sending of the `sign` message, both parties can safely decide to cancel the contract.
An empty `cancel` message can be used to inform the other party of the desire to cancel the contract.
If one of the party doesn't send a message after a given timeout, the other party should assume the contract to be canceled.
Timeouts should be proportional to the number of possible outputs of the contract, as the signing and verification of the CET signatures is expected to be the bottleneck.

After party A sends the `sign` transaction, they cannot unilaterally cancel the contract anymore.
If they want to cancel the contract, for example because of not observing the fund transaction in the Bitcoin network, they can try to do so by broadcasting a transaction spending at least one of the UTXO specified in the fund transaction.
If this transaction gets included in the blockchain prior to the fund transaction, the contract will be canceled as this will have rendered the fund transaction invalid.
In the even that the fund transaction gets included first, both party will be bound to the contract.

# Authors
Takatoshi Nakagawa
Ichiro Kuwahara
Thibaut Le Guilly
