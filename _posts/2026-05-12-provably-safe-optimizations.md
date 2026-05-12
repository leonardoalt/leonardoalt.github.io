---
layout: post
title: "Provably safe EVM optimizations that compilers won't do"
---

*Special thanks to Martin Lundfall for discussions about equivalence proofs.*

> _tl;dr_: write crazy gas optimizations, prove them safe in Lean, profit.

In the [previous post](https://leonardoalt.github.io/evm-smith), we showed
that AI can write EVM assembly directly and write proofs about it in
[Lean](https://lean-lang.org/).

In this post we are particularly interested in optimizations:
- what kind of optimizations exist when we remove the compiler?
- can we prove them safe?

As a starting point, I asked Claude to write simple but real optimizations to
the WETH contract from the previous post. Three were added, each with **a
Lean theorem proving equivalence** to the original 86-byte runtime:

- **`PUSH1 0` → `PUSH0`** (EIP-3855), 9 sites in WETH's runtime, 86 → 77
  bytes. **Equivalence proved** in
  [`Weth/OptimizedProgram.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/Weth/OptimizedProgram.lean)
  ([commit](https://github.com/leonardoalt/evm-smith/commit/268971c)).
- **`DUP1`/`SWAP1` → `CALLER` twice + drop the dead `POP`**: rebuilding the
  sender via a second `CALLER` (BASE, 2 gas) is cheaper than holding it on
  stack via `DUP1`/`SWAP1` (VERYLOW, 3 gas each). 77 → 74 bytes.
  **Equivalence proved** in
  [`Weth/OptimizedProgramV2.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/Weth/OptimizedProgramV2.lean)
  ([commit](https://github.com/leonardoalt/evm-smith/commit/a5bb0eb)).
- **1-byte revert-tail peephole**. **Equivalence proved**
  ([commit](https://github.com/leonardoalt/evm-smith/commit/025bcb3)).

Foundry tests confirm each optimization saves gas
([`Weth/foundry/test/WethGasCompare.t.sol`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/Weth/foundry/test/WethGasCompare.t.sol)).

The Lean equivalence theorems carry the original's solvency property over
to each optimized version automatically, no re-proving needed!

This leads to an interesting insight: we might end up with 3 versions of a
contract (or any software):
- Most human-readable: written as spec or HLL code. Humans read and understand it.
- Most provable: has the best shape/info for Lean proofs, AIs write the proofs.
- Most optimized: best gas usage, used for actual deployment.

As long as we can prove equivalence of all versions, we get the best of all
worlds. The good news: **if one version has proven properties, it is usually
much easier to prove that another version is equivalent to it than re-proving
the properties**.

To push this idea further, we look at the classic ERC20 contract.
Both for its impact and for a particular optimization I'm
interested in: custom storage layouts for gas performance.

When you write a Solidity contract with `mapping(address => uint256)
balances`, the compiler turns every read like `balances[msg.sender]`
into a `keccak256(slot ++ key)` lookup. Solady's ERC-20 tightens this
with a compact 32-byte preimage written in inline assembly:

```
mstore(0x0c, BALANCE_SLOT_SEED)
mstore(0x00, msg.sender)
sload(keccak256(0x0c, 0x20))
```

Vyper, and plain Solidity without Solady's trick, emit the more
straightforward 64-byte preimage version for `self.balanceOf[msg.sender]`:

```
mstore(0x00, BALANCE_SLOT_ID)
mstore(0x20, msg.sender)
sload(keccak256(0x00, 0x40))
```

In both cases: two `MSTORE`s, one `KECCAK256`, one `SLOAD`. ~42 gas
and a dozen bytes of bytecode per access; `transfer` does it four
times.

Why does the compiler hash? Storage is a single flat map from `uint256` to
`uint256`. If `balances[A]` happened to live at the same slot as
`allowances[B][C]`, writing one would corrupt the other.  Hashing each `(map,
key)` pair gives each entry a slot that, under Keccak's preimage resistance,
doesn't collide with any other map's entries. It's a sensible uniform scheme
the compiler can apply without per-contract analysis. However, this is not the
*only* way to get collision-free maps. A contract author can also hand-pick
slots and prove no collisions occur, and that is what we are exploring here
in an automated and provable way.

## The optimization, sketched

An EVM address is already a 160-bit injective key. For the balance
map alone, we don't need to hash anything: we can just put
`balances[addr]` at slot `addr` directly, skipping both `MSTORE`s
and the `KECCAK256`.

We can't reuse this trick for the other maps. Allowances are keyed by `(owner,
spender)`, two addresses; a single slot can't encode that pair injectively
without hashing. Nonces and other one-key maps *could* in principle, but we
keep it simple by optimizing only the balances map.

## Gas savings

Solidity / Solady, after the optimization:

| Operation              | Solady (keccak-balance) | Optimized (`~addr`-as-slot) | Delta  |
|------------------------|-------------------------|------------------------------|--------|
| `balanceOf` (warm)     |                 1,117   |                       1,067  |  -50   |
| `mint`                 |                 3,365   |                       3,322  |  -43   |
| `burn`                 |                 3,304   |                       3,252  |  -52   |
| `transfer` (warm-to)   |                 3,477   |                       3,402  |  -75   |
| `transferFrom`         |                25,906   |                      25,833  |  -73   |

Plus ~42 bytes off the deployed runtime. The Vyper deltas are
roughly double: Vyper recomputes the slot keccak at every access,
with no compiler-side reuse, so the savings stack up to -192 gas
per `transfer`.

A small win on this one transformation. The interesting claim is that **the
same proof scaffolding lets you stack many such per-contract optimizations**,
each tailored to what the deployed contract actually does, none of which a
general-purpose compiler can sensibly apply across the board. Combined across a
full contract, the savings can become meaningful. Every transformation carries
its safety property through to the deployed bytes, provided the proof framework
keeps up.

## The optimization, in code

Take a battle-tested ERC-20: Solady's, in our case. Its `balanceOf`
reads, written in inline assembly:

```solidity
function balanceOf(address owner) public view returns (uint256 result) {
    assembly {
        mstore(0x0c, _BALANCE_SLOT_SEED)      // 0x87a211a2
        mstore(0x00, owner)
        result := sload(keccak256(0x0c, 0x20))
    }
}
```

We override it with:

```solidity
function balanceOf(address owner) public view returns (uint256 result) {
    assembly {
        result := sload(not(shr(96, shl(96, owner))))   // ~addr as slot
    }
}
```

Same for `transfer`, `transferFrom`, `_mint`, `_burn`, `_transfer`.
Allowances stay keccak-mapped (two keys, can't pack injectively into
one slot). Nonces could in principle be optimized too but stay
keccak-mapped here for simplicity of the experiment.

You may have noticed the `not(...)` step on the new version. See the next
section for why.

## Why it's safe (and the bug we caught along the way)

Slot collisions are the only failure mode for any deterministic storage-layout
change. The first version of this optimization used `sload(addr)`: the address
directly as the slot, which is the obvious thing to do. That was wrong.

Solidity and Vyper automatically allocate state variables to *low* storage
slots: slot 0 for the first declared variable, slot 1 for the second, and so on
(also depending on storage packing possibilities). Our `MockERC20Optimized`
declares `string _name`, `string _symbol`, `uint8 _decimals` which get slots 0,
1, 2.

If `balance[addr]` lives at slot `uint256(addr)`, then `mint(address(0x01), v)`
*overwrites the contract's symbol*. The contract's `symbol()` getter starts
returning garbage. If the mint amount has the right low byte, the next call to
`name()` panics with `storage byte array incorrectly encoded`.

The Vyper fuzz test caught this immediately.  The Solidity fuzz test happened
to miss it because (a) our fuzz cases don't read the metadata getters after a
fuzzed mint, and (b) the random amounts didn't trip a visible getter panic. The
collision was silent: slot was overwritten, but the test asserted only
`balanceOf(addr) == v` (trivially true after writing `v` to slot `addr`).

**The fix is one byte: `sload(not(addr))` instead of `sload(addr)`.**

`not(addr)` for any 160-bit address has its high 96 bits all-one,
so the slot lies in `[2^256 - 2^160, 2^256)`. Every state-variable
slot the compiler assigns (small numbers: 0, 1, 2, ...) sits well
below 2^160, so no collision there.

The allowance and nonce slots are full 256-bit Keccak outputs, so
one of them *could* land in our high band. The probability of any
single keccak output landing in that band is `2^-96` per slot.
That's the same Keccak-distribution assumption every Solidity
`mapping` already rests on to keep its own slots from colliding
with each other. Same trust model.

`NOT` is bijective on `UInt256`, so distinct addresses map to
distinct slots and no two users alias.

After the fix, both backends pass an extended test suite that
mints to addresses 0x00 through 0x03 and asserts the metadata
getters survive, the regression test that would have caught the
original Solidity bug from the start.

The lesson generalises: any "use this identity-shaped value as a storage slot"
optimization has to also *prove* its slot function doesn't intersect any other
use of storage. `NOT` is the cheapest way to lift the slot into a region
nothing else can reach in this specific case.

## The no-aliasing invariant: cheap defence

Beyond the specific NOT fix, both backends ship a generic Foundry **fuzz
invariant** that catches storage aliasing, including collisions we didn't
know to look for:

```solidity
function invariant_no_phantom_balance() public view {
    uint256 sum;
    for (uint256 i = 0; i < handler.actorCount(); i++) {
        sum += token.balanceOf(handler.actors(i));
    }
    assertLe(sum, token.totalSupply(),
             "phantom balance: aliased addresses inflate the visible sum");
}
```

A handler drives the contract through fuzzed `mint`/`transfer` sequences over a
small actor dictionary that always includes low addresses, exactly the known
collision zone. If two distinct actors share one storage slot, both read its
value as "their" balance, and the sum exceeds `totalSupply`.

Sanity-checked by temporarily reverting the `NOT(addr)` fix and
re-running. The invariant fires immediately:

```
[FAIL: invariant_no_phantom_balance]
  phantom balance: aliased addresses inflate the visible sum:
  76318471673650654452077537475063650456547488489235383969275586155053664174114 > 0
```

That huge number is the corrupted `_name` string's bytes read as
a uint256 balance. After restoring NOT, the invariant passes
across 2,048 fuzzed calls on 2 backends.

Fuzzing is the cheap operational backstop during development: 30 lines of
Solidity, every CI cycle, catches aliasing collisions we didn't anticipate in
our human spec! The proper answer is in the next sections: the
`SlotAbstraction` refinement framework makes a slot-aliasing optimization
literally unprovable at the type level.

## Proving it at the peephole level

Call the two storage layouts **K** (keccak-mapped, Solidity/Solady's
default) and **N** (`~addr`-as-slot, the new one). The Lean side
proves equivalence between two short `Program` values: K's keccak
prefix and N's `[NOT, SLOAD]`. Bytecode level, sorry-free, no new
axioms.

### What is proven

1. `UInt256.lnot` is injective.
2. For any 160-bit `a`, `~a ∉ {0, 1, 2, _TOTAL_SUPPLY_SLOT}`. Pure
   arithmetic.
3. K's 10-opcode keccak prefix leaves `balanceSlot a` on the stack.
4. N's 2-opcode prefix leaves `sload(~a)` on the stack.
5. Under a per-address storage relation linking K's slot to N's,
   the two prefixes produce equal stack-tops.
6. The two store blocks land at structurally-known post-states.
7. An abstract `Function.update`-style refinement preserves a
   balance map across mint / burn / transfer, provided you supply
   a `SlotAbstraction` (which bundles injectivity and named-slot
   disjointness).

### To be proven

A Lean theorem about the full `solc`/`vyper`-compiled bytecode, rather than
just the 10-opcode prefix. As we learned from the previous post, this is doable
with the scaffolding from the WETH case, and left out of scope for this post.

### How the proof works

The point of leverage: **we never need to compute Keccak**. We
define K's slot as exactly the Lean term the bytecode produces:

```lean
def balanceSlot (a : UInt256) : UInt256 := ffi.KEC (preimage a)
```

`ffi.KEC` is `opaque`, so the term is irreducible. Lean carries
it through the proof symbolically.

The relation between K-state `σ_K` and N-state `σ_N` says:

> For every address `a`,
> `σ_K[balanceSlot a] = σ_N[~a]`.

After `transfer(from, to, x)` the K-side writes the new balance
at `balanceSlot from`; the N-side writes the same value at
`~from`. The relation still holds because both sides write the
same value at the slot the relation pairs them by. We never had
to know what `balanceSlot from` *is* as a number, only that
addresses map to slots consistently. Automatic, because
`balanceSlot` is a function.

`#print axioms balanceLoad_observable_equiv` gives Lean's three
standard foundations (`propext`, `Classical.choice`, `Quot.sound`)
and nothing else. The trust story has two layers:

- `~addr` doesn't collide with `{0, 1, 2, _TOTAL_SUPPLY_SLOT}`:
  proved in Lean, arithmetic.
- `~addr` doesn't collide with the keccak-derived allowance and
  nonce slots: trusted via Keccak's preimage resistance, the same
  axiom every Solidity mapping already needs.

Same trust model as the compilers.

## Making the soundness obligations type-level: the refinement framework

The peephole theorem above takes a per-address storage relation as
a *hypothesis*. It doesn't check whether you can actually construct
that relation across a sequence of mint/transfer/burn operations.
With a bad slot function (non-injective, or colliding with named
state), the relation would silently become degenerate and the
peephole proof would still go through.

That's the gap the collision bug walked through, and we close it
at the structure level. The refinement framework in `Spec.lean`
defines:

```lean
structure SlotAbstraction where
  ValidAddr : UInt256 → Prop          -- what counts as an address
  NamedSlot : UInt256 → Prop          -- slots used for non-balance state
  slotFn    : UInt256 → UInt256       -- the slot derivation
  inj       : ∀ a b, ValidAddr a → ValidAddr b →
              slotFn a = slotFn b → a = b
  disjoint  : ∀ a, ValidAddr a → ¬ NamedSlot (slotFn a)
```

You cannot construct a `SlotAbstraction` without supplying proofs
of both `inj` (distinct addresses → distinct slots; no user
aliasing) and `disjoint` (no slot lands on a named state slot; no
metadata corruption). The three preservation theorems
(`mint_refines`, `burn_refines`, `transfer_refines`) only apply
when you hand them a `SlotAbstraction`. So:

- A **non-injective** slot function (e.g., `addr mod 2^32`) fails
  to prove `inj`. The structure can't be constructed. The
  preservation theorems can't be applied. The optimization is
  type-rejected.
- A **named-slot-colliding** slot function (e.g., `id`, which gives
  `slotFn 0 = 0 ∈ {0, 1, 2, _TOTAL_SUPPLY_SLOT}`) fails to prove
  `disjoint`. Same rejection.

The actual `lnotSlotAbstraction` instance discharges both fully,
sorry-free. The pre-fix `idSlotAbstraction` would fail `disjoint`
and *cannot be created*.

### What an AI (or human) has to do to ship a new optimization

The recipe:

1. Pick a slot function `f`.
2. Define `ValidAddr` (typically `a.toNat < 2^160` for real
   addresses).
3. Define `NamedSlot`: what slots your contract uses for
   non-balance state.
4. Prove `inj`: distinct valid addresses give distinct slots.
5. Prove `disjoint`: `f`'s image (restricted to valid addresses)
   avoids every named slot.
6. Construct the `SlotAbstraction`.
7. Apply the preservation theorems to your operations.

If steps 4-6 fail, `f` is bad. The proof obligation **is** the
safety obligation. The framework refuses to admit a `SlotAbstraction`
whose safety properties don't hold.

This is how type-theory is used to "make the bad design unprovable." The AI
synthesising the optimization can't sneak past the obligations the way a human
reviewer might miss them in code review, of course as long as someone is
actually checking.

## Same trick, different compiler: Vyper

Solidity and Vyper are different source languages but both compile to
EVM bytecode. Both compile `mapping(address => uint256)` (or
Vyper's `HashMap[address, uint256]`) into a keccak-derived slot
lookup. So the same optimization should work on Vyper-compiled
contracts too.

We tried it against [Snekmate](https://github.com/pcaversaccio/snekmate),
the Solady-equivalent Vyper library. Two differences this time:

**The optimization is applied at the bytecode level, not the source
level.** Vyper has no inline assembly, so we can't rewrite the slot
derivation in the source. Instead, we compile Snekmate's ERC-20
unmodified, find every site that matches the keccak-prefix pattern,
and patch them in place. The patcher is length-preserving (so jumpdest
addresses stay valid) and fail-closed (the patched-site count must
match the source-side `self.balanceOf[…]` count, or the patcher
refuses to emit). 6,602 bytes of runtime, 9 balance-access sites
patched.

**The Vyper fuzz test caught the same collision the Solidity-side
version had silently been carrying** (see "Why it's safe" above). Vyper also
places its named state at low slots: `ownable.owner` at slot 0x01,
`totalSupply` at 0x04.  A user with address `0x01` collides with the owner
field.  Foundry's fuzz dictionary tries `address(0x01)` early in any
`testFuzz*` run, so the bug showed up on the first run: `mint(0x01, value)`
overwrote the contract's owner with `(owner address + value)`.

We applied the same `NOT(addr)` fix to the Solidity side and added
metadata-preservation regression tests that would have caught it
from the start.

The patched 15-byte sequence becomes:

```
JUMPDEST x 10       ; padding (length-preserving)
PUSH1 <P>           ; memory offset where the address argument sits
MLOAD               ; addr := mem[P]
NOT                 ; slot := ~addr
SLOAD | SSTORE
```

Gas table (Vyper):

| Operation              | Original (gas) | Patched (gas) | Δ       |
|------------------------|---------------:|--------------:|--------:|
| `balanceOf` (warm)     |            920 |           872 |  -48    |
| `mint` (warm)          |          3,510 |         3,414 |  -96    |
| `burn`                 |          3,216 |         3,114 | -102    |
| `transfer` (warm)      |          3,606 |         3,414 | -192    |
| `transferFrom`         |         28,137 |        27,945 | -192    |

Bigger savings than Solidity-via-`solc` because Vyper recomputes the
balance-slot keccak at every access (no compiler-side reuse). Each
keccak saved is ~48 gas; transfer does 4 balance accesses, hence
-192.

The Vyper-side proofs in
[`EquivalenceVyper.lean`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/ERC20/EquivalenceVyper.lean)
work the same way as the Solidity ones: K's keccak result is
carried through Lean symbolically (we never reduce it), and the
`NOT` substitution plugs in unchanged. Same axiom story: Lean's
three standard foundations, no new evm-smith axioms, no sorries.

## Why tho 

Auditors typically flag micro-optimizations like this as too clever for
humans to attempt. The bug surface is too big: a wrong
slot-derivation in one of seven places and the whole token corrupts itself
silently. So contract authors stick with `mapping(address => uint256)` and
users pay more gas.

AI-with-proofs enables more optimizations. If an AI can write
the optimized version *and* a machine-checkable proof that it's
equivalent to the canonical implementation, the cleverness becomes
free. The proof is the audit. The optimization is also small enough
that an AI can come up with it: the peephole is local, the soundness
condition is one storage relation, and the proof obligations
decompose cleanly per block.

This demo is a small first step in that direction. The pattern generalizes:
anywhere you have a Solidity mapping whose key is already injective, you can
drop the keccak prefix. We did it for one bucket of one contract. Many others
(registries, balances in single-asset vaults, owner-of in NFTs, simple state
machines indexed by address) follow the same shape. There are certainly more
cases where smart contract compilers sensibly inject generic patterns that
could be optimized in specific cases. Now we can optimize it **and** prove it
safe, without the compiler.

## How Claude did

Everything I asked for was one-shot and the whole experiment took less than 2h.

## How the demo is wired up

Both the Solidity and Vyper sides are stand-alone Foundry projects
under `EvmSmith/Demos/`. Each pins its compiler version and its
upstream library as a git submodule. The Lean proofs live one
level up under `EvmSmith/Demos/ERC20/*.lean` and reuse the
existing evm-smith / EVMYulLean toolchain.

**Solidity / Solady side** (`EvmSmith/Demos/ERC20/foundry/`):

- `solc` 0.8.20, optimizer on with 200 runs, no `via_ir`. Matches
  Solady's own CI configuration.
- `lib/solady` is a git submodule pinned to a known commit; the
  storage layout, base contract, and allowance/nonce behaviour all
  flow from that one pin.
- Two contracts: `MockERC20Original.sol` inherits Solady unchanged;
  `MockERC20Optimized.sol` overrides the six `virtual` functions
  that touch the balance map. Both are deployed with the same
  metadata so the events match.

**Vyper / Snekmate side** (`EvmSmith/Demos/ERC20-Vyper/`):

- Vyper 0.5.0a1, installed in a local `python -m venv` so the
  toolchain is self-contained. The pin matches Snekmate's
  `# pragma version ~=0.5.0a1`.
- `optimize = "gas"` matches Snekmate's CI. We verified across all
  three optimizer levels (`none`, `codesize`, `gas`) that the
  balance-keccak prefix is emitted at every access site regardless.
  Vyper doesn't seem to do common-subexpression elimination on storage-slot
  derivation.
- `lib/snekmate` is a git submodule pinned to [commit `ba43b69`](https://github.com/pcaversaccio/snekmate/commit/ba43b69).
  A relative symlink `src/snekmate → ../lib/snekmate/src/snekmate`
  lets Vyper's import resolver find the package.
- One contract: `mock.vy` initializes ownable + erc20 modules. The
  *optimization* is delivered as a bytecode patcher
  (`script/patch.py` offline, `test/BytecodePatcher.sol` in-test)
  applied length-preservingly to the compiled runtime, then
  `vm.etch`'d on top of a constructor-initialised deployment.

**Lean side** (no Vyper/Solidity toolchain needed):

```bash
lake build EvmSmith.Demos.ERC20.Equivalence       # Solidity peephole
lake build EvmSmith.Demos.ERC20.EquivalenceVyper  # Vyper peephole
lake build EvmSmith.Demos.ERC20.AxiomCheck        # #print axioms on all theorems
```

## Try it yourself

```bash
# Proofs (Lean 4 + EVMYulLean):
lake build EvmSmith.Demos.ERC20.Equivalence
lake build EvmSmith.Demos.ERC20.EquivalenceVyper
lake build EvmSmith.Demos.ERC20.AxiomCheck

# Solidity tests + gas comparison:
cd EvmSmith/Demos/ERC20/foundry
forge test
forge test --match-contract ERC20GasCompareTest -vv

# Vyper tests + gas comparison (needs the venv):
cd EvmSmith/Demos/ERC20-Vyper
python3 -m venv .venv
.venv/bin/pip install vyper==0.5.0a1
cd foundry
PATH=$(pwd)/../.venv/bin:$PATH forge test
PATH=$(pwd)/../.venv/bin:$PATH forge test --match-contract ERC20VyperGasCompareTest -vv
```

Solady's full ERC-20 test surface (mint, burn, transfer,
transferFrom, allowance, fuzz cases, the Permit2 escape hatch) runs
against both backends and passes identically. The proofs in Lean
check the peephole soundness on hand-rolled `Program` mirrors of
the keccak prefix.

You can read the full report at
[`REPORT_ERC20.md`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/ERC20/REPORT_ERC20.md), and the bytecode-level
investigation at [`STORAGE_LAYOUT.md`](https://github.com/leonardoalt/evm-smith/blob/main/EvmSmith/Demos/ERC20/STORAGE_LAYOUT.md).

## Closing

Smart contract compilers use generic pattens that are known to be safe for all
contracts. However, users often bypass compiler choices by using assembly,
which is considered dangerous. Changing the storage layout of a contract purely
for performance would be frowned upon. Now we can prove these optimizations
correct and get the most performance at the same time.

In this specific experiment, we saw how a Solidity-compiled `mapping(address =>
uint256)` pays keccak every single access for a property (slot injectivity)
that the address already gives you for free. We showed that AI can improve it
and prove that the new code is correct.

There's nothing magic about ERC-20 here. Every Solidity/Vyper codebase that
maps an address to a single value is paying the same extra cost. The
generalization is straightforward, and the proof framework is the same. Since
both compile to EVM, the proof obligation underneath was identical.

Putting the previous post and this one together, I am quite bullish on AI
writing optimized EVM assembly, proving it correct, and proving it equivalent
to something a human prefers, in collaboration with humans. Compilers in their current form may give way to simpler spec
languages that combine human-readable intent with formal
statements.

All the code and proofs above are in
[evm-smith](https://github.com/leonardoalt/evm-smith). Contributions of demos
and proofs are welcome!
