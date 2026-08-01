# NL Specification — Coherence Tracker

This document lists all known inconsistencies, errors, omissions, and gaps across the NL specification
documents (`specs.md`, `stdlib.md`, `compiler.md`, `vm.md`, `optimizations.md`, `milestones.md`, `tests.md`, `README.md`).
Each item has a checkbox to track resolution.

Audit performed on **2026-03-03**, against spec version **0.8.1**.

**Resolved items archived:**

- [archives/coherence_closed_20260714.md](archives/coherence_closed_20260714.md) — final batch (27 items: II, IV, VI, VIII; resolved in versions 0.8.37–0.8.43, 2026-07-14)
- [archives/coherence_closed_20260305.md](archives/coherence_closed_20260305.md) — second pass (49 items, 2026-03-05)
- [archives/coherence_closed_20260303.md](archives/coherence_closed_20260303.md) — first pass (20 items, all resolved in versions 0.3.1–0.8.1)

---

## Summary (open items)

| Category | Open | Description |
|----------|------|-------------|
| — | 0 | All items from the 2026-03-03 audit are resolved. |

**Total open: 0**

Every question opened by the audit has been decided and folded into the specification (see the archives above for
each item and its resolution). Remaining *forward-looking* work is tracked elsewhere:

- **Planned language features** — [specs.md § Planned](../docs/specs.md#planned): `char` type, RAII /
  try-with-resources, `Parsable<T>` interface, subscript operator overloading.
- **Open security recommendations** (design work for future versions, not spec gaps) —
  [security-audit.md](security-audit.md): configurable resource limits (SEC-04), RAII for resources (SEC-12),
  SSRF documentation (SEC-13), secure string handling (SEC-14), duplicate detection at VM link time (SEC-19),
  destructor exception logging (SEC-21), `system.crypto` namespace (SEC-24), capability sandboxing (SEC-25),
  hardening of examples (SEC-26).

A new audit should be performed before tagging **0.9 / 1.0** to re-verify cross-document coherence after the
0.8.41–0.8.43 changes.

---

## Items raised after the 2026-03-03 audit

Coherence issues reported outside the audit (typically as GitHub issues) are listed here with their resolution.

- [x] **P-1 — Frame-eliding optimizations vs. the observability contract** *(fixed 0.8.49)*.
  `optimizations.md` permitted inlining and tail call optimization while listing stack traces as observable, and
  TCO additionally changes whether `StackOverflowException` is thrown at all — two transformations principle 1
  then forbade. Resolved by narrowing observability rather than restricting the optimizations: the number and
  identity of `stackTrace` frames are implementation-defined (with three invariants that hold at every
  optimization level — non-empty, innermost-first, no fabricated frames), and the TCO/`StackOverflowException`
  divergence is stated explicitly as the only sanctioned difference in which exceptions are thrown.
  Reported in [nlvm-specs#1](https://github.com/nlvm-lang/nlvm-specs/issues/1); unblocks the inlining and
  tail-call checkboxes of [nlvm#18](https://github.com/nlvm-lang/nlvm/issues/18).
