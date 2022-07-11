# zkLlama Protocol Design

(by [@0xstrobe](https://twitter.com/0xstrobe), for the llamas 🦙)

Confidential token streaming vaults. Name is a place holder. :)

## How it works

zkLlama Vault contracts allow users to anonymously lock funds for confidential token streaming, by minting UTXO-like secret vault notes that derive configs and state (e.g. amount of funds, expiration time, etc.) from an encrypted on-chain dictionary. A user will then be able to consolidate, transfer, or liquidate the notes with any accounts they have access to, as long as they can prove they are the owner of the vault note, thus breaking the connection of depositors, vault parameters, and fund recipients, achieving complete anonymity.

There will be a corresponding zkLlama Vault for each ERC20 asset.

### Example / User story

A depositor `0xngmi` sets up a **12-month streaming** vault with a total of **$100,000.00 DAI** to **`0xstrobe`**. The only public info on-chain will be that `0xngmi` deposited $100,000.00 DAI to the `zkLlamaVault DAI` vault contract.

The owner of `0xstrobe` may also have access to account `0xsifu`. As long as `0xsifu` can prove it has access to `0xstrobe`'s private key, it will be able to manage unlocked funds in the streaming vault on `0xstrobe`'s behalf, essentially breaking the connection of `0xstrobe` and `0xngmi`. In the vanilla implementation described below, this private key is `0xstrobe`'s ETH wallet key, but it could be detached from any on-chain entity. If we use a keypair generated off-chain, this keypair will act as an _account_ within the shielded environment (of course the pubkey is never revealed either, it's always encoded in the hashes or ecrypted).

`0xsifu` can transfer the unlocked funds to other accounts (pubkeys), or consolidate the unlocked balance of multiple vaults into a single vault, or liquidate any of their vaults and withdraw the unlocked DAI.

## Data structure

### zkLlama Vault

zkLlama Vault operate on the data structure called _vault notes_. A vault note is denoted by a tuple that is the combination of 5 elements - public key of the note owner `pk`, token balance of vault `v`, timestamp of creation `t`, streaming rate (token/sec) `r`, and a random nonce `n`.

```
vaultNote = (pk, v, t, r, n)
```

