# CoinJoin + Hardware wallets

CoinJoin is [coming to hardware wallets](https://twitter.com/HillebrandMax/status/1118019355713966085?s=09). And as usual, with new functionality come new attacks.

Relevant discussions:

- https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-August/014843.html
- https://github.com/trezor/trezor-firmware/issues/37

Current spec to fix it (work in progress): https://github.com/satoshilabs/slips/blob/slips-19-20-coinjoin-proofs/slip-0019.md

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

A modern standard for communication with hardware wallets is a [Partially signed Bitcoin transaction (PSBT)](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) that provides to the hardware wallet not only the transaction to sign but also some additional information. Important pieces of information in PSBT are derivation paths to derive a key for input signing, previous transaction outputs to calculate the fee and derivation paths of the output addresses to understand which outputs are change outputs.

Using this information the hardware wallet can calculate the actual amount the user is sending and the fee. It is a very nice and convenient transaction format and all hardware wallets should support it. Right now it is natively supported only by the ColdCard, but there is [a tool](https://github.com/bitcoin-core/HWI) to make other hardware wallets compatible with PSBT transactions.

# CoinJoin + Hardware Wallets

## A naive way to do CoinJoin

A naive integration is pretty simple:

- Our software wallet on the computer prepares the transaction, communicates with the CoinJoin server and when it is ready to sign we send the PSBT to the hardware wallet. This PSBT contains all meta information about the user's inputs and outputs but it doesn't contain any meta information about inputs and outputs of other users.
- Hardware wallet checks the inputs and the outputs and displays the information about transaction. If it is just a mixing transaction HW will display something like `sending <fee> to <many CJ addresses>`.
- The user can verify that the fee is ok and confirm a transaction
- The hardware wallet signs its inputs and the computer sends the PSBT to the server.
- The server then combines all PSBTs received from the participants and broadcasts the transaction.

Here the user confirms a transaction and he is aware that he is spending the `fee` to increase his privacy. Even if the computer is compromised user shouldn't lose anything but the fee. Ideally.

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

PSBT contains all the derivation information for user's inputs and outputs and looks like this:

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

With this transaction the hardware wallet calculates the amount being spent and shows that to the user that he is spending `0.02 BTC` for the fee - looks ok.

## Replay attack

### Important sidenotes

Keep in mind that we don't trust our computer - we consider it to be potentially compromised. Otherwise why do you need a hardware wallet? So our computer can send anything it wants to fool us and steal our money.

The hardware wallet only knows the information provided by the PSBT. If the PSBT doesn't contain information about one of the user's inputs the hardware wallet will think that it doesn't belong to the user. But without this information the hardware wallet will not sign the corresponding input as well. Also hardware wallets should be stateless - they shouldn't store information about transactions and inputs they've already seen or signed. Basically they should be able to forget everything as soon as the transaction is signed and returned to the computer.

Another thing to remember is that often to do a CoinJoin you need to sign almost the same transaction several times. CoinJoin transactions often fail - if one of the participants disappears and doesn't sign the transaction everyone else needs to start over and make a new transaction without the guy who dissapeared. So if we need to confirm every transactions on the hardware wallet we need to be ready that a single CoinJoin transaction will ask for confirmation several times. Also CoinJoin users like to do multiple CoinJoin transactions in a row to reach desirable anonymity level. This means that it's pretty common to sign multiple transactions one after another when using CoinJoin.

Also, if you use CoinJoin often you will have many unspent outputs with the same amount. That's how CoinJoin works - it generates many outputs of the same value.

From the points above we can construct an attack.

### The attack

Here is what the malware on our computer could do:

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
PSBT_1: CJ_transaction

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
PSBT_2: CJ_transaction

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

As each PSBT contains only partial derivation information, the hardware wallet thinks that only `user_in1` belongs to the user in the first PSBT and only `user_in2` in the second, but in each of them we have an output that belongs to the user - therefore HW displays to the user two CoinJoin transactions with a fee of `0.01 BTC` in each of them.

The user sees two confirmation requests, thinks that CoinJoin failed on the first try, confirms the transactions, the hardware wallet signs, the attacker's software combines these PSBTs and sends it to the CoinJoin server. The user saw twise that he is spending `0.01 BTC` and thinks that CoinJoin failed once but worked on the second try, in reality he signed the same transaction twice, lost `1.01 BTC` and the attacker got `0.99 BTC` to his address.

The problem here comes from the fact that we are dealing with transactions where inputs may belong not only to the user, but also to other people, and the hardware wallet can't reliably determine if other inputs are owned by the user or not. Therefore calculation of the spending amount is wrong. The same applies not only to CoinJoin transaction but to any transaction that uses external inputs.

# Solution: Ownership proof

The solution is not prefect yet, we are currently discussing how to make it reasonably complete. Here are my thoughts:

If we could prove to the hardware wallet that every input is or is not owned by the hardware wallet we could reliably calculate the spending amount. The proof should be signed by the keys required for this input and hardware wallet should be able to determine that this proof was generated by it. Other participants should learn nothing from this proof but the fact that the proof was signed by the key owner.

Also we need to keep in mind that we may have inputs with multisigs, Schnorr MuSig in the future, contain HTLCs, custom scripts etc. Even though these types of scripts are not very usable for CoinJoin as they decrease privacy, CoinJoin is not the only application where you can have external inputs. So it makes sense to construct something that would work for other applications as well.

The proof itself can be different depending on the application, but the signature(s) coming with the proof should be valid - it also works as a proof that the user will be able to sign the input.

When we spend an input we provide a scriptsig that will make our scriptpubkey evaluate to `true`. The proof could use the same logic - we need to provide a set of elements (in most cases just a single signature, but might be multiple signatures & public keys & other elements). The proof combined with the scriptpubkey should evaluate to `true` when we check signatures against the proof itself.

So the proof is actually a proof itself and a scriptsig for this proof:

```
proof = (p, scriptsig)
```

In case of P2WPKH it will be `(p, sig|pub)`.

This scriptsig contains signatures and other stack values such that scriptpubkey evaluation against scriptsig for message `p` succeeds. Data in the message itself can be anything, but for example it can be:

`p = (n*G, n*id_pub)`

Here `n` is a random nonce, `n*G` is a corresponding curve point. `id_pub` is a unique identity public key. It can be either a root public key of the wallet, or a public key of a fixed path, or whatever public ket wallet will be checking against. If multiple keys are required then we provide multiple `n*id_pub` points there and every co-signer shares `id_pub` with others during the setup phase.

The good thing here is that if we use multisig any signer can multiply `n*G` by their root private key and verify if the resulting point is in the proof or not. Also the proof message itself can be generated by the untrusted software wallet and hardware wallets just add signatures to it.

Proper verification of this scheme requires bitcoin script evaluation on the hardware wallet that can be problematic. On the other hand in case of P2(W)PKH inputs we end up with a single signature. Initially hardware wallets can support only known script types for these kind of transactions. Later it can be extended to miniscripts or even full bitcoin scripts.

## Possible implementation in PSBT

Complete proof for multisignature addresses requires several signatures from different signers, therefore it has to be combined when all the partial proofs are there, similar to `partial_sigs` in PSBT.

Partial proof in PSBT can look like this:

Proof message:
```
key: {0x10} // no need to include scriptpubkey here - it will be in PSBT as a different key
value: {n*G}|{n*P1}|{n*P2}| ... |{n*Pn} // however many id pubkeys are required or known
```

Partial proof signature
```
key: {0x11}|{n*Pi}
value: `{signature_i}`
```

Finalized proof:

```
key: {0x12}|{n*G}|{n*P1}|{n*P2}| ... |{n*Pn}
value: {scriptsig}
```

To verify the proof one needs to validate the signatures in the scriptsig against corresponding proof and execute the scriptpubkey with scriptsig. Signer can then decrypt the fingerprints with the ECDH key and verify that it owns one of the keys.

## User experiense

Confirming multiple transactions on the hardware wallet is annoying. Ideally the user should confirm the transaction once and then the whole coinjoin process should happen automatically, even if the coinjoin fails and the server needs to rebuild the transaction. 

For this purpose user commitment can be used. The hardware wallet can ask the user to confirm a certain transaction _type_, for example a transaction that contains particular inputs and outputs, and all other inputs and outputs can be anything as soon as they don't belong to the user. The hardware wallet then sends back the commitment that can be used later for automatic signing. 

In this scenario the user needs to confirm the CoinJoin transaction only once, and it will be signed by the hardware wallet automaticaly as soon as CoinJoin is ready.

Suggestions and comments are welcome.

TODO:
- Add lightning use-case - routing and opening channel from remote key
- Lightning attack: without a proof we can fool HW to sign a commitment tx from its own input