# CoinJoin + Hardware wallets

Hardware wallets are [coming to Wasabi Wallet](https://twitter.com/HillebrandMax/status/1118019355713966085?s=09). I think it's not as easy as it looks — there is a way to attack existing hardware wallets using CoinJoin.

# Background 

## CoinJoin transactions

A CoinJoin transaction itself is just a big transaction with bunch of inputs from different people and bunch of similar outputs. With many similar outputs it becomes extremely hard to say which outputs correspond to which input — privacy increases.

A very simplified example of a CoinJoin transaction looks like this:

```
CoinJoin transaction:

inputs:
user1_in1: 0.7 BTC
user1_in2: 0.3 BTC
user2_in1: 1 BTC
user3_in1: 1 BTC
... many other inputs ...
userN_in3: 1 BTC

outputs:
user1_out1: 0.99 BTC
user2_out1: 0.99 BTC
... many other outputs ...
userN_out3: 0.99 BTC
```

When the transaction is sorted according to [bip69](https://github.com/bitcoin/bips/blob/master/bip-0069.mediawiki), all inputs and outputs are shuffled and as all outputs look the same it's hard to say who owns which outputs.

## Hardware wallets and PSBT

A modern standard for communication with hardware wallets is a [PSBT transaction](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) that provides to the hardware wallet not only the transaction to sign but also some additional information. Important is that it includes a derivation path to derive a key for input signing, previous transaction output to calculate the fee and a derivation path of the output addresses to understand which outputs are change outputs and go back to the user.

Using the information from PSBT the hardware wallet can calculate the actual amount the user is spending and the fee. It is a very nice and convenient transaction format and all hardware wallets should support it. Right now it is natively supported only by the ColdCard, but there is [a tool](https://github.com/bitcoin-core/HWI) to make other hardware wallets compatible with PSBT transactions.

# CoinJoin + Hardware Wallets

## An easy way to do CoinJoin with existing hardware wallets

A naive integration is pretty simple:

- Software wallet prepares the transaction, communicates with the CoinJoin server and when it is ready to sign it sends the PSBT to the hardware wallet. This PSBT contains all the information about user's inputs and outputs but it doesn't contain any information about inputs and outputs of other users.
- Hardware wallet checks the inputs and outputs and displays the information about transaction. If it is just a mixing transaction HW will display something like `sending <fee> to <many CJ addresses>`.
- The user can verify that the fee is ok and confirm a transaction
- Hardware wallet signs its inputs and CJ wallet sends the PSBT to the server.

Here the user confirms a transaction and he is aware that he is spending the `fee` to increase his privacy. Even if the computer is compromised user doesn't lose anything but the fee. Ideally.

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

PSBT contains all derivation information for user's inputs and outputs and looks like this:

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

With this transaction the hardware wallet will calculate the amount being spent and shows that to the user that he is spending `0.02 BTC` for the fee - looks ok.

## Replay attack

### Important sidenotes

Keep in mind that we don't trust our computer - we consider it to be potentially compromised. Otherwise why do you need a hardware wallet? So our computer can send anything it wants to fool us and steal our money.

The hardware wallet only knows the information provided by the PSBT. If the PSBT doesn't contain information about one of the user's inputs the hardware wallet will think that it doesn't belong to the user. But without this information the hardware wallet will not sign the corresponding input. Also current hardware wallets are stateless - they don't store information about transactions and inputs they've already seen or signed. Basically they forget everything as soon as the transaction is signed and returned to the computer.

Another thing to remember is that CoinJoin transactions often fail. If one of the users disappears and doesn't sign the transaction everyone else needs to start over and make a new transaction without the guy who dissapeared. So if you need to confirm every transactions on the hardware wallet be ready that a single CoinJoin transaction will ask for confirmation several times. Also CoinJoin users like to do multiple CoinJoin transactions in a row to reach desirable anonymity level. This means it's pretty common to sign multiple transactions one after another when using CoinJoin.

Also, if you use CoinJoin often you will have many unspent outputs with the same amount. That's how CoinJoin works - it generates many outputs of the same value.

From the points above we can construct an attack.

### The attack

Here is what the malware on our computer would do:

1. Prepare a CoinJoin transaction with several inputs from the user and outputs belonging both to the user and to the attacker:

```
CJ_transaction:

inputs:
user_in1: 1 BTC
user_in2: 1 BTC
... external_cj_inputs: ...

outputs:
user_out1: 0.99 BTC
attacker_out: 0.99 BTC
... external_cj_outputs: ...
```

2. Prepare two PSBTs for the same CoinJoin transaction with partial inputs metadata. Each of these PSBTs include derivation information only for one of the user's inputs:

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

As each PSBT contains only partial derivation information, the hardware wallet thinks that only `user_in1` belongs to the user in the first PSBT and only `user_in2` in the second, but in each of them there is the same output that belongs to the user - therefore HW displays to the user two CoinJoin transactions with a fee of `0.01 BTC` in each of them.

The user confirms both transactions, the hardware wallet signs, the attacker's software combines these PSBTs and sends it to the CoinJoin server. The user thinks that he spent `0.02 BTC` and made two CoinJoin transactions, in reality he lost `1.01 BTC` and the attacker got `0.99 BTC` to his address in a single CoinJoin transaction.

# Solution: A single user commitment per transaction

This attack is possible mostly because the user needs to confirm the same transaction several times. So instead I suggest getting a user commitment to a particular **type** of transaction and introduce delayed signing:

- Software wallet prepares the transaction inputs and outputs we want to inlude in CoinJoin transaction and maximum fee value that is expected for this transaction
- Hardware wallet displays this information to the user, user confirms
- Hardware wallet generates a commitment and sends it to the software wallet. This commitment says that the user confirmed a transaction that will have these particular inputs and outputs.
- Software wallet proceeds, and when CoinJoin transaction is ready to be signed it sends the PSBT together with the commitment to the hardware wallet
- Hardware wallet verifies that PSBT has inputs and outputs from the commitment and signs the transaction
- Software wallet sends the signed PSBT to the CoinJoin server
- If CoinJoin fails, the software wallet and the server construct another CoinJoin transaction, but it will contain the same inputs and outputs from the user - in this case software wallet can ask the hardware wallet to sign a new transaction with the same commitment.

If this is implemented, the user would need to confirm only once, and replay attack becomes impossible.

Suggestions and discussion are welcome.