# PSBT with multisignature

We have a very nice standard for communication with hardware wallets - [Partially Signed Bitcoin Transaction format](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki). Ideally PSBT should provide all necessary information to the hardware wallet not only to derive the signing keys but also to determine which of the outputs is a change output.

From the user experiense perspective hardware wallets should be able to reliably determine the change address and calculate the real spending amount. For single key addresses it works perfectly well - we have a `PSBT_OUT_BIP32_DERIVATION` field that allows the hardware wallet to verify that the output corresponds to a key owned by the wallet.

Unfortunately multisignature is a bit more complicated. Currently the derivation path contains only the fingerprint of the root key and the hardware wallet can verify only that its own key is included in the output.

## Multisignature attack

Below I describe how uncompleteness of the information currently defined in PSBT can cause loss of user funds in a m/n multisignature setup where `m ≤ n/2`.

Let's consider a multisignature setup where 1 of 2 keys are required to sign the transaction. This setup is chosen only for simplicity but can be easily applied for any `m` and `n` such that `m ≤ n/2`.

The evil coordinator (i.e. hacked watch-only software wallet) prepares the PSBT transaction with an input controlled by signers (i.e. hardware wallets) `A` and `B`. He includes two ouputs in the transaction: one output that was intended and a "change" output where he replaces the key of the signer `B` to the key of the attacker `E`.

He also includes the derivation information for signer `A`. He can also include derivation information for key `E` with the fingerprint of the signer `B` for both the input and the "change" output. He sends then the PSBT to the signer `A`.

Signer `A` can verify that his key controls the input and the "change" address using provided derivation information. As derivation information contains only the fingerprint of the root key of the signers and not the master public key, the signer `A` can't verify that the second key of the change address belongs to signer `B` - he can only compare the fingerprint but this information can be forged.

In this case signer `A` considers the output as a change and signs the transaction. Now the attacker can steal the funds from "change" address as he controls the key `E`.

## Solution: xpub field in PSBT

To verify that the change address uses keys derived from the same master keys we can add an xpub field in the PSBT inputs and outputs.

I suggest the following structure:

- Type: BIP 32 public key `PSBT_IN_BIP32_XPUB = 0x10`
	- Key: derivation path for xpub
		- `{0x10}|{master key fingerprint}|{32-bit int}|...|{32-bit int}`
	- Value: 78-byte xpub value
		- `{xpub}`

- Type: BIP 32 public key `PSBT_OUT_BIP32_XPUB = 0x03`
	- Key: derivation path for xpub
		- `{0x03}|{master key fingerprint}|{32-bit int}|...|{32-bit int}`
	- Value: 78-byte xpub value
		- `{xpub}`

Using these fields the signer can verify that the same xpubs are used for the input and the change output and that the scripts used in them have the same structure.
