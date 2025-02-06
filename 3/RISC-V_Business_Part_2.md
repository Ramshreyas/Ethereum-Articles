---
layout: default
title: RISC-V Business Part 2
author: Ramshreyas Rao
---
# RISC-V Business part 2: The Road to Real-time Proving

!["The Road"](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/00006_4272034470_0000_by_somesh7322_dgqlxcb-pre.jpg)

[Image by Somesh](https://www.deviantart.com/somesh7322/art/00009-969351387-0000-1012953609)

In [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg) of this series, we explored at a high level how the adoption of RISC-V as an open standard is likely to be the foundation on which the [programmable cryptography](https://0xparc.org/blog/programmable-cryptography-1) future of Ethereum will be built, using [Glue and Coprocessor architectures](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png). We then used Risc0's VM to 'host' a simple piece of external code, executed it, and then produced a proof that it was executed correctly, to get a concrete, albeit simple caricature of this paradigm.

In Part 2, we will dive deeper first with an overview of [the recently launched ETHProofs.org](https://ethproofs.org), followed by a hands-on demonstration of proving a toy Ethereum 'block' using Risc0's zkEVM implementation. This should give us a sense of all the bits and pieces that go into creating an Ethereum proof. 

Once we have the lay of the land, we can proceed to lay out the road to real time proving and discuss how RISC-V and especially its extentions can get us there.

---

### The 'ZK' Schelling point with Ethproofs.org

> Ethproofs is a block proof explorer for Ethereum. It aggregates data from various zkVM teams to provide a comprehensive overview of proven blocks, including key metrics such as cost, latency, and proving time.
> 
> Users can compare proofs by block, download them, and explore various proof metadata(size, clock cycle, type) to better understand the individual zkVMs and its proof generation process.

Ethproofs aggregates the proving time for multiple proof providers (currently [Snarkify](https://ethproofs.org/prover/1a13f366-1a1a-455e-8d82-9516b32dda75), [Succinct](https://ethproofs.org/prover/789bd1d9-158a-4dc7-bb9e-e0f12efda0d1) and [ZKM](https://ethproofs.org/prover/00a4d8ed-9272-42f8-a716-af1eead50de9)) and reports on all their performance parameters, highlighting the best one. The proofs for each can even be downloaded and verified by anyone. These provers are being run as single AWS instances in comparable and non-optimized deployments for all providers purely for the purposes of benchmarking and comparison - so only one out of every hundred Ethereum blocks is being tested currently.

![EthProofs](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/ethproofs.png)

In the image above from the landing page of the website, for each block that is proven, we can see multiple metrics:

1. **Gas used**: This is the total gas that was executed in that particular block, and is a proxy for the computational effort required to execute said block, and therefore the effort required to prove it. The bar gives an indication of its proportion of the total gas limit per block.
2. **Cost per proof**: This is an AWS-equivalent, hourly US dollar price estimate for the cost of producing the proof using hardware most similar to that being used by competing provers. The numbers for the best are shown with the average displayed below.
3. **Cost per Mgas**: This is a US dollar estimate of the cost of proving 1 Mgas worth of compute, using the two estimates above.
4. **Proving time**: The time taken to produce the proof of the given block.
5. **Proof status**: How many of the provers successfully completed the proof, are in progress, or are in queue to begin proving.

To understand this better, let us identify a block with almost complete gas limit utilization - [21728900](https://etherscan.io/block/21728900) (which will be buried in the history when you read this) and dig deeper on [Ethproofs.org](https://ethproofs.org/block/21728900) to get a sense of what the current state of the art (at the time of writing) is for block proving:

![A full block](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/fullblock.png)

These are the performance figures for each prover on block 21728900. As we can see, Succinct was the fastest at ~15 mins, and also the cheapest at ~7 cents an hour. Of course we must remember these are just unoptimized single-aws-instance deployments that are meant for comparisons - provers in production can use many kinds of optimizations and accelerations to deliver much faster proving times. The purpose of this is to establish the performance of 'raw' software, without any other confounders. As you can see in the last column of the table, we can also download the proof of the validity of the block and verify it.

[Ethproofs.org](https://ethprooks.org) is therefore an amazing public good that is creating a schelling point around the performance proving of zkEVMs. Companies like Succinct, Snarkify and ZKM stand to gain tremendously by topping this leaderboard, and the competition is just heating up - moving the entire space of zk proving forward in true Ethereum fashion.

So the race to achieve real-time proving is off to a great start. And clearly, proving is not commoditized yet, with zkVM companies already developing their own optimizations and accelration technologies to keep pushing the performance frontier. But what exactly are they doing? How does one get from the current state of the art to real-time proving?

To get a better sense of that, let's create a toy ethereum block, execute it, and prove its validity!

---

### 👋👋👋 A hand-wavy verification of the valid execution of a toy Ethereum block 👋👋👋

![zkEVMs](https://www.notion.so/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F80d2ba52-e0eb-4864-9603-a53e69a49dcc%2Fa54c7c60-f729-4c36-921c-29cb07ea8b27%2F1_(1).png?table=block&id=5aeb1325-0fa7-4865-930a-9a2592330e41&cache=v2)

[From risc0's blog - Desiging High-performance zkVMs](https://risczero.com/blog/designing-high-performance-zkVMs)

Let's back up a bit and review what we learnt in [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg). In this paradigm, we execute 'guest' code to a 'host' which executes that code with inputs, and produces the results along with a proof of the validity of said execution. So in the Ethereum context: 

- The 'guest' code is basically an EVM
- The inputs are all the transactions included in the block, and the previous block state.
- The 'host' is the (sophisticated) entity who produces the next block along with a proof (a block builder)

So essentially we need to be able to quickly verify that the block proposed is valid, given the included transactions and previous block state.

We are now going to do this for a criminally simplified Ethereum block and world state:

- There are only two participants relevant to this block, Alice and Bob
- Alice has 5 ETH and Bob has 0 ETH in the previous block
- The only transaction included in this block is Alice sending her 5 ETH to Bob

First, let's get our zkVM setup.

---

#### RISC0 Setup

As we did in [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg), we will first create a docker environment and install the RISC0 zkVM [(skip if you've already done this)](#Implementation):

**Docker**

RISC Zero requires some specific libraries to work (like libssl.so.1.1), so let's work in Docker to make sure everything works out of the box, and we can focus on getting familiar with RISC and zkVMs. If you haven't already, [install Docker](https://docs.docker.com/engine/install/), pull the [Ubuntu:20.04 image](https://hub.docker.com/layers/library/ubuntu/20.04/images/sha256-e5a6aeef391a8a9bdaee3de6b28f393837c479d8217324a2340b64e45a81e0ef), run, and enter it:

```bash
docker pull ubuntu:20.04
docker run -it ubuntu:20.04 /bin/bash
```

Let's install curl and build-essential, both of which we need:

```bash
apt update
apt install curl build-essential
```

**Installing Rust and rustup**

If you haven’t installed Rust yet, follow [the official instructions](https://www.rust-lang.org/tools/install):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Follow the prompts to install Rust. Once installed, you can verify with:

```bash
rustc --version
cargo --version
```

Install rustup

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Verify with:

```bash
rustup --version
```

**Install the RISC Zero Toolchain**

rzup is the RISC Zero toolchain manager, which can be installed with:

```bash
curl -L https://risczero.com/install | bash
```

Now install the toolchain and cargo-risczero with:

```bash
rzup install
```

**Create a new project**

Create a new project and cd into it:

```bash
cargo risczero new toy-proof --guest-name hello_guest
cd toy-proof
```

You now have a working template project which looks like this:

```bash
.
|-- Cargo.toml
|-- LICENSE
|-- README.md
|-- host
|   |-- Cargo.toml
|   `-- src
|       `-- main.rs
|-- methods
|   |-- Cargo.toml
|   |-- build.rs
|   |-- guest
|   |   |-- Cargo.lock
|   |   |-- Cargo.toml
|   |   `-- src
|   |       `-- main.rs
|   `-- src
`-- rust-toolchain.toml
```

---

#### Implementation

** Guest **

The 'guest' code is our toy EVM/State Transition Function. To keep things very simple we are not even going to remotely implement a real state transition where we would verify the signature of the sender, account for gas, update the (merkle particia trie) state etc. We will implement this fictional flow:

```mermaid
flowchart LR
    A[Read Block Input & Balances] --> B(For each Transaction)
    B --> C[Verify Signature]
    C --> D[Gas Accounting]
    D --> E[Run Opcodes]
    E --> F[Perform Balance Transfer]
    F --> G[Update Merkle/Verkle Trie]
```
Paste the following in to your methods/guest/src/main.rs code:

```rust=
use risc0_zkvm::guest::env;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

// ----- Data Structures for Our Toy Demo -----
#[derive(Serialize, Deserialize)]
struct BlockHeader {
    block_number: u32,
    parent_hash: [u8; 32],
}

#[derive(Serialize, Deserialize)]
struct Tx {
    from: String,
    to: String,
    amount: u64,

    // Instead of Option<[u8; 64]>, use Option<Vec<u8>>
    // This avoids the big-array limitation.
    signature: Option<Vec<u8>>,
}

#[derive(Serialize, Deserialize)]
struct DemoBlockInput {
    header: BlockHeader,
    transactions: Vec<Tx>,
    balances: HashMap<String, u64>,
    state_root: [u8; 32],
}

// ----- Placeholder Functions for “Ethereum-like” Steps -----
fn verify_signature(_tx: &Tx) -> Result<(), ()> {
    // Dummy: Always pass
    Ok(())
}

fn charge_gas_for_tx(_tx: &Tx) -> Result<(), ()> {
    // Dummy: No actual gas logic
    Ok(())
}

fn run_opcodes(_tx: &Tx) -> Result<(), ()> {
    // Dummy: No real EVM opcode execution
    Ok(())
}

fn update_state_trie(_balances: &HashMap<String, u64>) -> [u8; 32] {
    // Dummy: Return zeros for new root
    [0u8; 32]
}

fn main() {
    // 1. Read the toy block data from the host
    let serialized_input: Vec<u8> = env::read();
    let mut block_input: DemoBlockInput = bincode::deserialize(&serialized_input).unwrap();

    // 2. Process each transaction
    for tx in &block_input.transactions {
        // Placeholder: signature check
        if verify_signature(tx).is_err() {
            // In real logic, revert or skip
            continue;
        }
        // Placeholder: gas accounting
        if charge_gas_for_tx(tx).is_err() {
            // Revert or skip
            continue;
        }
        // Placeholder: EVM opcode execution
        if run_opcodes(tx).is_err() {
            // Revert or skip
            continue;
        }

        // Minimal “balance transfer”
        if let Some(balance) = block_input.balances.get_mut(&tx.from) {
            if *balance >= tx.amount {
                *balance -= tx.amount;
                *block_input
                    .balances
                    .entry(tx.to.clone())
                    .or_insert(0) += tx.amount;
            } else {
                // Not enough funds; real logic would revert or fail
            }
        }
    }

    // 3. Update the (dummy) state trie & store the new root
    let new_root = update_state_trie(&block_input.balances);
    block_input.state_root = new_root;

    // 4. Commit final (balances, root) to the journal so it’s publicly verifiable
    let final_balances_bytes = bincode::serialize(&block_input.balances).unwrap();
    let final_root_bytes = bincode::serialize(&block_input.state_root).unwrap();
    let combined_output = bincode::serialize(&(final_balances_bytes, final_root_bytes)).unwrap();

    env::commit(&combined_output);
}
```

If you look at the main function above, you will see the simple state transition function (with multiple dummy steps) in the lines 57 to 90. 

You will also have to update your methods/guest/Cargo.toml:

```yaml
[package]
name = "hello_guest"
version = "0.1.0"
edition = "2021"

[workspace]

[dependencies]
risc0-zkvm = { version = "1.2.0", default-features = false, features = ["std"] }

# Since we are enabling "std" in the RISC0 guest,
# we can use default features of serde & bincode that rely on std.
serde = { version = "1.0", default-features = false, features = ["derive"] }
bincode = "1.3.3"
```

So now that we have our guest code, the host can for each block, take the previous state, the list of new transactions, execute the state transition function, and produce the results along with a proof of valid execution. 

Change your host/src/main.rs to the following:

```rust=
// These constants represent the RISC-V ELF and the image ID generated by risc0-build.
// The ELF is used for proving and the ID is used for verification.
use methods::{
    HELLO_GUEST_ELF, HELLO_GUEST_ID
};
use risc0_zkvm::{default_prover, ExecutorEnv};

use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use tracing_subscriber;

#[derive(Serialize, Deserialize)]
struct BlockHeader {
    block_number: u32,
    parent_hash: [u8; 32],
}

#[derive(Serialize, Deserialize)]
struct Tx {
    from: String,
    to: String,
    amount: u64,

    // Change [u8; 64] to Vec<u8>, avoiding the big-array serde limitation.
    signature: Option<Vec<u8>>,
}

#[derive(Serialize, Deserialize)]
struct DemoBlockInput {
    header: BlockHeader,
    transactions: Vec<Tx>,
    balances: HashMap<String, u64>,
    state_root: [u8; 32],
}

fn main() {
    // Initialize tracing. In order to view logs, run `RUST_LOG=info cargo run`
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::filter::EnvFilter::from_default_env())
        .init();

    // Dummy initial balances
    let mut balances = HashMap::new();
    balances.insert("Alice".to_string(), 5u64);
    balances.insert("Bob".to_string(), 0u64);

    // A single transaction from Alice → Bob
    let tx = Tx {
        from: "Alice".to_string(),
        to: "Bob".to_string(),
        amount: 5,
        signature: Some(vec![0u8; 64]), // placeholder signature data
    };

    let block_input = DemoBlockInput {
        header: BlockHeader {
            block_number: 1,
            parent_hash: [0u8; 32],
        },
        transactions: vec![tx],
        balances,
        state_root: [0u8; 32],
    };

    // Serialize the block input
    let serialized_input = bincode::serialize(&block_input).unwrap();

    // Build the ExecutorEnv with input
    let env = ExecutorEnv::builder()
        .write(&serialized_input)
        .unwrap()
        .build()
        .unwrap();

    // Prove
    let prover = default_prover();
    let prove_info = prover.prove(env, HELLO_GUEST_ELF).unwrap();
    let receipt = prove_info.receipt;

    // Verify the receipt
    receipt.verify(HELLO_GUEST_ID).unwrap();

    // Decode final (balances, state_root) from the journal
    let combined_output: Vec<u8> = receipt.journal.decode().unwrap();
    let (final_balances_bytes, final_state_root_bytes): (Vec<u8>, Vec<u8>) =
        bincode::deserialize(&combined_output).unwrap();

    let final_balances: HashMap<String, u64> =
        bincode::deserialize(&final_balances_bytes).unwrap();
    let final_state_root: [u8; 32] =
        bincode::deserialize(&final_state_root_bytes).unwrap();

    println!("Final Balances: {:?}", final_balances);
    println!("Final State Root: {:?}", final_state_root);
    println!("I generated a proof of valid block execution!");
}
```

Remember, the host has the inputs and the previous state, which will be provided to the guest code for execution. So if you scan the code above, you can see that we: 

- Set up our initial state - which is just the balances (Alice has 5 ETH and Bob 0)
- Create the (only) transaction where Alice sends her 5 ETH to Bob
- Serialize this input to pass to the state transition function
- Verify the receipt, and if all goes well, 
- Print out the new balances and (dummy) state root

This proof can be verified by any entity to confirm the block was valid.

Also change your host/Cargo.toml to the following:

```yaml
[package]
name = "host"
version = "0.1.0"
edition = "2021"

[dependencies]
methods = { path = "../methods" }
risc0-zkvm = { version = "1.2.0" }
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Enable Serde derive macros (Serialize, Deserialize)
serde = { version = "1.0", features = ["derive"] }

# Add bincode for serialization / deserialization
bincode = "1.3.3"
```

Also ensure your Cargo.toml (at the root) looks like this:

```yaml
[workspace]
resolver = "2"
members = ["host", "methods"]

# Always optimize; building and running the guest takes much longer without optimization.
[profile.dev]
opt-level = 3

[profile.release]
debug = 1
lto = true
```

We are now ready to execute our block and produce a proof! At the root of the project, run:

```bash
cargo run --release
```

If all goes well, you should see something like this:

```bash
   Compiling methods v0.1.0 (/hello-world/methods)
hello_guest: Starting build for riscv32im-risc0-zkvm-elfs(build)                                                           
hello_guest:     Finished `release` profile [optimized] target(s) in 0.04s
   Compiling host v0.1.0 (/hello-world/host)
    Finished `release` profile [optimized + debuginfo] target(s) in 31.87s
     Running `target/release/host`
Final Balances: {"Bob": 5, "Alice": 0}
Final State Root: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
I generated a proof of valid block execution!
```

As we can see Bob has received his 5 ETH from Alice.

---

### 🍼 Glue and Coprocessor ⚙️

![Glue and Coprocessor](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png)

*[Diagram from 'Glue and coprocessor architectures'](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png)*

What was the point of this exercise? In [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg) we discussed Vitalik's article about the rising paradigm of glue (complex, but not computationally expensive) code that calls (highly structured, but computationally incredibly) expensive coprocessor code that can be run in a more optimized environment.

First of all, there are many opportunities to slice and dice the (pre-PECTRA) Ethereum State Transition Function (STF) into glue and coprocessor buckets (STF flow from [epf.wiki](https://epf.wiki/#/wiki/EL/el-specs)):

1. **Retrieve the Header**  
   - Obtain the header of the most recent block on the chain (the “parent block”).  
   - *Glue* — Light parsing and small data reads, mostly logic checks.

2. **Excess Blob Gas Validation**  
   - Calculate `excess_blob_gas` from the parent header and confirm it matches the current block’s header parameter `excess_blob_gas`.  
   - *Glue* — Simple arithmetic and comparison checks, minimal heavy computation.

3. **Header Validation**  
   - Compare and validate fields in the current block’s header against those of the parent block (e.g., timestamps, difficulty, etc.).  
   - *Glue* — Mostly logic-based validation, low computational intensity.

4. **Ommers Field Check**  
   - Ensure the `ommers` field in the current block is empty (no uncles).  
   - *Glue* — Straightforward check, minimal computation.

5. **Block Execution**  
   - Execute all transactions in the block, resulting in:
     - Gas Used: Total gas consumed.  
     - Trie Roots: Updated tries for transactions and receipts.  
     - Logs Bloom: Bloom filter for logs.  
     - State: Final state after all transactions.  
   - Mixed:  
     - *Glue* — Managing transaction order, reading/writing account fields, orchestrating EVM logic.  
     - *Coprocessor* — Cryptographic signature checks, hashing for storage tries, heavy precompiles like BN256 or Keccak.

6. **Header Parameters Verification**  
   - Confirm that execution outputs (e.g., `state_root`) match the corresponding fields in the block header.  
   - *Glue* — Mostly straightforward comparisons. (If large hashing or final accumulations are recalculated, that portion can be *Coprocessor*.)

7. **Block Addition**  
   - If all validations pass, append the block to the blockchain.  
   - *Glue* — Basic pointer updates, chain indexing.

8. **Pruning Old Blocks**  
   - Remove blocks older than the most recent 255 blocks.  
   - *Glue* — Housekeeping logic.

9. **Error Handling**  
   - If any checks fail, raise “Invalid Block.” Otherwise, return `None` (success).  
   - *Glue* — Control flow for passing/failing blocks.

There are largely two ways to speed up the code in the coprocessor buckets:

Precompiles in Ethereum act like built-in contracts that handle heavy-lift operations in a more optimized way than raw EVM bytecode. For example, cryptographic primitives such as [BLS12-381 curve operations](https://eips.ethereum.org/EIPS/eip-2537) can be invoked via these precompile addresses. At the protocol level, this cuts down on the large number of instructions a normal contract would need to perform these operations. Conceptually, these precompiles are a software-level “coprocessor”—the EVM recognizes them as special addresses that execute native code, returning results at a fraction of the gas cost and time it would otherwise require.

Pushing this idea further, one can imagine hardware acceleration of these same precompile tasks—particularly under RISC-V extensions—where specialized instructions handle elliptic-curve arithmetic, large integer math, or hashing at the hardware layer. Instead of funneling big cryptographic workloads through general-purpose CPU instructions, these expansions let you run them on dedicated circuits or co-processors optimized for parallel computation and low-latency arithmetic. 

But remember - our the task is not just to execute the block efficiently, we now want to run the whole STF inside a zkVM, and produce a proof for the validity of the execution. In this scenario, each step of block validation—header checks, transaction execution, Merkle trie updates—generates a cryptographic proof that every operation was carried out correctly, without needing a third party to re-run the computations. Because the zkVM must handle voluminous cryptographic operations (especially if it’s proving large arithmetic or hashing circuits), specialized RISC-V extensions become even more valuable here. By introducing instructions tailored for finite-field arithmetic, fast polynomial evaluations, or efficient hashing, the proof system can dramatically reduce the number of low-level steps required to finalize a proof. Instead of spending tens of millions of cycles on big integer math or repeated Keccak calls, the zkVM can offload those operations to dedicated hardware paths—essentially running “precompiled” or “accelerated” versions of those primitives natively on RISC-V. This combination of zkVM + hardware-enhanced cryptography unlocks new performance ceilings, potentially allowing real-time or near-real-time proof generation for complex Ethereum transactions and block validation. Taking this one step further, if we run this code directly on a RISC-V processor with cryptographic extentions rather than a VM, we can gain a large speedup just from losing the massive overhead of virtualization.

Finally, ensuring the correctness of the underlying RISC-V architecture itself via formal verification further cements trust in this zkVM model. If the RISC-V extensions and base ISA are proven to adhere to a rigorously specified, mathematically checked model, then the proof of execution isn’t just attesting to the correctness of higher-level operations (like hashing or signature checks)—it’s also anchored in the guarantee that each instruction in the CPU pipeline conforms precisely to the specification. This end-to-end assurance means that no hidden bugs or deviations in the hardware instructions can silently undermine the zero-knowledge proof process. In other words, formal verification of the RISC-V spec ensures that both the “coprocessor” tasks (like cryptographic math) and the “glue” logic (like branching and state transitions) faithfully follow the intended logic at the hardware level, extending trust from the software stack all the way down to the silicon.

---

### Conclusion - why RISC-V

In conclusion, RISC-V stands out as the natural foundation for a “glue + coprocessor + zkVM” paradigm precisely because it seamlessly unifies all the qualities necessary for real time proving. By design, modern zkVMs such as the current leader [SP1](https://www.succinct.xyz/) already use RISC-V as their core instruction set, leveraging its openness and flexibility. Because RISC-V is open source, developers can freely add cryptographic extensions—whether it’s specialized big-integer arithmetic, hardware Keccak, or elliptic-curve engines—without the licensing barriers of proprietary architectures. Meanwhile, the RISC-V ecosystem continues to grow and mature, offering a broader support base, toolchains, and commercial-ready solutions. Critically, formal verification work on the RISC-V spec ensures that these hardware-level enhancements remain provably correct, extending end-to-end trust from the software proof to the silicon.

All of these factors help bridge today’s reality—where a single Ethereum block proof might take 15 minutes—toward a future of real-time proving. By accelerating the cryptographic heavy lifting through specialized hardware, offloading structured tasks to “coprocessors,” and relying on a robust open architecture to handle “glue code,” RISC-V paves the way for generating valid zk proofs well within an Ethereum slot, adn the road to real-time proving. 