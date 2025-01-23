---
layout: default
title: RISC-V Business Part 1
author: Ramshreyas Rao
---
# RISC-V Business: Part 1

!["RISC-V Business"](https://szmer.info/pictrs/image/6313c4e4-5b15-41ee-afa8-a7765cda4893.png)

Angelina Jolie in Hackers, 1995

It's almost 2025, and the great RISC revolution that has been looming for decades is finally threatening to materialize. The Ethereum Foundation is [funding a formal verification effort for RISC-V ZKVMs](https://verified-zkevm.org/), chip industry legends like [Jim Keller](https://en.wikipedia.org/wiki/Jim_Keller_(engineer)) are starting RISC-V-driven technology efforts like [tenstorrent](https://tenstorrent.com/en/vision/tenstorrent-risc-v-and-chiplet-technology-selected-to-build-the-future-of-ai-in-japan), RISC-V International (the governing body for the RISC-V standard) has [ratified the RISC-V specifications](https://riscv.org/announcements/2024/04/risc-v-international-achieves-milestone-with-ratification-of-40-specifications-in-two-years/), laying the ground for an open standard that industry can now build upon, and Justin Drake in his ambitious [Beam Chain proposal](https://youtu.be/lRqnFrqpq4k) outlined how the onset of RISC-V can massively accelerate Ethereum's ponderous roadmap dramatically on many fronts.

This article aims to introduce RISC-V, motivate its importance to the Ethereum vision, and then introduce RISC-V zkVMs with a toy example of coding in this setting.

---

### 1. ZK is Ethereum's future

![The Verge](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/TheVerge.png)

[Possible futures of the Ethereum protocol, part 4: The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

As Vitalik outlines in his blog post about [The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html): 

> "One of the most powerful things about a blockchain is the fact that anyone can run a node on their computer and verify that the chain is correct. Even if 95% of the nodes running the chain consensus (PoW, PoS...) all immediately agreed to change the rules, and started producing blocks according to the new rules, everyone running a fully-verifying node would refuse to accept the chain... However, for this property to hold, running a fully-verifying node needs to be actually feasible for a critical mass of people... Today, running a node is possible on a consumer laptop (including the one being used to write this post), but doing so is difficult. The Verge is about changing this, and making fully-verifying the chain so computationally affordable that every mobile wallet, browser wallet, and even smart watch is doing it by default."

The goal of 'The Verge' is to make verifying blocks as easy as verifying a SNARK/STARK proof. Achieving this would mean the complexity of running a validator is reduced to just being able to download some data and verifying a 'ZK' proof. This reduces the barrier to entry for becoming a full verifying node, and increases Ethereum's censorship resisitance and resilience.

*Note: 'ZK' has unfortunately become a catch-all term which is not always accurate - we will address this technology as 'Programmable Cryptography' in the next section, but keep using ZK for now.*

But what does it take to verify such a proof, and, indeed, produce it in the fist place?

To understand this, let us first understand that we would be proving three kinds of things in each block:

1. [State](https://vitalik.eth.limo/general/2024/10/23/futures4.html#1): What are all the balances, addresses and raw data currently, and do they agree with the entire history of the chain? 
2. [Execution](https://vitalik.eth.limo/general/2024/10/23/futures4.html#2): Now that we know what the current state is, what is the post execution state transition, once all the transactions in the current block are executed? Has this been done correctly without any cheating?
3. [Consensus](https://vitalik.eth.limo/general/2024/10/23/futures4.html#3): Has the cryptoeconomics of the proof of stake mechanism (which ultimately secures the chain) been executed correctly? Have all the validator actions, balances, withdrawals signatures etc been updated correctly?

>"Stateless verification is a technology that allows nodes to verify blocks without having the entire state. Instead, each block comes with a witness, which includes (i) the values (eg. code, balances, storage) at the specific locations in the state that the block will access, and (ii) a cryptographic proof that those values are correct. "

State in Ethereum is currently stored in a (Merkle Patricia) tree structure, and is currently "extremely unfriendly to implementing any cryptographic proof scheme" per Vitalik. This is one of the motivations to move to a more friendly structure such as Verkle or STARKed binary trees. Changing this structure has already become a priority expressly to enable SNARK/STARK proving of state as described in the quote above.

>"...you should be able to verify an Ethereum block by (i) downloading the block, or perhaps even only small parts of the block with data availability sampling, and (ii) verifying a small proof that the block is valid... Getting to this point requires having SNARK or STARK proofs of (i) the consensus layer (ie. the proof of stake), and (ii) the execution layer (ie. the EVM)... Today, validity proofs for the EVM are inadequate in two dimensions: security and prover time."

Reducing the time taken to prove a block with sufficiently 'weak' hardware is an essential criteria for The Verge - software run by solo home stakers, or even users with a smartwatch for that matter, needs to be sufficiently fast enough so that an "Ethereum block can be proven in less than ~4 seconds.".

>"If we want it to be possible to fully verify an Ethereum block with a SNARK, then the EVM execution is not the only part we need to prove. We also need to prove the consensus: the part of the system that handles deposits, withdrawals, signatures, validator balance updates, and other elements of the proof-of-stake part of Ethereum."

While this is a significantly simpler problem than execution proving, it will need to be done 'from scratch' as these efforts are not also being driven by the Layer 2s, who are heavily investing in developing zk technology for execution, but not for consensus as they delegate consensus duties to the Layer 1. 

The main takeaway is this: the future of Ethereum is ZK (or PM, really, as we will introduce in the next section) - and there are significant hurdles to cross, especially in terms of performance, before we can get there.

---

### 2. Programmable Cryptography and the rise of 'Glue and Coprocessor' architectures

**2.1 Programmable Cryptography**

![PC Tech tree](https://0xparc.org/static/pc_tech_tree.png)

*[Diagram from 'Programmable Cryptography (Part 1)](https://0xparc.org/blog/programmable-cryptography-1)*

As introduced in a series of posts on [Programmable cryptography](https://0xparc.org/blog/programmable-cryptography-1) by [gubsheep](https://x.com/gubsheep):

>"Cryptography is undergoing a generational transition, from special-purpose cryptography to programmable cryptography."

Currently, the majority of cryptography use-cases are special-purpose - custom protocols which are designed to do one thing, and one thing only. Public key encryption and the signature schemes that underlie WWW certification are obvious examples. The next generation of 'Programmable Cryptography' introduces the notion of cryptographic primitives that are flexible and composable - "they allow us to perform general-purpose computation inside or on top of cryptographic protocols" per Gubsheep.

This means we now have the capacity to perform any computation our little hearts may desire, and get cryptographic verification that it was performed correctly, without having to eat glass and develop specialized cryptographic proofs for a multitude of critical functions scattered across the code. This suggests that with the magic of Programmable Cryptography, for example, we can get a ZK proof of valid block execution for all EVM protocols that were interacted with in that particular block, seemingly for free - without writing any specialized code. 

**2.2 The rise of Glue and Coprocessor architectures**

![Glue and Coprocessor](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png)

*[Diagram from 'Glue and coprocessor architectures'](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png)*

In his blog post ['Glue and coprocessor architectures'](https://vitalik.eth.limo/general/2024/09/02/gluecp.html), Vitalik describes another rising trend:

>If you analyze any resource-intensive computation being done in the modern world in even a medium amount of detail, one feature that you will find again and again is that the computation can be broken up into two parts:
>- A relatively small amount of complex, but not very computationally intensive, "business logic"
>- A large amount of intensive, but highly structured, "expensive work"

ZK versions of regular Ethereum code are exactly like this. The 'business logic' is the smart contract and EVM primitive execution code. The 'expensive work' is all the ZK proving of said execution, with esoteric moon math involving 'elliptic curves' and other costly computation. This is where Virtual Machines come in - if you write prover logic for a VM, you can prove any arbitrary code that can run in that virtual machine. This is how you get 'ZK for free' (nothing is free, as we will learn below) - without having to rewrite the smart contracts themselves.

But there's a problem - it is very, very expensive to virtualize the EVM (itself a virtual machine), and then couple execution of EVM code with proving - easily ten-thousand-fold regular EVM execution, per Vitalik. 

The solution? Create specialized low-level modules for all the expensive operations (often hashes, signatures, etc), and accept the (relatively miniscule) computational expense of running the business logic in the zkVM. This tradeoff is especially attractive for the kind of code typically found in smart contracts and regular Ethereum transactions. Vitalik again, in 'Glue and Coprocessor architectures':

>"Programmable cryptography is already very expensive; adding the overhead of running code inside a RISC-V interpreter is too much. And so developers have come up with a hack: you identify the specific expensive operations that make up the bulk of the computation (often that's hashes and signatures), and you create specialized modules to prove those operations extremely efficiently. And then you just combine the inefficient-but-general RISC-V proving system and the efficient-but-specialized proving systems together, and you get the best of both worlds."

**2.3 zkEVMs**

![zkEVMs](https://www.notion.so/image/https%3A%2F%2Fprod-files-secure.s3.us-west-2.amazonaws.com%2F80d2ba52-e0eb-4864-9603-a53e69a49dcc%2Fa54c7c60-f729-4c36-921c-29cb07ea8b27%2F1_(1).png?table=block&id=5aeb1325-0fa7-4865-930a-9a2592330e41&cache=v2)

[From risc0's blog - Desiging High-performance zkVMs](https://risczero.com/blog/designing-high-performance-zkVMs)

In August 2023, [Risc0 released Zeth, a ZK execution layer block prover for Ethereum and Optimism](https://github.com/risc0/zeth). Since then, multiple teams have followed suit, such as [Succinct Labs' SP1](https://github.com/succinctlabs/sp1). By running a compatible execution client like [Reth](https://github.com/paradigmxyz/reth) inside a zkVM like Zeth, we can obtain a ZK proof of valid block execution including:

- Verifying transaction signatures.
- Verifying account & storage state against the parent block’s state root.
- Applying transactions.
- Paying the block reward.
- Updating the state root.

There are actually [multiple types of zkEVMs](https://vitalik.eth.limo/general/2022/08/04/zkevm.html), occupying different parts of the trade-off space between full Ethereum compatibility and proving speed:

- Type 1: Fully Ethereum-compatible, inheriting all of Ethereum's infrastructure and security, but with longer prover times (the time it takes to generate zk-proofs).
- Type 2: Fully EVM-equivalent, striking a balance between compatibility and prover times.
- Type 2.5: EVM-equivalent except for gas costs, prioritizing scalability but with some compatibility limitations for certain dApps and tools.
- Type 3: Almost EVM-equivalent, making it easier to build and offering better scalability, but requiring some modifications to Ethereum dApps.
- Type 4: Compiles smart contract source code directly to ZK-SNARKs, achieving the fastest prover times but with the lowest compatibility with Ethereum dApps and tools. This variety of zkEVM types allows developers to choose the solution that best suits their specific needs and priorities.

[Since RiscZero’s pioneering initial release of their VM, general-purpose ZKVMs based on RISC-V have surged in popularity.](https://argument.xyz/blog/riscv-good-bad/)

So Programmable Cryptography enables us to abstract the ZK-proving code away from regular EVM execution by running the EVM inside a zkVM for which a prover has already been written. And this is made performant thanks to 'glue and coprocessor architecture', which uses specialized low-level modules to run cryptographic moon-math, and the tolerably expensive zkVM execution for the 'business logic'. 

But prima facie, zkEVMs do not need to use RISC-V. In fact, many of the early pioneer zkEVMs such as Polygon zero, ZKSync, scroll and others targeted various architectures in their own PC adventures. How does RISC-V enter the picture?

---

### 3. RISC-V is probably the best target architecture for zkEVMs

**3.1 RISC-V is open source**

RISC-V is a free and open source Instruction Set Architecture (ISA). It is an open standard managed by RISC-V international which has recently ratified the RVA23 Profile, which ensures software portability across any compliant hardware implementation. This is important because it provides a stable target for hardware companies to target. More importantly for Programmable Cryptography usecases, it is very flexible and customizable enabling the addition of specialized 'extentions' that can perform high speed cryptographic computation at a low level.

**3.2 RISC-V is modular and extendable**

RISC-V has been expressly designed to make it easier to add these 'extentions' to the ISA, and its licensing as well as the community posture welcome such extentions. This is in stark contrast to its primary competitor ARM (itself a RISC ISA), which is expensive, closed-source and prohibitively licensed, as evidenced by the [Qualcomm-ARM Nuvia Lawsuit](https://www.forbes.com/sites/patrickmoorhead/2024/11/07/qualcomm-and-arm-trade-jabs-ahead-of-december-court-case/). The whole glue and coprocessor approach requires custom extentions at a low level, and of all competing architectures such as x86, AMD, ARM and more, RISC-V is by far the most open and permissive architecture to target.

**3.3. RISC-V is growing**

In addition to an open standard, there is a rapidly growing ecosystem of toolchain support, such as compilers, debuggers and formal verification tools, not to mention a plethora of hardware startups targeting this new standard - all of which are essential for any entity considering using RISC-V.

**3.4 RISC-V is performance-oriented**

The simplicity of RISC, which stands for *Reduced* Instruction Set Architecture, [is by nature more efficient for the kinds of computation which will be required under Programmable Cryptography](https://risczero.com/blog/designing-high-performance-zkVMs). This is critical for achieving objectives such as real-time proving and single-slot finality. 

**3.6 Comparing RISC-V to Alternatives**

Comparing RISC-V to MIPS, WASM, and custom architectures reveals its strengths and weaknesses in the context of zkEVM development:

- RISC-V vs. MIPS: Both RISC-V and MIPS are RISC ISAs with mature ecosystems. However, RISC-V's open-source nature and growing community give it an edge in terms of flexibility, innovation, and accessibility.
- RISC-V vs. WASM: WASM's portability and web compatibility make it attractive for certain use cases. However, RISC-V's focus on performance and efficiency might be more suitable for the computationally intensive nature of zkEVM proof generation.
- RISC-V vs. Custom Architectures: Custom architectures can be optimized for specific zkEVM implementations, potentially offering performance advantages. However, they require significant development effort and might lack the community support and tooling available for RISC-V.

**3.7 Projects and Initiatives**

Several projects and initiatives are exploring the use of RISC-V for zkEVMs:




| Project| Description | Key Features |
|--------|-------------|--------------|
|RISC Zero|Develops a zkVM based on RISC-V that allows developers to prove the correct execution of arbitrary code written in Rust.|Supports Rust, can be used to build zkEVMs by running an EVM bytecode interpreter.|
|Succinct Labs|Builds a zkVM called SP1 that uses RISC-V as its ISA.|Designed to be highly performant and efficient, suitable for zkEVM implementations.|
|Nexus|Explores the use of RISC-V for zkEVMs.|Aims to provide a secure and scalable platform for decentralized applications.|
|a16z's Jolt|Develops a zkVM called Jolt that leverages RISC-V.|Designed to be developer-friendly and efficient, with a focus on improving the performance of zkEVMs.|
|Ethereum Foundation|Funds a project called "Verified zkEVM" that focuses on formally verifying the correctness of RISC-V-based zkEVMs.|Aims to improve the security and reliability of zkEVM implementations.|



**3.7 Drawbacks and Challenges**

While RISC-V offers numerous advantages, some drawbacks and challenges need to be considered:

- **Performance Bottlenecks**: Although RISC-V is generally efficient, certain instructions or operations may still present performance bottlenecks in zkEVM implementations. Developers need to carefully optimize their code and consider using specialized hardware or accelerators to mitigate these bottlenecks.
- **Compiler Bugs**: Compilers can introduce bugs where the generated RISC-V assembly code does not accurately reflect the original high-level code. These bugs can affect the correctness and security of zkEVMs. Thorough testing and formal verification methods are crucial to identify and address these bugs.
- **Benchmarking Challenges**: Comparing the performance of different zkVM implementations, especially those using different ISAs, can be challenging due to the lack of standardized benchmarks. This makes it difficult to objectively assess the performance advantages of RISC-V compared to other architectures.
- **Potential for Performance Limitations**: As an open standard, RISC-V might not always have the same level of performance optimization as proprietary architectures like x86 or ARM, which are often tailored for specific use cases. This could potentially affect the efficiency of zkEVM implementations, particularly for computationally intensive operations. However, the flexibility and customizability of RISC-V allow developers to implement optimizations and extensions to address these limitations.

While RISC-V presents a strong case, it's crucial to acknowledge its limitations and explore alternative architectures that might offer different advantages.

RISC-V is as a strong contender for the best target architecture for zkEVMs due to several key advantages. Its open-source nature fosters a collaborative environment, encouraging innovation and community-driven development. The modular design allows for customization and optimization, while compatibility with existing languages like Rust enables developers to leverage their skills and reuse code, accelerating development. The growing ecosystem and toolchain support further contribute to the accessibility and rapid adoption of RISC-V-based zkEVMs.

While potential performance limitations and compiler bugs need to be addressed, the benefits of RISC-V for Ethereum and crypto use cases are significant. Its efficiency can improve scalability, reduce costs, and enhance security for zkEVM implementations. Compared to alternative architectures like MIPS, WASM, and custom architectures, RISC-V strikes a balance between performance, flexibility, and community support, making it a compelling choice for zkEVM development.

To conclude - it's likely that Ethereum infrastructure will extensively use zkVMs, especially in the execution layer. Hardware acceleration using custom extentions on RISC-V virtual machines and, indeed, chips are the likely way this will be achieved. This entire field is especially being driven by Layer 2's, who stand to gain by massively reducing blob cost by commiting ZK proofs instead of entire blocks to L1. Solutions like SP1 are already becoming viable and demonstrating significant performance gains - and numerous competitors are snapping at their heels. 

This possible future may well become a reality ahead of schedule thanks in no small measure to RISC-V.

---

### 4. RISC Zero Hello ZK

To get a concrete sense of all this, let's do a quick excursion into the world of RISC-V zkVMs with a very basic 'Hello ZK' in the RISC Zero VM. We will just follow the [RISC Zero Hello World tutorial](https://dev.risczero.com/api/zkvm/tutorials/hello-world), condensed to quickly give you a feel of this type of programming. 

**4.1 [What is a zkVM application?](https://dev.risczero.com/api/zkvm/tutorials/hello-world#)**

![zkVM application flow](https://dev.risczero.com/assets/images/from-rust-to-receipt-23117368c4f46d78c8cac3b753245a5a.png)

*[Diagram from RISC Zero dcoumentation](https://dev.risczero.com/api/zkvm/)*

As discussed above in the Programmable Cryptography section, the idea is to take any generic piece of code, execute it, *and* produce a proof of the valid execution of that code. This flow can be seen in the RISC Zero process diagram above:

- The 'Guest Code' is the code you want to provably execute (Rust code, in the case of RISC Zero)
- This code is compiled into an ELF binary (the executable format for the RISC-V ISA)
- The binary is executed inside the VM, and the session (a complete execution trace) is recorded
- The prover then checks and proves the validity of this session...
- ...and outputs a Receipt - which consists of the results of your execution and a proof 
- This receipt can be used by anybody to verify that this was a valid execution of the original guest code

So a zkVM takes some guest code as an input, and produces the results along with a  verifiable proof of the validity of those results. 

Now that we know how a zkVM application works, let's get started.

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
cargo risczero new hello-world --guest-name hello_guest
cd hello-world
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

**Host and Guest**

As you can see, we have 'host' and 'guest' folders, which both contain a main.rs file each. The guest code is the code that we want to provably execute, and it currently looks like this:

```rust
use risc0_zkvm::guest::env;

fn main() {
    // TODO: Implement your guest code here

    // read the input
    let input: u32 = env::read();

    // TODO: do something with the input

    // write public output to the journal
    env::commit(&input);
}
```

This guest code will be executed by the host, which handles the inputs and the final receipt output. To motivate the setting a little better, let us quickly remember what we are trying to do. 

- We have some guest code which we have knowledge of
- We want to run this code with our own (private, if necessary) inputs
- We want to verify that the outputs produced are correct based on our input, and our knowledge of the guest code

So the host has to take the guest code and an input, execute it, and return the results and a proof as a receipt. 

So let us now implement some dummy logic in the guest code - let us double the input provided to it by the host:

```rust=
use risc0_zkvm::guest::env;

fn main() {
    // TODO: Implement your guest code here

    // read the input
    let input: u32 = env::read();

    // double the input
    let doubled = input * 2;

    // write public output to the journal
    env::commit(&doubled);
}
```
So whatever input we give the host program, our guest code will double it. Let us look at the host code (line 27):

```rust=
// These constants represent the RISC-V ELF and the image ID generated by risc0-build.
// The ELF is used for proving and the ID is used for verification.
use methods::{
    HELLO_GUEST_ELF, HELLO_GUEST_ID
};
use risc0_zkvm::{default_prover, ExecutorEnv};

fn main() {
    // Initialize tracing. In order to view logs, run `RUST_LOG=info cargo run`
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::filter::EnvFilter::from_default_env())
        .init();

    // An executor environment describes the configurations for the zkVM
    // including program inputs.
    // A default ExecutorEnv can be created like so:
    // `let env = ExecutorEnv::builder().build().unwrap();`
    // However, this `env` does not have any inputs.
    //
    // To add guest input to the executor environment, use
    // ExecutorEnvBuilder::write().
    // To access this method, you'll need to use ExecutorEnv::builder(), which
    // creates an ExecutorEnvBuilder. When you're done adding input, call
    // ExecutorEnvBuilder::build().

    // For example:
    let input: u32 = 15 * u32::pow(2, 27) + 1;
    let env = ExecutorEnv::builder()
        .write(&input)
        .unwrap()
        .build()
        .unwrap();

    // Obtain the default prover.
    let prover = default_prover();

    // Proof information by proving the specified ELF binary.
    // This struct contains the receipt along with statistics about execution of the guest
    let prove_info = prover
        .prove(env, HELLO_GUEST_ELF)
        .unwrap();

    // extract the receipt.
    let receipt = prove_info.receipt;

    // TODO: Implement code for retrieving receipt journal here.

    // For example:
    let output: u32 = receipt.journal.decode().unwrap();

    // The receipt was verified at the end of proving, but the below code is an
    // example of how someone else could verify this receipt.
    receipt
        .verify(HELLO_GUEST_ID)
        .unwrap();
}
```

We can see that the input is currently an arbitrary large number. Let us replace it with 7 (please make the change on line 27), and include a print statement (after line 55) to output the result so we can see it:

```rust=
// These constants represent the RISC-V ELF and the image ID generated by risc0-build.
// The ELF is used for proving and the ID is used for verification.
use methods::{
    HELLO_GUEST_ELF, HELLO_GUEST_ID
};
use risc0_zkvm::{default_prover, ExecutorEnv};

fn main() {
    // Initialize tracing. In order to view logs, run `RUST_LOG=info cargo run`
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::filter::EnvFilter::from_default_env())
        .init();

    // An executor environment describes the configurations for the zkVM
    // including program inputs.
    // A default ExecutorEnv can be created like so:
    // `let env = ExecutorEnv::builder().build().unwrap();`
    // However, this `env` does not have any inputs.
    //
    // To add guest input to the executor environment, use
    // ExecutorEnvBuilder::write().
    // To access this method, you'll need to use ExecutorEnv::builder(), which
    // creates an ExecutorEnvBuilder. When you're done adding input, call
    // ExecutorEnvBuilder::build().

    // WE MODIFY THE INPUT HERE
    let input: u32 = 7;
    let env = ExecutorEnv::builder()
        .write(&input)
        .unwrap()
        .build()
        .unwrap();

    // Obtain the default prover.
    let prover = default_prover();

    // Proof information by proving the specified ELF binary.
    // This struct contains the receipt along with statistics about execution of the guest
    let prove_info = prover
        .prove(env, HELLO_GUEST_ELF)
        .unwrap();

    // extract the receipt.
    let receipt = prove_info.receipt;

    // TODO: Implement code for retrieving receipt journal here.

    // For example:
    let output: u32 = receipt.journal.decode().unwrap();

    // The receipt was verified at the end of proving, but the below code is an
    // example of how someone else could verify this receipt.
    receipt
        .verify(HELLO_GUEST_ID)
        .unwrap();

    // WE ADD THE PRINT STATEMENT HERE
    println!("Hello, world! I generated a proof of guest execution! {} is a public output from journal", output);
}
```

Please note that the print statement is after the verification of the receipt, so we expect 7 to be doubled by our guest code running in the host, and be printed upon completion only if the code was executed correctly. 

To run the code:

```bash
cargo run --release
```

As you will see, this takes a long time to process, and you should see someting like this at the end of the command output:

```bash
Hello, world! I generated a proof of guest execution! 14 is a public output from journal
```
---

### 5. Conclusion

And there we have it - a simple way to run generic code and produce not only the results, but a verifiable proof of the valid exeution of the code. While this is indeed a great unlock with so many obvious use cases for Ethereum as described above, you no doubt saw that it took significant time and resources to run this simple program. 

In part 2 of this article, we will dig into how RISC-V extentions can make the world of [real-time proving](https://www.fabriccryptography.com/blog/risc-zero-announcement) a reality - potentially a lot sooner than expected!

---

