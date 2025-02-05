---
layout: default
title: RISC-V Business Part 2
author: Ramshreyas Rao
---
# RISC-V Business part 2: The Road to Real-time Proving

!["The Road"](https://ramshreyas.github.io/Ethereum-Articles/assets/imgs/00006_4272034470_0000_by_somesh7322_dgqlxcb-pre.jpg)

[Image by Somesh](https://www.deviantart.com/somesh7322/art/00009-969351387-0000-1012953609)

In [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg) of this series, we explored at a high level how the adoption of RISC-V as an open standard is likely to be the foundation on which the [programmable cryptography](https://0xparc.org/blog/programmable-cryptography-1) future of Ethereum will be built, using [Glue and Coprocessor architectures](https://vitalik.eth.limo/images/gluecp/BJdoaxqo0.png). We then used Risc0's VM to 'host' a simple piece of external code, executed it, and then produced a proof that it was executed correctly, to get a concrete, albeit simple caricature of this paradigm.

In Part 2, we will dive deeper first with an overview of [the recently launched and amazing ETHProofs.org](https://ethproofs.org), verify the validity of a block by checking a SNARK proof, followed by a hands-on demonstration of proving an Ethereum block using Zeth, Risc0's zkEVM implementation. This should give us a sense of all the bits and pieces that go into creating an Ethereum proof, and also how incredibly long it currently takes to produce it! 

Once we have the lay of the land, we can proceed to lay out the road to real time proving and discuss how RISC-V and especially its extentions can get us there.

---

### The ZK Schelling point with Ethproofs.org

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

To get a better sense of that, we must first understand how the proof of the validity of an Etherum block is created in the first place.

---

### The proof in the pudding - verifying valid execution of an Ethereum block

To do this, we are not going to start with a highly optimized proving system like SP1. Let us instead start with a 'plain vanilla' execution of an Ethereum block, prove the validity of the execution trace as [we saw in Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg), and then verify that the proof is indeed correct.

To do this, we will use Zeth - RISC0's test zkEVM implementation.

### Run Ubuntu 20.04 in a Virtual Machine

In [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg), we installed and ran RISC0's zkVM using Docker. However, we now want to run Zeth, which itself requires both the RISC0 zkVM, as well as docker - so to avoid a docker-in-docker situation with all it potential for complexity, we will instead run Ubuntu in a virtual machine using [QEMU/KVM](https://www.qemu.org/). Here we will provide the instructions for (Debian-based) linux and MacOS:

#### Linux 
[(skip to MacOS)](#MacOS)

Check for hardware virtualization support
Make sure your host supports hardware virtualization (Intel VT or AMD-V). You can check with:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

If the output is 0, your CPU or BIOS may not support virtualization or it is not enabled in the BIOS.

**Install KVM/QEMU and related tools**

On a Debian/Ubuntu host, run:

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

- qemu-kvm: QEMU hypervisor and KVM support
- libvirt-* & bridge-utils: Provide the virtualization environment and networking
- virt-manager: (Optional but recommended) a GUI to simplify VM creation and management

Add your user to the libvirt and kvm groups

```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

Then log out and log back in (or reboot) so that these group changes take effect.

**Download the Ubuntu 24.04 ISO**

Go to [Ubuntu Releases](https://releases.ubuntu.com/24.04.1/) and download the ubuntu-24.04.1-live-server-amd64.iso.

Launch virt-manager:

```bash
virt-manager
```

**Create a new VM:**

- Click File → New Virtual Machine.
- Choose “Local install media (ISO image or CDROM)”.
- Browse and select the Ubuntu 20.04 ISO you downloaded.
- Allocate memory and CPU cores - 8 GB RAM, 2 cores.
- Create or select a new virtual disk image 50 GB.
- Install with Docker - you will be given this option during installation
- Follow the prompts to finish.

**Start the VM**

Once created, the VM should appear in virt-manager’s list. Double-click to start it. This will boot from the ISO.

**Install Ubuntu**

- Follow the standard Ubuntu installation steps (choose language, keyboard, etc.).
- On completion, remove the ISO from the VM’s virtual CD drive (virt-manager → VM’s Settings → IDE/CDROM).
- Reboot the VM into the newly installed Ubuntu 24.04.

[(skip to RISC0 setup)](#risc0-setup)

#### MacOS

Please follow the instructions [here](https://medium.com/code-uncomplicated/virtual-machines-on-macos-with-qemu-intel-based-351b28758617) to install Qemu and get Ubuntu 24.04 up and running. Ensure you have at least 8GB RAM and 50GB of Disk space.

[Install Docker](https://docs.docker.com/engine/install/ubuntu/) once you have booted the virtual machine.

Once you are in the Ubuntu shell verify that docker is running:

```bash
sudo docker ps
``` 

You should see an empty list of containers. You can now proceed to the next step.

### RISC0 Setup

As we did in [Part 1](https://hackmd.io/zl82XRhXQYuVB84GEZmBJg), we will first create a docker environment and install the RISC0 zkVM [(skip if you've already done this)](#zeth):

Let's install some libraries we will need:

```bash
sudo apt update
sudo apt install curl build-essential git wget pkg-config libssl-dev
```

**Installing Rust and rustup**

If you haven’t installed Rust yet, follow [the official instructions](https://www.rust-lang.org/tools/install):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Follow the prompts to install Rust. Once installed, you can verify with:

```bash
. "$HOME/.cargo/env"
rustc --version
cargo --version
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

### Zeth Setup

Make sure you have [installed Docker](https://docs.docker.com/engine/install/ubuntu/) (required by Zeth). Then, ensure your user can run Docker without sudo:

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```

Restart the virtual machine so the changes can take effect.

**Install Just** 

```bash
sudo snap install --edge --classic just
```

Go to [the Zeth repo](https://github.com/risc0/zeth), clone it, and enter the root directory.

```bash
git clone https://github.com/risc0/zeth.git
cd zeth
```

Clone the repository and build for CPU Proving (use 'just metal' for apple/metal and 'just cuda' for nvidia/cuda if your VM setup allows):

```bash
just build
```

