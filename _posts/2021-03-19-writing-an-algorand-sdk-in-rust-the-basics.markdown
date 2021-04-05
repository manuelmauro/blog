---
layout: post
title: "Writing an Algorand SDK in Rust: The Basics"
date: 2021-03-19
categories: Writing an Algorand SDK in Rust
---

[Algorand](https://www.algorand.com/) currently officially supports [SDKs](https://developer.algorand.org/docs/reference/sdks/) for JavaScript, Python, Java and Go. A [community SDK](https://developer.algorand.org/docs/community/#sdks) is available for Rust and I am personally developing an [alternative one](https://github.com/manuelmauro/algonaut).

Through this series of blog posts, I aim at getting you started with Algorand and Rust, giving you a clear idea of what an Algorand SDK does, and hopefully clear my mind and take some good design decisions for the project.

## Installation

Let's start by installing Rust. If you are running Linux or macOS it is going to be as easy as running the following command:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

but feel free to follow the [installation guide](https://www.rust-lang.org/tools/install).

Now it's time to get started with Algorand, we will need a node running on our machine. The easiest way to do this is using an [Algorand Sandbox](https://github.com/algorand/sandbox). The main requirement is [Docker Compose](https://docs.docker.com/compose/install/), once you have it installed, using the sandbox is straightforward, just run the following commands:

```bash
git clone https://github.com/algorand/sandbox.git
cd sandbox
./sandbox up
```

As described in the readme, the sandbox creates the following endpoints:

- `algod`:
  - address: `http://localhost:4001`
  - token: `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
- `kmd`:
  - address: `http://localhost:4002`
  - token: `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
- `indexer`:
  - address: `http://localhost:8980`

for a detailed description of Algorand node's artifacts refer to the [official documentation](https://developer.algorand.org/docs/reference/node/artifacts/). Here we can briefly say that `algod` (Algorand Daemon) is at the core of an Algorand node, it handles the blockchain, process the messages with other nodes, runs the consensus protocol, and writes blocks to the disk. Finally, as you can see, exposes an API that developers can use to interact with it.

`kmd` is Algorand's key management daemon, which is the process responsible for handling client's private keys. It can create or import spending keys, sign transactions, and interact with key storage mechanisms. Being a separate process, a user can run it on a different machine effectively isolating the keys from the network. Also, this process exposes an API that developers can call in their applications.

Finally, the `indexer` is an optional component that allows searching the blockchain for transactions, assets, accounts, and blocks with various criteria.

The sandbox also creates its own private network `sandnet-v1` with a list of accounts owning some **ALGO**, which will be very useful in the next section. In order to see this list just run:

```bash
./sandbox goal account list
```

## Introducing Algonaut: an Algorand SDK Written in Rust

[`algonaut`](https://github.com/manuelmauro/algonaut) is a community SDK and contributions are always welcome! Please feel free to open issues or send pull requests on the project's GitHub page.
Speaking of GitHub, let's head to the [repository](https://github.com/manuelmauro/algonaut) and clone it. We will send a transaction on the private Algorand network kindly offered by our sandbox.

If your sandbox is running and you found the list of accounts with a positive balance in your wallet we are ready to go.

First, create a new project with:

```bash
cargo new my-first-transaction
```

then open the file `Cargo.toml` in the root folder of your new project and add the following dependency:

```toml
[dependencies]
algonaut = {git = "https://github.com/manuelmauro/algonaut", rev = "11808fa3650712bbd903560306400730ecc1d7b5"}
```

This is the single dependency that we will need for this simple example, now let's head to `/src/main.rs` and let's write some code.

Let's immediately get rid of the use statements by copying all this:

```rust
use algonaut::core::{Address, MicroAlgos};
use algonaut::transaction::{BaseTransaction, Payment, Transaction, TransactionType};
use algonaut::{Algod, Kmd};
use std::error::Error;
```

and let's start writing our main function.

```rust
fn main() -> Result<(), Box<dyn Error>> {
    let kmd = Kmd::new()
        .bind("http://localhost:4002")
        .auth("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa")
        .client_v1()?;
```

We will need a `kmd` client and as you can see to create one we need some information about our sandbox that we discovered earlier. Also, we will return a `Result` to easily manage some unwrapping using the `?` operator.

```rust
    let list_response = kmd.list_wallets()?;

    let wallet_id = match list_response
        .wallets
        .into_iter()
        .find(|wallet| wallet.name == "unencrypted-default-wallet")
    {
        Some(wallet) => wallet.id,
        None => return Err("Wallet not found".into()),
    };
    println!("Wallet: {}", wallet_id);

    let init_response = kmd.init_wallet_handle(&wallet_id, "")?;
    let wallet_handle_token = init_response.wallet_handle_token;
```

Now it's time to locate our default wallet and obtain a handle to it. In this case, our wallet has no password, so we just use an empty string.

```rust
    let algod = Algod::new()
        .bind("http://localhost:4001")
        .auth("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa")
        .client_v1()?;

    let transaction_params = algod.transaction_params()?;
    let genesis_id = transaction_params.genesis_id;
    let genesis_hash = transaction_params.genesis_hash;
```

It's time to start building our transaction. We will need an `algod` client and some parameters identifying our private network. We can retrieve them with `algod.transaction_params()`.

```rust
    let public_key = "INSERT-HERE-YOUR-ADDRESS";
    let to_address = Address::from_string(public_key.as_ref())?;
    let from_address = Address::from_string(public_key.as_ref())?;
    println!("Receiver: {:#?}", to_address);
```

Time to describe the sender and the receiver! We will use the same address for simplicity. Here you must insert one of the addresses with a positive balance in your wallet (you can use this command: `./sandbox goal account list`).

```rust
    let base = BaseTransaction {
        sender: from_address,
        first_valid: transaction_params.last_round,
        last_valid: transaction_params.last_round + 1000,
        note: Vec::new(),
        genesis_id,
        genesis_hash,
    };
    println!("Base: {:#?}", base);
```

That's enough information for our base transaction struct. Let's build it.

```rust
    let payment = Payment {
        amount: MicroAlgos(100_000),
        receiver: to_address,
        close_remainder_to: None,
    };
    println!("Payment: {:#?}", payment);
```

But we said that we would send some **ALGO**! So, let's prepare the `Payment` part of our transaction. Note that there are five different kinds of transactions that we can send on Algorand, thus the subdivision in `Base` plus `Payment` (you can read more [here](https://developer.algorand.org/docs/features/transactions/)).

```rust
    let transaction =
        Transaction::new_flat_fee(base, MicroAlgos(1_000), TransactionType::Payment(payment));
    println!("Transaction: {:#?}", transaction);
```

Now that we have the two components of our transaction we can build it! Also, notice that here we can specify the fees that we are willing to pay. In this example, we will use 1000 MicroAlgos.

```rust
    let sign_response = kmd.sign_transaction(&wallet_handle_token, "", &transaction)?;
    println!("Signed: {:#?}", sign_response);
```

Up to this point, anyone could have built such a transaction. This means that it is time to sign it! To do so, we just invoke `kmd`'s `sign_transaction` function, passing our wallet's handle, its password, and the transaction itself.

```rust
    let send_response = algod.raw_transaction(&sign_response.signed_transaction)?;
    println!("Transaction ID: {}", send_response.tx_id);

    Ok(())
}
```

All that is left is to broadcast the transaction to the network using `algod`. Congratulations, you sent your first Algorand transaction using Rust!

## Conclusion

If you think that it feels a bit too convoluted to send a transaction... I completely agree! This is because `algonaut` is just at the beginning. I am adopting an example-driven development approach to improve its API. Please contribute with issues, pull requests, examples, and let's discuss how we can make `algonaut` better on Algorand's [discord channel](https://www.algorand.com/)!
