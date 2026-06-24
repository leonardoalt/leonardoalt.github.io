---
layout: post
title: "evm-smith: human-readable smart contract specs in Lean, tied e2e to bytecode"
---

In [evm-smith](https://leonardoalt.github.io/evm-smith) we write smart contracts directly in EVM bytecode and prove properties about them in Lean, with no compiler in the trusted base. Most of what I have written so far is about the proof side. This post is about the other side: the spec, the thing a human actually has to read and agree with. I want it to read like a high level language, while staying a real Lean object that the bytecode is proven to satisfy.

Here is the whole behavioural spec of our WETH contract, the part an auditor reads:

```lean
deposit : ∀ s, Calls .deposit s →
  ensures
    balance[sender] = old balance[sender] + value
  ∧ untouched (others)
  ∧ returndata = empty

withdraw_debits : ∀ s, Calls .withdraw s →
  amount ≤ old balance[sender] →
  before_call:
    balance[sender] = old balance[sender] - amount

withdraw : ∀ s, Calls .withdraw s →
  amount ≤ old balance[sender] → NoReentrancy s →
  ensures
    balance[sender] = old balance[sender] - amount

fallback : ∀ s, Calls .unknown s →
  ensures
    storage = old storage
```

The two `withdraw` lines are the same debit seen at two points: `withdraw_debits` at the outbound call, with no extra assumption, and `withdraw` at the end of the whole run, assuming the recipient does not reenter. More on that distinction below.

The 86 bytes of bytecode are separately proven to satisfy each of these properties. Humans read the spec, and the proof is what ties it to the bytes.

## The goal: specs that read like the contract, but are still Lean

A formal proof is only as good as the statement you proved. If the statement is a wall of `EVM.State` projections, `Except` matches and fuel parameters, nobody reads it, and "we proved it" means very little. The interesting question for a project like evm-smith, asked by many people, is whether the spec can be as readable as the contract a developer would write, while remaining a genuine Lean proposition that the bytecode discharges.

We already have the proofs. So this post is an experiment in the readable surface: how far can we push a small embedded DSL in Lean so that the spec a smart contract developer reads looks like the intent of the contract, and an auditor can check it without learning the framework.

## The fun part: speccing what the compiler hides

To make it concrete, I picked properties that a Solidity developer (almost) never writes by hand, because the compiler generates them: the function selector dispatch and the ABI decoding of arguments.

When you call `deposit()` or `withdraw(5)`, the EVM does not know about functions. It sees calldata, a flat byte string. Solidity's compiler emits a prologue that reads the first four bytes of calldata, compares them against the known function selectors, and jumps to the matching code. Arguments are read at fixed offsets: the first `uint256` lives at bytes 4 to 36. None of this appears in the source; it is a convention the compiler implements and everyone trusts.

In evm-smith there is no compiler, so we wrote those bytes ourselves, and we can also state and prove what they mean. That is the gap we wanted to close: make the selector and the ABI decoding explicit in the spec, and tie them to exactly what a compiler would produce.

## Modelling it in Lean

The spec lives in one file, [`Spec.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/Weth/Spec.lean), and reads against a tiny vocabulary defined elsewhere. A few pieces do all the work.

`Calls .deposit s` says "`s` is the entry state of a call to `deposit`": the program counter is at the start, the stack is empty, the contract's code is installed, and the ABI selector in calldata is `deposit`'s. So a single predicate captures "this is a well formed call to this function".

`ensures Q` is the postcondition of running the contract to its halt. Inside `Q`, `balance[a]` reads the token ledger in the final state and `old balance[a]` in the initial state, `sender` and `value` are `msg.sender` and `msg.value`, `amount` is the decoded argument, and `untouched (others)` says every balance except the caller's is unchanged. There is also `before_call: Q`, which states a postcondition at the moment the contract makes its outbound call, before any external code runs.

That is essentially the whole DSL: a predicate for "a call to f", a couple of macros for "running it ensures", and readable accessors for the state. It is about 150 lines of plain definitions and notation, split across [`Spec/Dsl.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Spec/Dsl.lean) (the contract-agnostic part) and [`SpecDSL.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/Weth/SpecDSL.lean) (the WETH-specific part), with no metaprogramming beyond two small macros. If you want to change how a property reads, or check that a notation means what you think, you read those files, not a proof.

## Functions as big-steps, or Hoare triples

Look again at the `deposit` guarantee from the top of the post, with this lens:

```lean
deposit : ∀ s, Calls .deposit s →
  ensures balance[sender] = old balance[sender] + value ∧ untouched (others) ∧ returndata = empty
```

This is a big-step semantics over a "function" that responds to a selector under some precondition. Given any state that is a call to `deposit`, running the bytecode to a halt produces a state where the caller's balance grew by `msg.value`, nobody else moved, and nothing is returned. If you prefer the program-logic phrasing, it is a Hoare triple: a precondition `Calls .deposit`, the command "run the contract", and a postcondition relating the final state to the initial one. `withdraw` adds two more preconditions, sufficient balance and no reentrancy, and concludes the caller is debited by exactly the requested amount.

The `ensures` macro hides the part nobody wants to read. Underneath, "running to a halt" is a gas-free interpreter that steps the real EVM semantics instruction by instruction, decoding each opcode from the contract's own code, and the proof threads roughly fifteen to forty steps of dispatch and function body. The reader sees one hypothesis, "the contract ran", and the resulting state change. The instruction walk stays in the proof.

## Selector and ABI, pinned to the compiler

For the compiler-internal properties, we make the correspondence explicit and prove it. Two one line lemmas connect the spec's notion of "the selector" and "the argument" to the exact EVM operations a Solidity compiler emits:

```lean
selector_is_abi      : functionSelector cd = shr(224, calldataload(0))
withdraw_arg_is_abi  : withdrawArg s       = calldataload(4)
```

So when the spec says `Calls .withdraw s`, the selector it matches on is `shr(224, calldataload(0))`, the dispatcher prologue the compiler generates, and `amount` is `calldataload(4)`, the standard load of the first `uint256` argument. The abstract reading, that this equals the first four bytes and the calldata slice from 4 to 36, is the meaning of the `SHR` and `CALLDATALOAD` opcodes, and we take it as their semantics. The point is that the spec's selector and argument are, by construction, the ABI's, with nothing left implicit between the spec and what a compiler would do.

## Solvency reads the same way

The global safety property that we proved before gets the same treatment. WETH should never owe its users more ETH than it holds:

```lean
def Solvent σ weth := recordedTokenSupply σ weth ≤ ethBalance σ weth
```

and the guarantee, over any transaction:

```lean
solvent : ∀ ctx σ tx S_T weth, … → SolventAfter ctx σ tx S_T weth
```

"Run any Ethereum transaction, and if WETH was solvent before it is solvent after." Reentrancy is worth a note here, because it shows the DSL drawing a real distinction rather than smoothing one over. Solvency holds with no reentrancy assumption at all: `withdraw` writes its decrement before the external call, so a reentrant caller sees the lowered balance and cannot over withdraw. The only place a no reentrancy assumption appears is the exact end balance of a single `withdraw`, where a recipient that calls back and deposits would change the final number. The spec says exactly that.

## The bytecode is proven to satisfy all of it

`Spec.lean` is an interface: a structure whose fields are these guarantees, with no proofs. The proof that the 86 bytes satisfy every field is a separate witness, [`SpecProofs.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/Weth/SpecProofs.lean): one named theorem per guarantee, each discharged by a single line into the engine that does the instruction walk. Lean checks that each theorem matches its field, so the readable statement cannot drift from what is proven. The whole thing depends only on Lean's standard axioms, with no `sorry` and no contract specific axiom.

So a human reviewing this contract reads `Spec.lean`: a handful of pre and post conditions in a vocabulary that looks like the contract's intent, including the selector and ABI behaviour that a compiler would normally hide. Everything below it, the interpreter, the instruction by instruction walk, the byte level decode lemmas, is machinery that Lean has already checked.

## Closing

The point of evm-smith was contracts in bytecode with Lean proofs and no compiler in the trusted base. The risk with that is that the spec becomes unreadable and the guarantee becomes a black box. This experiment shows that it does not have to. We can write specs in Lean that read like a high level language, capture even the conventions a compiler usually hides, and still be the exact statement the bytecode is proven to obey. The readable file is the thing a human signs off on, and the proof is what makes the signature mean something.

Moving forward, one could have a fully spec'ed and proved ABI encoder/decoder for example, and show that a contract's bytecode has a specific interface.
This can also be applied to other properties that the compiler needs to guarantee.

Reach out if you'd like to contribute or experiment with evm-smith!
