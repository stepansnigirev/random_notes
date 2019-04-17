# CoinJoin + Hardware wallets

It looks like CoinJoin is [coming to Wasabi Wallet](https://twitter.com/HillebrandMax/status/1118019355713966085?s=09). I think there is a way to attack hardware wallets users using CoinJoin.

## An easy way to do CoinJoin with existing hardware wallets

The zero-order integration is pretty simple:
- CoinJoin-enabled software wallet prepares the transaction, communicates with the CoinJoin server and when it is ready to sign it sends the [PSBT](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) to the hardware wallet.
- Hardware wallet operates normally without any changes - it checks the inputs and outputs and displays the information about transaction. If it is just a mixing transaction HW will display something that like `sending <fee> to <many CJ addresses>`.
- The user can verify that the fee is ok and confirm a transaction
- Hardware wallet signs its inputs and CJ wallet sends the PSBT to the server.

Here the user confirms a transaction and he is aware that he is losing the `fee` amount. Even if the computer is compromised user doesn't lose anything but the fee.

Let's say a transaction looks like this:

```
CJ_transaction:

inputs:
user_in1: 1 BTC
user_in2: 1 BTC
other_cj_inputs: ...

outputs:
user_out1: 0.99 BTC
user_out2: 0.99 BTC
other_cj_outputs: ...
```

PSBT contains derivation information and looks like this:

```
PSBT: CJ_transaction

inputs:
user_in1: 1 BTC
user_in2: 1 BTC
other_cj_inputs: ...

outputs:
user_out1: 0.99 BTC
user_out2: 0.99 BTC
other_cj_outputs: ...

inputs_metadata:
user_in1: key_derivation, prevout, ...
user_in2: key_derivation, prevout, ...
<empty for external inputs>

outputs_metadata:
user_out1: key_derivation
user_out2: key_derivation
<empty for external outputs>

```

With this information hardware wallet sees which inputs and outputs belong to the user and can calculate how much the user is spending: `sum(user_prevouts.amount) - sum(user_outputs.amount)`. The hardware wallet will show that the user is spending `0.02 BTC` for the fee - Looks ok.

## Replay attack

Keep in mind that we don't trust our computer - we consider it to be potentially compromised. Otherwise why do you need a hardware wallet? So our computer can send anything it wants to fool us and steal our money.

The hardware wallet only knows the information provided by the PSBT. In particular PSBT contains information on how to derive the keys for inputs and outputs that belong to the user. With this knowledge the hardware wallet can calculate the amount being sent by the user and display it. In case of CoinJoin hardware wallet can calculate the difference between inputs and outputs amounts and let the user know how much he is sending.

If we don't include infromation about a certain input, it will be considered external and not belonging to the user, but it also won't be signed by the hardware wallet.

Now let's see what happens if the user makes several CoinJoin transactions in a row (normally CJ users do multiple transactions to increase their anonymity set).

The user repeatedly verifies on the screen of the hardware wallet the amount he is spending in every CJ transaction and makes sure it is small (it's mostly just the fee of the coinjoin server + part of the tx fee). He confirms, sees how his anonymity set is increasing and he is happy.

What can the attacker's software do now:

1. Prepare a coinjoin transaction with multiple inputs from the user and outputs belonging both to the user and to the attacker:

```
CJ_transaction:

inputs:
user_in1: 1 BTC
user_in2: 1 BTC
other_cj_inputs: ...

outputs:
user_out1: 0.99 BTC
attacker_out: 0.99 BTC
other_cj_outputs: ...
```

If you use coinjoin often you will have many unspent outputs with the same amount. That's how CoinJoin works - it generates many outputs of the same value.

2. Prepare two PSBTs for the same coin join transaction with partial inputs metadata:

```
PSBT: CJ_transaction

inputs:
user_in1: 1 BTC
user_in2: 1 BTC
other_cj_inputs: ...

outputs:
user_out1: 0.99 BTC
user_out2: 0.99 BTC
other_cj_outputs: ...

inputs_metadata:
user_in1: key_derivation, prevout, ...
user_in2: <empty>
<empty for external inputs>

outputs_metadata:
user_out1: key_derivation
attacker_out: <empty>
<empty for external outputs>

```

```
PSBT: CJ_transaction

inputs:
user_in1: 1 BTC
user_in2: 1 BTC
other_cj_inputs: ...

outputs:
user_out1: 0.99 BTC
user_out2: 0.99 BTC
other_cj_outputs: ...

inputs_metadata:
user_in1: <empty>
user_in2: key_derivation, prevout, ...
<empty for external inputs>

outputs_metadata:
user_out1: key_derivation
attacker_out: <empty>
<empty for external outputs>

```

As each PSBT contains only partial derivation information the hardware wallet thinks that only `user_in1` belongs to the user in the first PSBT and only `user_in2` in the second - therefore it displays to the user two transactions with a fee of `0.01 BTC` in each of them.

The user confirms both transactions, the hardware wallet signs, the attacker's software combines these PSBTs and sends it to the CoinJoin server. The user thinks that he spent `0.02 BTC` and made two CoinJoin transactions, in reality he lost `1.01 BTC` and the attacker got `0.99 BTC` to his address in a single CoinJoin transaction.

## How to fix it?

I don't have a nice solution yet. Open for discussion.

Easy fix: the hardware wallet could warn the user that it already signed a transaction with this `txid` before, or that one of the inputs was already signed. It may require smarter hardware wallets, maybe not state-less.

Suggestions?
