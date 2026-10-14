<!-- solve-problem:7c7db2a7-2538-44f0-8c2b-284ec9ba1319 -->
# Cross-client timestamp-defense validation

A [Tangle Solve problem](https://solve.tangle.tools/problems/7c7db2a7-2538-44f0-8c2b-284ec9ba1319).

## What is the problem?

In Bitcoin-style proof-of-work blockchains, a block's header contains a miner-supplied timestamp used by the difficulty-adjustment algorithm (DAA). Clients accept blocks whose timestamps satisfy validity rules; historically Bitcoin used a median-of-past-11 rule, Ethereum used "parent timestamp + 1" with future-block-gassing semantics. Yaish, Stern and Zohar (2023) showed an attacker can craft timestamp histories that manipulate a descendant chain's DAA to lower difficulty and win races ("Uncle Maker" attack on Ethereum PoW). The open problem: **validate that timestamp-based defenses (e.g., median-past-time / MPT rules, as adopted in Ethereum Classic's ECIP-109 and similar mitigations) are actually implemented consistently and correctly across independently maintained clients and descendant protocols.**

Formal setting: Let each client C ∈ {Geth, Nethermind, Besu, Erigon, core-geth, ...} implement a validity predicate V_C(header, chain) and difficulty function D_C(header, chain). Soundness requires: (a) all clients agree on acceptance/rejection of every header sequence (consistency), and (b) the incentive analysis assumed by the mitigation (e.g., no profitable timestamp-manipulation strategy under MPT) holds under the *implemented* arithmetic, including integer rounding, off-by-one boundary cases, and behavior across reorganizations where the "past median" window itself changes. The task is to find — via differential testing, formal refinement verification, or both — inputs (header sequences with adversarial timestamps, equal-timestamp edges, deep reorgs) where V_C or D_C diverges across clients or from the specification.

Note ambiguity: "validate" is not a single theorem; it is an engineering/verification target — either a demonstrated counterexample (divergence) or a machine-checked proof of conformance for given clients.

## Why does it matter?

A single acceptance divergence between two clients forks the chain — a consensus bug exploitable for money (double-spends, difficulty manipulation, uncle-reward theft as in Uncle Maker). Descendant PoW chains (ETC and smaller forks) inherit patched-but-under-audited code where a rounding difference is enough to break them. Sound incentive analyses in the literature (Garay et al.'s Bitcoin backbone proofs) assume an idealized timestamp rule; this problem is the bridge between those proofs and deployed code. Concrete payoff: a reusable differential-test harness and any discovered divergences, which are immediately reportable/fundable bugs.

## What's already known?

- **Uncle Maker (Yaish, Stern, Zohar, CCS/CCS-BC 2023):** demonstrated a profitable timestamp-manipulation attack on Ethereum PoW exploiting the "+1 second" rule and uncle mechanics; proposed MPT-style mitigation (require block timestamp > median of last N block timestamps). This motivated EIP/ECIP-level fixes.
- **Garay, Kiayias, Leonardos (2017, ePrint 2016/1048):** the Bitcoin backbone model — proofs of chain growth/common prefix under idealized assumptions; the reference point for "sound incentive argument" that implementations must not undermine.
- **Known landscape:** Differential testing of consensus logic exists (e.g., consensus-spec tests for Ethereum 2.0, fork-detection efforts after the 2016 ETC/ETH SPV-DOS-era splits and Geth/Parity consensus bugs), but no systematic, adversarially-driven differential suite specifically targeting timestamp/DAA logic across the maintained descendants of the Ethereum PoW clients. Refinement verification of timestamp rules in these codebases has not been published (verify).

## What would count as solving it?

1. **Counterexample outcome:** a concrete header sequence (with timestamps, difficulty values, and a reorg schedule) on which two maintained clients disagree on validity or computed difficulty, reproducible against specified versions — a real, demonstrable bug.
2. **Clean outcome:** a differential fuzzing harness plus a specification of the MPT/DAA rule precise enough (including rounding and reorg semantics) that N clients pass M billions of generated cases with zero divergence, with the harness and corpus released.
3. **Strongest outcome:** machine-checked refinement proof that a client implementation (or an extracted model) implements the specified timestamp/difficulty predicate exactly.
Any of the three, scoped to at least two clients and one descendant chain (e.g., ETC), would be accepted by a referee as validation work.

## How to start

1. Read Yaish–Stern–Zohar §"countermeasures" (their MPT proposal and its game-theoretic argument) and Garay et al. §3 (backline model) to fix the intended semantics you're testing against.
2. Write a single executable specification of the timestamp validity + DAA arithmetic per target chain (Ethereum PoW legacy, ETC with MPT/ECIP-109), pinning down integer types, division rounding, and how the median window is rebuilt after a reorg.
3. Build a differential harness: generate header sequences from a grammar biased toward boundary conditions (equal timestamps, minimal increments, deep reorgs around window boundaries), replay against client header-validation entry points (or extracted pure functions) and diff.
4. Main obstacle: client validation code is entangled with I/O and state; you may need to extract the timestamp/DAA logic into testable form without changing semantics — every extraction risks hiding the bug you're hunting.

## Key references

- Giora Yaish, Aviv Zohar, Uriel Stern? — correct authors: **G. Yaish, A. Zohar** et al. — verify author order — *Uncle Maker: (Time)Stamping Out The Competition in Ethereum*, ACM CCS 2023, DOI:10.1145/3576915.3616674. Demonstrates profitable timestamp manipulation on Ethereum PoW and proposes the MPT mitigation this problem validates. https://doi.org/10.1145/3576915.3616674
- **J. Garay, A. Kiayias, N. Leonardos**, *The Bitcoin Backbone Protocol: Analysis and Applications*, IACR ePrint 2016/1048. Proves chain growth/common prefix under idealized assumptions — the incentive soundness that implementation bugs can defeat. https://eprint.iacr.org/2016/1048
- Ethereum Classic ECIP-109 (MPT timestamp rule) — the deployed mitigation descendant chains adopted; primary spec to test against. (verify current status/number)

## Warnings

- "Ethereum" here means the pre-Merge PoW chain and descendants (ETC, etc.); post-Merge Ethereum is PoS and lacks this DAA — don't waste effort there.
- Divergences only matter on chains where at least two independent clients are actually in use; check client/chain usage stats before targeting.
- Watch rounding semantics (integer division, unsigned underflow in difficulty formulas) and the median definition for even window sizes — historically where such bugs live.
- Uncle Maker has both a paper version and an extended version; cite carefully and don't conflate the original attack with the mitigation analysis.

---
*`BCGT-000248` · flagged for review*

## Acceptance guidance

1. **Counterexample outcome:** a concrete header sequence (with timestamps, difficulty values, and a reorg schedule) on which two maintained clients disagree on validity or computed difficulty, reproducible against specified versions — a real, demonstrable bug.
2. **Clean outcome:** a differential fuzzing harness plus a specification of the MPT/DAA rule precise enough (including rounding and reorg semantics) that N clients pass M billions of generated cases with zero divergence, with the harness and corpus released.
3. **Strongest outcome:** machine-checked refinement proof that a client implementation (or an extracted model) implements the specified timestamp/difficulty predicate exactly.
Any of the three, scoped to at least two clients and one descendant chain (e.g., ETC), would be accepted by a referee as validation work.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/7c7db2a7-2538-44f0-8c2b-284ec9ba1319?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000248.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000248)
