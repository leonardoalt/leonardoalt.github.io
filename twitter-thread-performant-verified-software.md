1/ Everyone assumes formal verification is a tax you pay on performance.

It keeps turning out the opposite. When the proof is the review, nobody audits the crazy trick, so you can let an AI optimize relentlessly and only require the proof still goes through.

A few data points:

---

2/ powdr's autoprecompiles optimizer, rewritten and verified in Lean.

It already beat the original Rust on optimization quality. The runtime was the surprise: most circuits sit near 0.1 on the ratio, so roughly 10x faster than the code base it replaced.

---

3/ Not only us. Kim Morrison's lean-zip tells the same story. His title is tongue in cheek, he is not claiming Lean beats Rust as a language. A proven implementation is one AIs can optimize freely.

30% faster than miniz_oxide, 8.8% better compression.

https://kim-em.github.io/blog/2026-7-24-why-lean-is-faster-than-rust/

---

4/ yul-compiler, our verified Yul to EVM compiler.

On the Aave and Uniswap tests it already generates code with better gas performance than solc.

A verified compiler is allowed to absolutely send it on codegen, in ways that would be too dangerous unverified.

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
