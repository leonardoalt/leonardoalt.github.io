---
layout: post
title: "What is the best ISA for Ethereum?"
---

*Special thanks to Alex Beregszaszi, Christian Reitwiessner, Georg Wiese, Guillaume Ballet and Justin Drake for review and feedback.*

About a month ago Vitalik's post on using RISC-V in Ethereum [[1]](https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617) led to several different paths being considered by a large group of people, re-opening some of the execution environment debate. Anticipating a lot of discussions, breakout sessions, and panels trying to answer that question during Berlin Blockchain Week, I wrote some notes that turned into this document.

The purpose here is to do a high level analysis of the pros and cons of different VMs/ISAs with respect to a few selected goals, injected with my personal opinion at times, as well as speculation on what could happen in some scenarios that are still under development. I will use some comments from answers to the post above, but I will **not** summarize the whole discussion. There are a lot of great comments there and you should read it for yourself. It is a long read though.

Note that Polkadot developers already went through a similar exercise [[2]](https://forum.polkadot.network/t/exploring-alternatives-to-wasm-for-smart-contracts/2434) that we should definitely learn from.

Here I'm considering options that already exist or are under active development, as opposed to trying to come up with something from scratch. These are:

- [EVM](https://ethereum.org/en/developers/docs/evm/)
- [RISC-V](https://riscv.org/)
- [MIPS](https://en.wikipedia.org/wiki/Loongson)
- [WASM](https://webassembly.org/)
- [eBPF](https://ebpf.io/)
- [CairoVM](https://book.cairo-lang.org/ch200-introduction.html)
- [Valida](https://lita.gitbook.io/lita-documentation/architecture/valida-zk-vm)
- [PetraVM](https://github.com/PetraProver/PetraVM)

We are trying to measure:

- Simplicity (of design and implementation)
- Execution performance (CPU and HW)
- Ecosystem
- Tooling
- Smart contract DX/UX
- ZK friendliness/performance

## EVM

The EVM is arguably the simplest next to RISCV and MIPS. Every ISA would need to be extended with Ethereum syscalls. If we consider the remaining EVM opcodes, the EVM is the simplest to write an interpreter for (again, next to RISCV and MIPS). Given that it is made for Ethereum, the bytecode generated from Solidity and Vyper is about an order of magnitude smaller than RISCV, for example, as I've personally observed in [R55](https://github.com/r55-eth/r55). However, EVM's 256-bit native word size and complex gas metering make JIT and specialized hardware support still an unsolved problem.

It has a huge, strong and cohesive ecosystem that causes widespread network effects. EVM developers (as in Solidity and Vyper developers) are (on average) quite security oriented: many if not most know of and are able to use tools for fuzzing and formal verification, topics that most developers in other ecosystems have never even heard of. Tooling has matured over the last years, and developers enjoy a huge selection of online material specific for development on Ethereum, and security researchers are developing eagle eyes at this point.

The EVM is not ZK friendly. First iteration zkEVMs that were built specifically for EVM are already being replaced by general purpose zkVMs that interpret EVM bytecode. This overhead could be reduced if there's more compatibility between the zkVM's ISA and the program being proven.

Sticking with EVM would mean that:
- Ethereum is happy with its developer community and tooling.
- Ethereum needs to somewhat focus on execution.
- General purpose zkVMs still need to get much faster.

In the original post, Lev actually suggests [[3]](https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/5) that the optimal case might actually be EVM + some container format + custom ISA for ZK proofs.

## RISC-V

RISC-V is very simple, quite modular and extensible. It can be JITed and runs in specialized hardware. CPU performance seems good according to PolkaVM experiments [[4]](https://forum.polkadot.network/t/announcing-polkavm-a-new-risc-v-based-vm-for-smart-contracts-and-possibly-more/) showing results that are 2x slower than wasmtime and cranelift. However, note that PolkaVM actually modifies the program [[5]](https://forum.polkadot.network/t/announcing-polkavm-a-new-risc-v-based-vm-for-smart-contracts-and-possibly-more/3811#the-compilation-pipeline-7)  [[6]](https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/122) after LLVM compilation with custom and WASM-inspiread ideas.

Extending RISC-V it with Ethereum syscalls should not be a problem, but I'm not sure what to do regarding gas metering. There was a good subdiscussion about that in the original post [[7]](https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/72). The bytecode is much larger than EVM's. This will be the case for most LLVM compiled programs.

RISC-V tooling is great, as the LLVM RISC-V backends are well maintained and you can compile Rust code to RISC-V with minimal effort. The developer community around RISC-V tooling is also quite strong. The smart contract developer community is limited.

However, there is great potential for smart contract DX/UX using RISC-V or WASM: Rust smart contracts. Established Rust smart contracts are quite ugly at the moment, but that can easily be improved considerably. [R55 contracts](https://github.com/r55-eth/r55/blob/main/examples/erc20/src/lib.rs#L54) show that they can be very neat and minimally invasive. Rust smart contracts enable a huge existing community to onboard into Ethereum with minimal effort. It also allows for existing security tooling to be used directly for smart contracts. Again, note that this is also true for WASM! So don't go around saying only RISCV enables this. I also do understand that many people don't necessarily value the argument above, but it's still true that it's a possibility. It's totally valid that a large portion of developers will still prefer Solidity or Vyper, and that's a good thing! Let people use whatever they want to use.

RISC-V is not designed to be ZK friendly, though of course many teams have already shown that it is possible to achieve great ZK proving performance via RISC-V. Most general purpose zkVMs use RISC-V as an IR for Rust programs aided by good tooling and the ISA's simplicity. RISC-V is the shortest path to make Rust ZK proofs practical. But is it the most performant? Unlikely. Several people including myself have given arguments on why other ISA's would be better for that purpose while still supporting high level Rust programs. The most obvious one being calling convention and how wasteful RISC-V is for local computation in number of cycles, highlighted by Mamy [[8]](https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/75) in the original discussion.

RISC-V may feel like a good compromise considering all the different goals and directions, but that still depends on what Ethereum truly wants to optimize for. It would require some customization in any case, which would be minimal compared to the other options, apart from the EVM. You would still have to trust (or well, verify) some level of custom compilation/transpilation. But if RISC-V, then I think there's a better way to do it, explained below in the conclusion.

## MIPS

MIPS is very similar to RISC-V. Well, or the other way around. RISC-V is generally considered a "modern MIPS". The main differences for this comparison being:

- RISC-V high level language tooling is better for Rust, but not in general. MIPS support in Go is quite mature. Optimism, for example, uses Go+MIPS in production for fraud proofs. MIPS has been used more and has a larger production community than RISC-V.
- MIPS on hardware is more mature than RISC-V, which can aid execution.
- MIPS seems to need fewer cycles while executing programs, which can be seen on EthProofs [[9]](https://ethproofs.org/blocks/22643600). This can lead to more efficient ZK proofs as well.
- The high level program likely to be used to target MIPS would be Go. Interestingly, Go has been used in production to write smart contracts in [CosmWasm](https://cosmwasm.com/), with an active community that could be potentially onboarded with a lower barrier to entry.

## WASM (WebAssembly)

One important note to start with is that WASM isn't an ISA per se, and it's generally higher level than EVM and RISC-V. In my opinion this sets a baseline for the comparison, which is that we would not use WASM as is, but rather further compile it to something lower level.

WASM is more complex than EVM/RISC-V. It has high level control flow, as well as more basic instructions. Its JIT performance is generally considered good, although that wasn't sufficient for eWASM to outperform EVM back then. WASM complexity has a reason: it has rigorous formal specs which provide compile-time guarantees regarding stack usage and validation that EVM/RISC-V can only dream of.

WASM tooling is arguably better than RISC-V's. It has great LLVM support and a large developer community. The smart contract developer community is limited, but likely larger than RISC-V's due to [Arbitrum Stylus](https://arbitrum.io/stylus).

As mentioned in the RISC-V section above, the potential for Rust smart contracts also applies to WASM seamlessly. In fact, Solidity used to have a direct compilation pipeline to eWASM.

WASM *can be* ZK friendly. We *just* need to transpile WASM to a ZK friendly ISA :) In my opinion this has big potential, and can enable ZK-optimized zkVMs to support not only DSLs but also high level Rust code (at least, hopefully other languages too). The reason for this is precisely WASM's higher level code. The one primitive we need is a Write Once Memory (WOM) region. This concept, pioneered in the zkVM context by CairoVM and also used by PetraVM, provides a much cheaper implementation of local computation in a zkVM. However, this isn't a native concept compilers are used to, so we have to build a compiler. With WASM code, we have enough information to turn locals and the stack into *Infinite Registers* which in turn are transpiled to WOM accesses in the zkVM (disclaimer: at powdr we are building precisely this - [WOMIR](https://github.com/powdr-labs/womir) - for PetraVM in collaboration with Irreducible).

Ultimately, by building a thin post-WASM compilation layer targeting specific zkVMs, it is likely that this path can yield better performance than a pure RISC-V zkVM. The price is more compiler work.

WASM has been considered for Ethereum before, but only as a standalone VM. From a compiler's perspective, it can be combined with any lower level VM, where combining it with the *right* VM could potentially lead to big performance gains. This could be PetraVM, RISC-V + extensions, or something else built from scratch.

## eBPF

eBPF was designed to provide native execution performance for guest code. In the blockchain context it is associated with Solanas' high performance smart contract execution. It is quite simple and likely scores the highest amount of points in the "execution performance" metric in this comparison. I don't know how one would implement gas in this context either.

Tooling seems to be available also via LLVM, but Polkadot devs noted that it may not be mature enough [[10]](https://forum.polkadot.network/t/exploring-alternatives-to-wasm-for-smart-contracts/2434#ebpf-4). If that works, Rust smart contracts could be enabled similarly to RISC-V and WASM. The smart contract ecosystem is likely the largest after EVM, with Solana attracting many new developers this and last year. DX could also be improved by using [R55](https://github.com/r55-eth/r55), for example.

eBPF's zkVM friendliness is likely similar to RISC-V's. It's not quite made for ZK proofs and some performance would be lost, but that could also be acceptable given the proven native execution benefits and existing smart contract community.

eBPF seems to have similar qualities and drawbacks to RISC-V, with likely better execution performance and worse tooling compatibility.

## CairoVM

CairoVM is a ZK-optimized general purpose zkVM that enables smart contracts on Starknet. It is rather simple to read and understand, and provides great insights into zkVM optimizations such as the Write Once Memory mentioned above. Its execution is interpreted, and JITing is likely difficult due to prime field arithmetics.

Tooling is specific to CairoVM, with the [Cairo language](https://www.cairo-lang.org/) being the main DSL to write programs targeting CairoVM. This is the main drawback that this approach would have in Ethereum. New compiler backends would need to be written, with the added difficulty of native prime field primitives as opposed to machine words. Without that users would be limited to Cairo only, which despite having a large community within Starknet would be a limitation for Ethereum.

## Valida

Valida is a new ZK-friendly ISA designed to support compilation from high level programs via LLVM. The Valida compiler implements a new LLVM backend, and therefore can perform many more high level compiler optimizations than RISC-V.

The pros and cons are similar to the WASM path mentioned above, although more extreme. We can compile high level programs to a ZK-optimized VM, and perform compiler optimizations (likely more here than in WASM, although not sure by what extent). However, this requires considerable extra work, much more than the WASM approach in the case of writing an LLVM backend. Since Valida supports Rust, it benefits from the same arguments as RISC-V and WASM with respect to Rust smart contracts and the community aspects.

Execution is interpreted and expected to be efficient at the assembly level. Valida is very promising and shows the potential gains [[11]](https://www.lita.foundation/blog/keccak-acceleration-chip-and-benchmarks) that can be made in ZK proof generation time.

## PetraVM

PetraVM is also a new ZK-optimized VM with one twist: it uses the Binius proof system as opposed to STARKs, used by most general purpose zkVMs. This means that prime field arithmetic is not an issue, as Binius supports powers of 2 as word sizes. This is of course compatible with the usual machine word sizes that high level languages and LLVM are used to. Execution can be done by an interpreter at the moment, with JIT and specialized hardware coming in the future. ZK proof generation is designed to happen on specialized hardware.

PetraVM being a custom new ISA and VM requires compilers. The PetraML language is being built as a DSL that compiles directly to PetraVM, and a compiler from WASM to PetraVM is being built to support high level programs as well. Similarly to Valida, with PetraVM supporting Rust via WASM, it also benefits from the same arguments with respect to Rust smart contracts and the community aspects.

The ISA natively defines a Write Once Memory, which enables powerful optimizations on the WASM side, as mentioned in the WASM section. It also contains the "usual" set of ALU/etc instructions, basically being a superset of RISC-V with ZK-friendly primitives.

It's early to tell, but on paper PetraVM does sound quite promising for Ethereum's goals as well.

## Personal Conclusion

The table below is an attempt at a visual approximation of the arguments given above. Please note that this is opinion based and nuanced, as not all circles of the same color mean the exact same thing, and we would need a lot more granularity to be fully accurate.

|                   | EVM | RISC-V | MIPS | WASM + WOM | eBPF | CairoVM | ValidaVM | PetraVM |
|-------------------|-----|:------:|:------:|:----------:|:----:|---------|----------|---------|
| Simplicity & Amount of Compiler Work        |🟢|  🟢 |  🟢     |      🟡      | 🟢/🟡    |    🟡/🔴     |     🟡/🔴     |    🟡     |
| Execution         |  🟡   |   🟡/🟢   | 🟢  |      🟡/🟢     |  🟢    |    🔴     |     🟡     |    🟡/🟢     |
| Tooling/Ecosystem & Smart Contracts |  🟢   |   🟢   | 🟡/🟢  |     🟢       |   🟢   |    🟡     |     🟢     |    🟢     |
| ZK-Friendly |  🔴   |   🟡   | 🟡  |      🟢      |  🟡    |   🟢      |    🟢      |    🟢     |

RISC-V still is the main hype for the future of Ethereum's execution environment. However, MIPS seems to have the best average next to RISC-V and potentially some advantages. Both are heavily powered by existing tooling, ease of use and simplicity. The preference for RISC-V in the zkVM community seems to be highly associated with using Rust instead of Go, as a lot of new crypto code is written in Rust. They are not the most ZK-optimized options. But do they have to be?

One option is to make a custom RISC-V extension with some primitives that would help it match other zkVMs in ZK performance. For example, a native Write Once Memory (WOM), mentioned above in the WASM and PetraVM sections. It provides a much cheaper way of implementing local computation in a zkVM, and could solve the RISC-V calling convention issue, as well as enabling other optimizations.

The trade-off is that it would require a custom compiler to make use of that. The counter-trade-off is that WASM is **very suitable** for this because its compile-time stack formal analysis - which RISC-V does not have - could lead to optimal allocation of stack and locals into such WOM. It could very well be that the optimal general purpose zkVM path is through WASM into a custom RISC-V extension, where the latter could also be implemented as PetraVM or a rather minor modification of an existing RISC-V zkVM.

Nothing is set in stone and I'm definitely looking forward to seeing all the developments made by various teams in the near future, as well as contributing to this effort.

## To Be Done

This document tries to provide a high level overview of the main pros and cons of different ISAs from the perspective of Ethereum L1. What it doesn't do, is weigh each of the metrics. The choice of which ISA to go for, if any change is required, will rely heavily on what's important for Ethereum. Once that is available, the technical discussions on each of the items and ISAs can be done in much more depth with specific goals in mind.

Another technical discussion that I explicitly avoided going in depth here is the choice of smart contract encoding to be stored on-chain. Different choices here lead to different requirements in terms of reproducible builds and verifiable transpilation. A topic for next Wednesday :)

---

- [1] https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617
- [2] https://forum.polkadot.network/t/exploring-alternatives-to-wasm-for-smart-contracts/2434
- [3] https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/5
- [4] https://forum.polkadot.network/t/announcing-polkavm-a-new-risc-v-based-vm-for-smart-contracts-and-possibly-more/
- [5] https://forum.polkadot.network/t/announcing-polkavm-a-new-risc-v-based-vm-for-smart-contracts-and-possibly-more/3811#the-compilation-pipeline-7
- [6] https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/122
- [7] https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/72
- [8] https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617/75
- [9] https://ethproofs.org/blocks/22643600
- [10] https://forum.polkadot.network/t/exploring-alternatives-to-wasm-for-smart-contracts/2434#ebpf-4
- [11] https://www.lita.foundation/blog/keccak-acceleration-chip-and-benchmarks
