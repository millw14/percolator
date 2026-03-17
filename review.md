# Review of ak-adl branch (A/K side index proposal)

Read through percolator.rs (~3.3k LOC), i128.rs, audit.md, and the current tests.

This review focuses on the proposed A/K lazy side index mechanism vs the current H-only model.

---

## Summary

Current implementation uses:

- H (haircut ratio) for exit fairness
- Per-account liquidation / write-off / force-realize for deficit clearing
- O(1) aggregates for pnl_pos_tot and c_tot
- bounded crank loops for liveness

The ak-adl proposal adds:

- A (position scaling index per side)
- K (PnL index per side)
- epoch state machine
- DrainOnly / ResetPending / Normal phases

Goal is to make deficit socialization O(1) and remove ordering dependence.

The design is sound, but touches more of the engine than the README suggests.

---

## What works well today

Haircut ratio is clean and proven.

`pub fn haircut_ratio(&self) -> (u128, u128)`

Uses only aggregates, O(1), well-covered by Kani.

Force-realize / write-off pipeline is ugly but bounded:

- FORCE_REALIZE_BUDGET_PER_CRANK
- LIQ_BUDGET_PER_CRANK
- ACCOUNTS_PER_CRANK

This keeps the engine live under load.

Weakness: fairness depends on account index order.

---

## Why A/K makes sense

A/K fixes two real problems:

1. Order-dependent force realize
2. O(N) deficit clearing
3. Partial ADL state risk
4. Long tail of zombie positions

Scaling positions via A and settling via K removes per-account loops.

This matches the intended constraint-engine design.

H + A/K composition is clean:

- A/K handles position fairness
- H handles exit fairness

No overlap.

---

## Real scope of the change

This is not just bankruptcy logic.

Every place reading position_size must change.

Affected paths:

- mark_pnl_for_position
- settle_mark_to_oracle
- settle_account_funding
- account_equity_mtm_at_oracle
- liquidation checks
- trade execution OI updates
- LP aggregate tracking
- funding settlement
- ADL / force-realize removal

Expect ~15–25 call sites.

Safer to add effective_pos() helper and route everything through that.

---

## Precision risk

Repeated A scaling will reduce precision.

Small positions will round to zero earlier.

Need proofs:

- error bounded per account
- no side imbalance
- total OI monotonic

DrainOnly helps but not enough alone.

---

## Funding interaction

Current code:

funding uses position_size directly.

With A/K:

either

1) fold funding into K (cleaner)
2) compute effective_pos every settlement (slower)

README implies funding → K.

That changes rounding behavior.

Needs new conservation proofs.

---

## Epoch transition cost

ResetPending requires settling old accounts.

At 4096 accounts + bounded crank, transition could take multiple cranks.

Must compose with existing scan budget.

Otherwise state machine stalls.

---

## Account size impact

Per account additions:

- basis_i
- a_basis_i
- k_snap_i
- epoch_i

~56 bytes per account.

Total still under Solana limits but worth tracking.

---

## Proof impact

All existing proofs assume direct position_size.

Need new invariants:

- A monotonic
- K monotonic
- epoch correctness
- conservation with K settlement
- no teleport under scaling

This is the largest cost of the change.

---

## Suggested structure

Add per-side state:

```rust
struct SideState {
    a: U128,
    k: I128,
    epoch: u64,
    total_basis: U128,
    phase: SidePhase,
}
```

Add effective_pos()

Replace direct reads.

Add settle_k()

Keep funding as-is first, then fold later.

Do not refactor everything in one commit.

---

## Verdict

The proposal fixes real fairness issues.

The refactor is large but contained.

Core accounting stays intact.

Worth doing if goal is production-grade risk engine.
