# Rust API

Below is example usage of this library in rust. This is pseudo-rust; for the sake of simplicity.

The inputs and outputs to all methods use basic types: String or u64; 

### Submarine Swap:

`Client pays onchain address and recieves funds on lightning.`

#### Create Secrets

We assume our client has a master key in the form of a 12/24 word mnemonic.

We will use this as input and use the 'account' number to rotate keys via bip32 derivation.

The preimage will be extracted from the invoice provided by the client.


```rust
// The amount to be paid is set in the invoice
let invoice = "lntb500u1pjcjh3npp5llyysjq9a5cpsjrt535vxdg57fm0fjj89vhp4k5jz8kx8t8p9u3qdq9d9h8gxqyjw5qcqp2sp5ucymlq0czg73wgkzdwc70va8kdj3zt2lfgtq3z5javzkz0ptdlpqrzjq2gyp9za7vc7vd8m59fvu63pu00u4pak35n4upuv4mhyw5l586dvkfkdwyqqq4sqqyqqqqqpqqqqqzsqqc9qyyssqn09n6lg8uvq7lur4e6r0rzy6jep9ja2tw48pn2m97e39c3652qekmx9mupjr0reun3rtcsxfm8fyksztac0zrn6w5q3phgf7tzfxthcqu9ex3q";
let preimage_states = PreimageStates::from_invoice_str(invoice_str);
let mnemonic = "bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon";
let account = 1; // 0 is not allowed as this is used by the default wallet
let keypair = KeyPairString::from_mnemonic(mnemonic, "".to_string(), account);

```

#### Create Swap w/ Boltz API

We now need to create a Boltz API Client. This can also be done natively in dart.

```rust
let boltz_client = BoltzApiClient::new(BOLTZ_TESTNET_URL);

// get fees
let boltz_pairs = boltz_client.get_pairs();
let pair_hash = boltz_pairs
    .pairs
    .pairs
    .get("BTC/BTC")
    .map(|pair_info| pair_info.hash);

let request = CreateSwapRequest::new_btc_submarine(
    pair_hash,
    invoice_str.to_string(),
    keypair.pubkey,
);
let response: CreateSwapResponse = boltz_client.create_swap(request);

let id = response.id;

let redeem_script_string = response
    .redeem_script;

let timeout = response
    .timeout_block_height;

let boltz_pubkey = response
    .refund_pubkey;

let funding_address = response
    .funding_address;


```

#### Construct SwapScript

Now create the main SwapScript and SwapTx structures.

First, SwapScript:

```rust
let sub_swap_script = BtcSwapScript::new(
    BitcoinNetwork::BitcoinTestnet,
    DEFAULT_TESTNET_NODE.to_owned(),
    SwapType::Submarine,
    preimage.hash160.to_string(),
    keypair.pubkey,
    timeout,
    boltz_pubkey,
);

// check/validate boltz response elements
assert!(response.validate_script_preimage160(preimage.hash160));
assert!(sub_swap_script.validate_redeem_script_string(redeem_script_string));
assert_eq!(
    funding_address,
    sub_swap_script.to_address().to_string()
);
```

#### Construct SwapTx

With submarine swaps, we will prompt the client to pay the `funding_address`,
after which the invoice will be paid by boltz.

There is no `claim transaction` for the client; Boltz will claim the onchain funds.

However, incase of a dispute, we will need to construct an onchain `refund transaction`.

```rust
let absolute_fees = 300;
let mut sub_refund_tx = BtcSwapTx::new_refund(
    sub_swap_script,
    RETURN_ADDRESS.to_string(),
    absolute_fees,
);

// out_amount value is checked. drain will fail if amount does not match.
let signed_tx = sub_refund_tx.drain(keypair, preimage);
let txid = sub_refund_tx.broadcast(&signed_tx);
```

### Reverse Submarine Swap:

`Client pays Lightning invoice and recieves funds onchain.`

#### Create Secrets

We assume our client has a master key in the form of a 12/24 word mnemonic.

We will use this as input and use the 'account' number to rotate keys via bip32 derivation.

The preimage will be created using an rng. Boltz will use this preimage in their hold invoice.


```rust

let RETURN_ADDRESS="tb1xyz...";

let mnemonic = "bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon";
let account = 1; // 0 is not allowed as this is used by the default wallet
let keypair = KeyPairString::from_mnemonic(mnemonic, "".to_string(), account);
let preimage = PreimageStates::new();
let out_amount = 50_000; // the exact utxo value we want to claim onchain
```
#### Create Swap w/ Boltz API

We now need to create a Boltz API Client. This can also be done natively in dart.

```rust
let boltz_client = BoltzApiClient::new(BOLTZ_TESTNET_URL);

// get fees
let boltz_pairs = boltz_client.get_pairs();
let pair_hash = boltz_pairs
    .pairs
    .pairs
    .get("BTC/BTC")
    .map(|pair_info| pair_info.hash);

let request = CreateSwapRequest::new_btc_reverse(
    pair_hash,
    preimage.sha256,
    keypair.pubkey,
    out_amount,
);
let response: CreateSwapResponse = boltz_client.create_swap(request);

let id = response.id;

let redeem_script_string = response
    .redeem_script;

let timeout = response
    .timeout_block_height;

let boltz_pubkey = response
    .refund_pubkey;

let lockup_address = response
    .lockup_address;

```
#### Construct SwapScript

