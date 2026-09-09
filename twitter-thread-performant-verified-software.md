1/ Everyone assumes formal verification is a tax you pay on performance.

It keeps turning out to be the opposite. Once a proof has your back, you can pull all sorts of crazy tricks that would be way too dangerous in unverified code.

A few recent data points:

---

2/ powdr's autoprecompiles optimizer, rewritten and verified in Lean.

It already beat the original Rust on optimization quality. The runtime was the surprise: most circuits sit near 0.1 on the ratio, so roughly 10x faster than the code base it replaced.

---

3/ And it is not only us. Kim Morrison wrote up lean-zip, where the Lean implementation also outperforms the Rust one.

https://kim-em.github.io/blog/2026-7-24-why-lean-is-faster-than-rust/

---

4/ yul-compiler, our verified optimizing compiler from Yul to EVM.

We compile the Yul that solc emits before its own optimizer runs, then compare against solc fully optimized. Total gas: 99.8% of solc's, and Aave v4 comes out 17% cheaper.

A verified compiler gets to send it.

---

5/ Then there are the autoresearch challenges, where you point an AI harness at a metric and every submission ships with a machine-checked proof that it is still correct:

ecdsa.fail
zk.golf
snark.fast
better.codes
precompile.fast

---

6/ Verified code means more than correct. With AI writing the proofs, it means better metrics, faster runtime, higher quality engineering.

We used to pay for safety with performance. All we need are specs and proofs.

https://leoalt.de/performant-verified-software
