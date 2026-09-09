---
layout: post
title: "Performant formally verified software"
---

Formally verified complex software brings more than quality assurance: it allows for ultra-performant code,
since you can do all sorts of crazy tricks as long as you prove them safe, a luxury that unverified software does not have.

This has been successful before at [AWS](https://aws.amazon.com/blogs/security/an-unexpected-discovery-automated-reasoning-often-makes-systems-more-efficient-and-easier-to-maintain/), with organic humans writing proofs!
Now that we can just ask AIs to write Lean proofs for us, we have already seen many instances of such results.

## apc-optimizer

We [wrote before](https://powdr.org/blog/formally-verified-autoprecompiles) about FV of the autoprecompiles optimizer,
and the [impact of this new paradigm](https://georgwiese.github.io/posts/formal-verification-ai/) in software engineering.

The posts above show that the new verified apc-optimizer quickly outperformed the original Rust code base in optimization
metrics. The graph below shows that the verified code is also considerably quicker than the unverified code in runtime.
Each dot is a circuit, and every dot below the "1" line represents a case where the verified optimizer is faster.

<figure>
  <img src="{{ '/assets/apc-optimizer.png' | relative_url }}" alt="Scatter plot of runtime ratio between the verified Lean optimizer and the original powdr Rust optimizer, against circuit size, on log-log axes. Most points cluster around 0.1.">
  <figcaption>Runtime ratio (verified Lean implementation / original powdr Rust) per circuit, plotted against circuit size and colored by workload. Everything below the 1 line is a win for the verified code, and the bulk of the cloud sits around 0.1.</figcaption>
</figure>

Worth stressing that this is the same optimizer, on the same circuits, in the same pipeline, so the comparison is apples to apples.

## lean-zip

Kim Morrison [has written](https://kim-em.github.io/blog/2026-7-24-why-lean-is-faster-than-rust/) about a similar
experience with lean-zip, where the Lean code also outperforms the Rust code in runtime comparisons.

## yul-compiler

[yul-compiler](https://github.com/powdr-labs/yul-compiler) is an optimizer verified compiler from Yul to EVM.
Experiments with [Aave and Uniswap tests](https://github.com/powdr-labs/yul-compiler/pull/172#issuecomment-5372651982) show that
powdr's yul-compiler is already able to generate code with better gas performance than solc. This is not surprising
for the same argument presented in the introduction above. A verified compiler is allowed to absolutely send it and heavily optimize
codegen in any way possible, which would simply be too dangerous for an unverified code base.

<figure>
  <img src="{{ '/assets/yul-compiler.png' | relative_url }}" alt="Table comparing gas of powdr's yul-compiler output against solc's across the aave-v4, gasTests, semanticTests and uniswap-v4 corpora.">
  <figcaption>Gas comparison between powdr's yul-compiler and solc on the Aave and Uniswap test corpora.</figcaption>
</figure>

## Autoresearch challenges

Over the last months, there have been several autoresearch challenges: open competitions where you point an AI harness at a
well specified problem, and every submission has to come with a machine-checked proof that the result is still correct. They
have successfully optimized different systems for different metrics such as circuit size, ZK prover performance, EVM precompile
gas usage, and so on. A few examples:

* [ecdsa.fail](https://ecdsa.fail): a benchmark arena for cracking ECDSA, where you run your own autoresearch harness and submit your score.
* [zk.golf](https://zk.golf): build the cheapest zero-knowledge circuits, proven correct in Lean 4 against the spec.
* [snark.fast](https://snark.fast): make post-quantum Ethereum faster, with a live optimization dashboard.
* [better.codes](https://better.codes): a rewards program for machine-checked soundness improvements.
* [precompile.fast](https://precompile.fast): make Ethereum precompiles cheaper, across two live EIP-8200 gas frontiers.

The pattern across all of them is the same: once correctness is machine-checked, you can let the optimizer off the leash and
just keep score.

## Conclusion

Verified code means more than correct. With AI, it means more productivity, better metrics, faster runtime, higher quality engineering.
We used to pay for safety with performance. Turns out the caution that unverified code forces on you was costing us both all
along, and with proofs in hand you get to stop paying. All we need are specs and proofs.