Now create the main SwapScript and SwapTx structures.

First, SwapScript:

```rust
let rev_swap_script = BtcSwapScript::new(
    BitcoinNetwork::BitcoinTestnet,
    DEFAULT_TESTNET_NODE.to_owned(),
    SwapType::ReverseSubmarine,
    preimage.hash160.to_string(),
    keypair.pubkey,
    timeout,
    boltz_pubkey,
);

// check/validate boltz response elements
assert!(response.validate_invoice_preimage256(preimage.sha256));
assert!(rev_swap_script.validate_redeem_script_string(redeem_script_string));
assert_eq!(
    lockup_address,
    rev_swap_script.to_address().to_string()
);
```

#### Check status

Invoice from boltz will be displayed on the client for payment
Once client pays the invoice, Boltz will fund the script.

We can monitor this in one of two ways: 

1. Check the swap status

```rust
let request = SwapStatusRequest { id: id.to_string() };
let response = boltz_client.swap_status(request);
assert!(response.is_ok());
let swap_status = response.status;
println!("SwapStatus: {}", swap_status);
if swap_status == "swap.created" {
    println!("Your turn: Pay the invoice");
}
if swap_status == "transaction.mempool" || swap_status == "transaction.confirmed"{
    println!("Ready to construct claim tx!");
}
```

2. Check the script balance

```rust
let script_balance = rev_swap_script
    .get_balance();
if script_balance.0 == out_amount || script_balance.1 == out_amount {
    println!("Ready to construct claim tx!");
}
// note balance is currently a tuple; however, it will be a struct {confirmed: u64 ,unconfirmed: u64}
```
#### Construct SwapTx

Now we need to construct the SwapTx and claim it:

```rust
// construct 1 sat/byte tx to get size; and then use fee_rate
let absolute_fees = 300;
let mut rev_claim_tx = BtcSwapTx::new_claim(
    rev_swap_script,
    RETURN_ADDRESS.to_string(),
    absolute_fees,
);


// out_amount value is checked. drain will fail if amount does not match.
let signed_tx = rev_claim_tx.drain(keypair, preimage, out_amount);
let txid = rev_claim_tx.broadcast(&signed_tx);
```

Incase of a Reverse Swap, `refund transaction` is the expiry of the Lightning invoice.


# Unified FFI API 

Language bindings written for the library will have a slightly different and more unified api, which can be found in src/api.rs

This will later be developed under a different crate `boltz-client-flutter`

The final dart API would ideally look as follows:

## dart

### SUBMARINE SWAP - PAYING EXTERNAL LN WALLET

m/purpose/network/21/0/*

```dart

import 'package:boltz_client_flutter/boltz_client_flutter.dart';
final network = BitcoinNetwork.Mainnet;
final electrumUrl = 'electrum.bullbitcoin.com:50002'
final boltzUrl = 'api.boltz.exchange';

final index = 12;
final keyPair = KeyPair.fromMnemonic('bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon, index);
final invoice = 'lnbc1...scanned-by-user-paying-external-ln-wallet...;

final fees = BtcSwapScript.getFees(boltzUrl);

final boltzBtcSubmarineSwap  = BtcSwapScript.newSubmarine(
    network,
    electrumUrl,
    boltzUrl,
    keyPair,
    invoice
);

final paymentDetails = boltzBtcSubarineSwap.paymentDetails();
// Construct local transaction to pay lockup_address from boltz with given amount
final txid = localWallet.send(paymentDetails.0, paymentDetails.1);
final status = boltzBtcSubmarineSwap.status();

// invoice will be paid by boltz. If not, we must refund after timeout
final address = localWallet.getNewAddress();
final absoluteFees = read.userInput();
final txid = boltzBtcSubmarineSwap.refund(address, absoluteFees);

```

### REVERSE SWAP - RECIEVING FROM EXTERNAL LN WALLET

m/purpose/network/42/0/*

```dart
final index = 13;
final keyPair = KeyPairString.fromMnemonic('bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon bacon', index);
final preImage = PreimageStates.new();
final outValue = 100000;
final fees = BtcSwapScript.getFees(boltzUrl);
final boltzBtcReverseSwap  = BtcSwapScript.newReverse(
    network,
    electrumUrl,
    boltzUrl,
    keyPair,
    preImage,
    outValue
);
final paymentDetails = boltzBtcReverseSwap.paymentDetails();
// Pay invoice provided by boltz (user is recieving a payment from an external LN wallet)
final status = boltzBtcReverseSwap.status();
// Construct swap transaction to claim lockup_address from boltz with given amount
// Genereate outputAddresss from local wallet
final address = localWallet.getNewAddress();
final absoluteFees = read.userInput();
final txid = boltzBtcReverseSwap.claim(address, fees);
```