A vault note resides in a dictionary in `zkLlamaVault` on-chain as a tuple `publicNote = (Hash(vaultNote), E_pk(vaultNote))`, where `E_pk()` denotes pubkey-encrypting of an input with key `pk`, and `Hash()` denotes the [Pedersen Hash](https://iden3-docs.readthedocs.io/en/latest/iden3_repos/research/publications/zkproof-standards-workshop-2/pedersen-hash/pedersen.html) function (used by [Tornado](https://iden3-docs.readthedocs.io/en/latest/iden3_repos/research/publications/zkproof-standards-workshop-2/pedersen-hash/pedersen.html)).

Let's call `Hash(vaultNote)` as the secret vault note. Since only the hashed secret note is saved on-chain, it's impossible to tell the pre-image that constitutes the hash and hence no one but the owner of the note knows the tuple `(pk, v, t, r, n)` constituting the note.

The encrypted version of the note `E_pk(vaultNote)` is for the owner to identify that a particular note belongs to them. If the user doesn't have the pointers of all the notes that belongs to them, the dApp will scan the vault contract events (or the dictionary) for all notes and try to decrypt each one with the user's private key to list all the notes that belong to a particular user,. Successfully decrypted notes are the ones that belong to the user. (These notes will be cached in `localStorage` for faster future access.)

When interacting with the zkLlama app, the dApp will derive from the `v, t, r` values of how much funds are unlocked, or how much funds are still pending.

### zkLlama Unlocked vault

The unlocked vaults look exactly like the vault notes from the outside, and simply have their `v, t, r` values set up so, that all the amount `v` is already unlocked at the current block timestamp.

One recommended use case of an unlocked vault is that when a depositor wants to set up streams for multiple accounts, and to shield away how the funds are distributed in different streams.

For example, `0xstrobe` may want to set up a 12-month streaming vault for `Spotify` ($10 USDC/mo), `Netflix` ($16 USDC/mo), and another 24-month streaming vault for `AmazonPrime` ($15 USDC/mo). They are therefore recommended to set up an unlocked vault with a total of $672 USDC, then perform 3 Shielded Mint operations to set up the 3 streams.

Another benefit of creating unlocked vaults could be that a user can consolidate the funds from multiple streams into one, or transfer unlocked funds to other owners, with all these operations never leaving the shielded environment of the zkLlama protocol.

## Operations implementation

### Setting up a stream (public `Mint` transaction)

A depositor sets up a token streaming vault by minting a vault note on-chain.

Conceptually, the depositor configures the token amount, vesting period, recipient, and generates an off-chain data note `vaultNote = (pk, v, t, r, n)`, and calculates `publicNote = (Hash(vaultNote), E_pk(vaultNote))` off-chain, then submits it to the zkLlamaData contract. The vault contract will at the same time transfer `v` amount of the ERC20 token to itself.

Zero-knowledge proof comes into the picture here since the depositor can write off a larger `v` in the hidden `vaultNote`, but tell the contract to transfer away a smaller amount `w` of the token. We basically want to prove that the public tuple has the correct `v` value encoded in it.

The snark computation of this will be like:

```
C(publicNote, v,    // only publicNote and v are the public inputs
  pk, t, r, n) { // private inputs
    vaultNote = (pk, v, t, r, n)
    assert publicNote == Hash(vaultNote)
    return 1
}
```

The on-chain function counterpart will be like:

```
proof = Proof(publicNote, v); // the proof generated off-chain above
encryptedNote = E_pk(vaultNote); // computed off-chain

function mint(proof, encryptedNote) {
    require(proof is valid);
    require(ERC20Contract.transferFrom(msg.sender, address(this), proof.v));

    add {encryptedNote: proof.publicNote} to publicNotes;
}
```

### Interacting with a vault note (UTXO-like transaction)

The vault notes can be interacted with like UTXOs. Common operations (shielded mints, consolidating vaults, ...) can be all easily constructed based on transferring the shielded tokens.

To transfer a particular value of tokens to a receiver (another pubkey owner), the current owner will select a sum of vault notes whose unlocked net values are at least the value that they want to transact with. The current owner can also set up an arbitrary streaming configuration for the new owner of the transferred tokens.

This value (`value`) will be propagated to the receiver in form of a new vault note (`recVaultNote`), the leftover value will become a new unlocked vault note (`sndVaultNoteUnlocked`) assigned to the sender as well as the corresponding newly consolidated locked vault notes `sndVaultNote0`, `sndVaultNote1` and so on, depending on how many original notes were involved.

To be able to spend a particular vault note (or 2 notes), the owner will generate a zkSNARK to demonstrate the knowledge of the tuple that constitutes the note. Using 2 notes as an example:

1. Ownership of the secret key (`sk`) that corresponds to the public key (`pk`) that a note belongs to
2. Value of the note (`v0` and `v1`)
3. The vault configs (`t0`, `t1`, `r0`, `r1`)
4. Nonce (`n0` and `n1`)

Along with that, the owner (sender) will generate in this case 4 new notes - one that will belong to the receiver (public key `rpk`) and the rest constituting the left over change, both with updated vault configs and nonces.

Essentially like this:

```
t0' = block.timestamp
t1' = block.timestamp
tRec = block.timestamp
tSnd = block.timestamp

av0 = calcUnlocked(v0, t0, r0, block.timestamp)
av1 = calcUnlocked(v1, t1, r1, block.timestamp)
assert av0 + av1 >= value

(v0', r0') = calcNewConfig(v0, t0, r0, block.timestamp)
(v1', r1') = calcNewConfig(v1, t1, r1, block.timestamp)
assert v0' == (block.timestamp - t0) * r0 == v0 - (block.timestamp - t0) * r0
assert v1' == (block.timestamp - t1) * r1 == v1 - (block.timestamp - t1) * r1

vSnd = (av0+av1) - value
rSnd = infinity
recVaultNote = (rpk, value, tRec, rRec, nRec)
sndVaultNoteUnlocked = (pk, vSnd, tSnd, rSnd, nSnd)
sndVaultNote0 = (pk, v0', t0', r0', n0')
sndVaultNote1 = (pk, v1', t1', r1', n1')
```

The snark computation will then be like:

```
// all the public inputs are hashed public notes
C(block.timestamp                           // public input of current time to check maths later
  inputNote0, inputNote1,                   // public inputs of the 2 notes to spend
  recVaultNote, sndVaultNoteUnlocked,       // public inputs of the newly minted public note, and the leftover unlocked note
  sndVaultNote0, sndVaultNote1,             // public inputs of the leftover, fully locked notes
  sk, v0, t0, r0, n0, v1, t1, r1, n1,       // private inputs of the 2 notes to spend
  v0', t0', r0', n0', v1', t1', r1', n1',   // private inputs of the new notes to mint
  rpk, value, tRec, rRec, nRec,             // private inputs of the newly minted receiver note
  vSnd, tSnd, rSnd, nSnd,                   // private inputs of the newly minted receiver note
  ) {
    pk = computePublicKeyFromSecret(sk)     // checking if it's indeed the owner
    assert inputNote0 == Hash(pk, v0, t0, r0, n0)
    assert inputNote1 == Hash(pk, v1, t1, r1, n1)

    var av0 = calcUnlocked(v0, t0, r0, block.timestamp)
    var av1 = calcUnlocked(v1, t1, r1, block.timestamp)
    assert av0 + av1 >= value

    (v0', r0') = calcNewConfig(v0, t0, r0, block.timestamp)
    (v1', r1') = calcNewConfig(v1, t1, r1, block.timestamp)
    assert v0' == (block.timestamp - t0) * r0 == v0 - (block.timestamp - t0) * r0
    assert v1' == (block.timestamp - t1) * r1 == v1 - (block.timestamp - t1) * r1

    assert recVaultNote == Hash(rpk, value, tRec, rRec, nRec)
    assert sndVaultNoteUnlocked == Hash(pk, vSnd, tSnd, rSnd, nSnd)
    assert sndVaultNote0 == Hash(pk, v0', t0', r0', n0')
    assert sndVaultNote1 == Hash(pk, v1', t1', r1', n1')

    return 1
}
```

The on-chain function counterpart will be like:

```
// the proof generated off-chain above
proof = Proof(inputNote0, inputNote1, recVaultNote, sndVaultNoteUnlocked, sndVaultNote0, sndVaultNote1);
encryptedInputNote0 = E_pk(vaultNote); // computed off-chain

function transfer(proof,
              encryptedRecVaultNote
              encryptedSndVaultNoteUnlocked,
              encryptedSndVaultNote0,
              encryptedSndVaultNote1) {
    require(proof is valid);
    require(proof.inputNote0 and proof.inputNote1 exist and are not spent);
    spend proof.inputNote0 and proof.inputNote1;

    add {encryptedRecVaultNote: proof.recVaultNote} to publicNotes;
    add {encryptedSndVaultNoteUnlocked: proof.sndVaultNoteUnlocked} to publicNotes;
    add {encryptedSndVaultNote0: proof.sndVaultNote0} to publicNotes;
    add {encryptedSndVaultNote1: proof.sndVaultNote1} to publicNotes;
}
```

### Liquidating an unlocked vault

Since liquidating from a partially unlocked vault is equivalent to consolidating (transferring everything to self) and then liquidate, I'll try to save some time by only describing liquidations from an unlocked vault.

Whoever holds the vault owner pubkey can transfer the equivalent amount of ERC20 tokens to any ethereum accounts, including their own. The value of the vault note `v` will be a public input.

The snark computation will then be like:

```
C(publicNote, v, block.timestamp,   // public inputs
  t, r, sk, n) {                    // private inputs
    pk = computePublicKeyFromSecret(sk)
    assert isFullyUnlocked(v, t, r, block.timestamp)

    publicNote == H(pk, v, t, r, n)
    return 1
}
```

The on-chain function counterpart will be like:

```
proof = Proof(publicNote, v); // the proof generated off-chain above
encryptedInputNote0 = E_pk(vaultNote); // computed off-chain

function liquidate(proof, address sendTo) {
    require(proof is valid);
    require(proof.publicNote exists and is not spent);
    spend proof.publicNote;

    ERC20Contract.transfer(sendTo, proof.v);
}
```

## dApp implementation

The dApp needs to be constantly scanning the `publicNotes` and decrypting the notes to check if it belongs to the user. It should also integrate well into the existing LlamaPay app, since most of the features described here already have their non-zk protected public versions.

## Acknowledgements

The current protocol design is heavily inspired by [ZkDai — Private DAI transactions on Ethereum using Zk-SNARKs](https://medium.com/@atvanguard/zkdai-private-dai-transactions-on-ethereum-using-zk-snarks-9e3ef4676e22) by gigachad [@atvanguard](https://twitter.com/atvanguard).

So far I've mostly studied these:

- [Tornado Cash](https://tornado.cash/) and especially its documentation on [circuits](https://docs.tornado.cash/tornado-cash-classic/circuits), which is a great high-level technical explanation of how Tornado Classic works.
- [Introduction to zk-SNARKs](https://consensys.net/blog/developers/introduction-to-zk-snarks/) is a good practical introduction to the on-chain part of zk-SNARKs. Not enough info for circuits.
- [zkDocs](https://www.zkdocs.com/) has a ton of super detailed (and interactive) explanations on how the math works.
- The official [Circom docs](https://docs.circom.io/). Must study this to make circuits. Would love to see more practical and advanced examples in it though.

Currently the design is powered by zk-SNARK, but [Bulletproofs](https://crypto.stanford.edu/bulletproofs/) are quite fascinating as well (no trusted setup needed but slower to verify) so I'll keep digging. [EIP-1724 Confidential Token Standard](https://github.com/ethereum/EIPs/issues/1724) from Aztec also has a lot of good ideas in it. Right now this design is not compatible with EIP-1724 yet, but maybe I will refine the design in the next iteration and make them work well together. 

[Offshift](https://offshift.io/content/offshift_whitepaper_v1.pdf) also has some awesome zkAsset designs I need to go through.
