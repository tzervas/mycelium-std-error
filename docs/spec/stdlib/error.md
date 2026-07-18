# Spec — `std.error` / `std.option` / `std.result` (errors-as-values ergonomics: propagate, recover, or re-propagate — never drop)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-error` (M-527, #168, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.error` / `std.option` / `std.result` · Ring `2` ([RFC-0016 §4.2](../../rfcs/RFC-0016-Core-Library-and-Standard-Library.md)) · Tier `B` ([RFC-0016 §4.4](../../rfcs/RFC-0016-Core-Library-and-Standard-Library.md)) |
| **Tracks** | `M-527` (#168) — the Phase-5 task this spec delivers: the errors-as-values ergonomics layer (combinators + `?`-style propagation) held to the never-silent floor (I1). |
| **Scope** | The ergonomic combinator surface over the value model's `Option`/`Result`/error sums: the total, pure combinators (`map`/`map_err`/`and_then`/`or_else`/`unwrap_or`/`ok_or`/…), the explicit propagation form (`?`-style), the explicit named partial accessors (`expect`/`unwrap`), and the **bridge** from an error value into RFC-0014 reified recovery. It owns *ergonomics*, not error representation. |
| **Boundary** | OUT of scope: (a) the error *representation* itself — `Option`/`Result`/structured-error values and the guarantee-lattice tags are owned by `core` / prelude ([`core.md`](./core.md), M-515, re-exporting RFC-0001); (b) the reified *recovery policies* + bounded effects — owned by `std.recover` (M-520, the library form of RFC-0014); (c) error *presentation* (rendering/levels/JSON projection) — owned by `std.diag` (M-510, the library form of RFC-0013). This module is the *value-combinator and propagation glue* that sits between them. |
| **Depends on** | RFC-0016 §4.1 (the C1–C6 contract); RFC-0001 (the value model — `Option`/`Result`/structured error sums, the guarantee lattice); [RFC-0014](../../rfcs/RFC-0014-Declarative-Error-Recovery-and-Bounded-Effects.md) §4.1/§4.2 (errors are explicit propagating values; the I1 never-silent and I5 additive/opt-in invariants this module must hold and bridge to). |
| **Grounds on** | the `core`/prelude re-exports of the RFC-0001 value model (KC-3: this module adds **no trusted code** — it is pure combinators over existing value sums, plus a typed handoff into the M-520 `recover` surface). |

---

## 1. Summary

`std.error` is the **errors-as-values ergonomics layer**: the combinators and propagation form that make
working with `Option`/`Result` pleasant *without* ever giving the programmer a way to make an error vanish.
Its **honesty crux** is structural, not a convention: an error is **transformed, recovered (explicitly), or
re-propagated — never dropped** (RFC-0014 §4.2 **I1**; RFC-0016 **C1**; G2). There is no combinator in the
surface that silently discards an `Err`/`None`; the only ops that can *consume* an error without producing a
success are the explicitly-named **partial** accessors (`unwrap`/`expect`), which **refuse loudly** (panic /
abort-with-diagnostic), never a silent default. Recovery is **additive and opt-in** (RFC-0014 **I5**): the
default of every fallible value is to propagate; turning an error into a value happens only through an
explicit, named combinator or the typed bridge into `std.recover`. The module is Ring 2 and adds **no
trusted code** (KC-3) — every op is a pure function over the value sums that `core` already owns.

## 2. Scope & module boundary

- **In scope:** the pure combinator surface over `Option<τ>` and `Result<τ, ε>` —
  `map` / `map_err` / `and_then` / `or_else` / `filter` / `unwrap_or` / `unwrap_or_else` / `ok_or` /
  `ok_or_else` / `transpose` / `flatten` / `zip` / `inspect` / `inspect_err`; the explicit **propagation**
  form (`?`-style, "propagate this `Err`/`None` to my caller"); the explicitly-named **partial** accessors
  (`unwrap` / `expect` / `unwrap_err`); and the **`recover` bridge** — a typed handoff that hands an error
  value to an RFC-0014 reified recovery policy (M-520) and returns its explicit outcome.
- **Out of scope (and who owns it):**
  - The *types* `Option`/`Result`/structured-error + the lattice tags — `core` / prelude ([`core.md`](./core.md), M-515; re-exporting RFC-0001). This module imports them; it does not define them.
  - The reified *recovery actions* (`fallback`/`retry`/`escalate`/`cleanup_then_propagate`), effect budgets, and the `PolicyRef` machinery — `std.recover` (M-520, library form of [RFC-0014](../../rfcs/RFC-0014-Declarative-Error-Recovery-and-Bounded-Effects.md)). This module only *bridges* to it.
  - Error *presentation* (graded levels, dual human/JSON projection) — `std.diag` (M-510, library form of RFC-0013). This module never renders.
- **Ring & layering:** Ring 2 (RFC-0016 §4.2) — written to the §4.1 contract over Ring 0 (`core`'s value
  sums) and bridging to Ring 1 (`recover`). It **re-exports** the value sums from `core` for ergonomics and
  **wraps** them with combinators; it builds **no** new trusted code (KC-3). The `?`-style propagation form
  is **surface sugar that elaborates to a `Match` on the result sum** (the same elaboration RFC-0014 §4.3
  uses for explicit handling sites) — no new L0 node, no kernel change.

## 3. Exported-op surface (design sketch)

A signature sketch — value-semantic, immutable-by-default. Every fallible accessor returns `Option`/`Result`;
the partial accessors are explicitly named and refuse loudly. This fixes the surface and feeds the §4
matrix; it is **not** a committed grammar.

```text
// illustrative signatures (not a committed surface). τ, υ, ε, φ are type vars.

// ---- transform (keep the sum shape; never collapse an error away) ----
map      : Result<τ, ε>, (τ -> υ)            -> Result<υ, ε>     // Ok-side map; Err passes through untouched
map_err  : Result<τ, ε>, (ε -> φ)            -> Result<τ, φ>     // Err-side map; Ok passes through untouched
and_then : Result<τ, ε>, (τ -> Result<υ, ε>) -> Result<υ, ε>    // monadic bind; Err short-circuits (propagates)
or_else  : Result<τ, ε>, (ε -> Result<τ, φ>) -> Result<τ, φ>    // EXPLICIT recovery hook; must return a Result
filter   : Option<τ>,   (τ -> bool)          -> Option<τ>       // Some->None is a *typed transition*, not a drop
inspect    : Result<τ, ε>, (&τ -> ())        -> Result<τ, ε>    // peek Ok; value unchanged (effectful arg, §C6)
inspect_err: Result<τ, ε>, (&ε -> ())        -> Result<τ, ε>    // peek Err; value & propagation unchanged

// ---- convert between Option and Result (no information lost silently) ----
ok_or     : Option<τ>, ε                     -> Result<τ, ε>    // None -> Err(ε): names the absence explicitly
ok_or_else: Option<τ>, (() -> ε)             -> Result<τ, ε>
ok        : Result<τ, ε>                     -> Option<τ>       // Err -> None: see §5/C1 — flagged, NOT a drop
transpose : Option<Result<τ, ε>>             -> Result<Option<τ>, ε>
flatten   : Result<Result<τ, ε>, ε>          -> Result<τ, ε>

// ---- defaulted accessors (recover with an HONEST tag; never fabricate a guarantee) ----
unwrap_or      : Result<τ, ε>, τ             -> τ              // default value: tagged Declared (§I2/VR-5)
unwrap_or_else : Result<τ, ε>, (ε -> τ)      -> τ              // computed default: honest tag from the closure

// ---- EXPLICIT propagation (the default posture) ----
propagate (`?`): Result<τ, ε> -> τ  [propagates Err(ε) to the caller's Result]   // elaborates to a Match

// ---- EXPLICIT, NAMED partial accessors (refuse loudly; never a silent default) ----
unwrap     : Result<τ, ε> -> τ      // PARTIAL: on Err, refuses loudly (abort + diagnostic); never silent
expect     : Result<τ, ε>, msg -> τ // PARTIAL: same, with a caller-supplied reason in the refusal
unwrap_err : Result<τ, ε> -> ε      // PARTIAL: symmetric; on Ok, refuses loudly

// ---- the RFC-0014 bridge (additive, opt-in — I5) ----
recover : Result<τ, ε>, PolicyRef -> RecoverOutcome<τ, ε>
//   hands Err(ε) to a reified RFC-0014 policy (M-520); returns its EXPLICIT outcome:
//   Recovered(τ, honest-tag)  |  Propagated(ε')  -- never a drop (I1). Ok passes through unrecovered.
//   The exact RecoverOutcome shape + PolicyRef API are owned by M-520 — FLAGGED in §7 (Q1).
```

## 4. Guarantee matrix (the load-bearing deliverable — [RFC-0016 §4.5](../../rfcs/RFC-0016-Core-Library-and-Standard-Library.md))

Rows = exported ops. Encoded here as a checked table (the RFC-0003 §4 template); asserted in tests once code
lands — never prose only. **No row permits a silent drop of an error (RFC-0014 I1 / RFC-0016 C1):** every
row either *transforms* the sum (the error survives in the result), *re-propagates* it, *recovers* it
explicitly with an honest tag, or — for the partial accessors — *refuses loudly*. The "Fallibility" column
states each op's explicit outcome set; none of them is "silently returns success".

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `map` | `Exact` | total — `Err` passes through unchanged (error preserved) | none | n/a (pure combinator) |
| `map_err` | `Exact` | total — `Ok` passes through; `Err` transformed (error preserved) | none | n/a |
| `and_then` | `Exact` | total — `Err` short-circuits and **propagates** (never dropped) | none | n/a |
| `or_else` | `Exact` | total — **explicit recovery hook**; must yield a `Result` (recover or re-propagate, never a drop) | none | n/a |
| `filter` | `Exact` | total — `Some→None` is a **typed transition** (named absence), not a silent loss | none | n/a |
| `ok_or` / `ok_or_else` | `Exact` | total — `None` becomes an **explicit** `Err(ε)` (absence is named) | none | n/a |
| `ok` (`Result→Option`) | `Exact` | total — `Err→None`; the discarded `ε` is a **flagged, lossy conversion** (see C1/§5), never an unflagged drop | none | yes (lossy-conversion note in EXPLAIN) |
| `unwrap_or` / `unwrap_or_else` | `Declared` (the substituted default — VR-5 / RFC-0014 **I2**) | total — recovers with an **honestly-tagged** default; never upgrades a guarantee | none (`_or_else` closure may declare its own) | yes (the substitution is recorded) |
| `?`-propagate | `Exact` | total at this site — **propagates** `Err`/`None` to the caller (the default posture); elaborates to a `Match` (RFC-0014 §4.3) | none | yes (the propagation is an ordinary, inspectable `Match`) |
| `unwrap` / `expect` / `unwrap_err` | `Exact` (when it returns) | **PARTIAL** — on the wrong variant it **refuses loudly** (abort + diagnostic), never a silent default | none | yes (the refusal carries a diagnostic record) |
| `recover` (RFC-0014 bridge) | inherits the policy's honest tag (`Declared` for a `fallback`; `Empirical`/`Proven` only with a checked basis — VR-5 / I2) | total — yields `Recovered(τ, tag)` **or** `Propagated(ε')`; **never a drop** (I1). Bounded effects (`retry`/`cascade`) carry budgets → `EffectBudgetExhausted` (I4) | declared by the policy (`retry`/`alloc`/`io`/`cascade`), **bounded** (RFC-0014 I3/I4) | yes — every outcome records the `PolicyRef` (RFC-0014 §4.4) |

**Justification of non-`Exact` tags (downgrade to stay honest — VR-5):**
- `unwrap_or` / `unwrap_or_else` are **`Declared`** for the *substituted value*: a fallback is asserted, not
  proven, so it is at most `Declared` and flagged — RFC-0014 **I2** ("recovery never fabricates or upgrades
  a guarantee"). The op itself is total; the tag describes the *result value's* honesty.
- `recover` carries **no fixed tag** — it **inherits** the honest tag the RFC-0014 policy attaches to a
  recovered value (`Declared` for a bare `fallback`, higher only with an independent checked basis). This
  module never launders that tag upward (VR-5 / I2).
- Every other row is **`Exact`**: pure value combinators with no accuracy/precision/probability semantics
  are `Exact` by RFC-0016 **C2** (the `len`-style case).

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2 / RFC-0014 I1):** *the crux.* Propagation is the **default** posture and
  suppression is **structurally impossible** — there is no combinator in §3 that consumes an `Err`/`None`
  and silently yields success. `map`/`and_then` preserve the error (it survives in the result sum or
  short-circuits and propagates); `or_else`/`recover` are the **only** ways to turn an error into a value,
  and both are explicit and must themselves produce a `Result`/outcome (recover or re-propagate). The
  defaulted accessors (`unwrap_or`*) recover with an **honestly-tagged** value (never a silent guarantee
  upgrade — I2). The `unwrap`/`expect` family is the **named partial** exception: it **refuses loudly**
  (abort + diagnostic record), never a silent default — it is the explicit "I assert this is `Ok`" op, the
  one place an error can stop a computation, and it does so audibly. The one *lossy* op, `ok`
  (`Result→Option`), discards `ε` — this is a **flagged lossy conversion** (EXPLAIN-able, C3), not an
  unflagged drop; FLAGGED in §7 (Q2) for whether it should be gated behind a name that makes the loss
  unmissable.
- **C2 — honest per-op tag (VR-5):** every row carries a tag on `Exact ⊐ Proven ⊐ Empirical ⊐ Declared`
  (§4). Pure combinators are `Exact`; `unwrap_or`* substitutions are `Declared`; `recover` **inherits** the
  policy's tag and never upgrades it (RFC-0014 I2). Downgrade is the rule.
- **C3 — no black boxes / EXPLAIN (SC-3/G11):** the propagation form elaborates to an ordinary, inspectable
  `Match` (RFC-0014 §4.3), so it is `EXPLAIN`-able like any term. The `recover` bridge is fully transparent:
  every recovered/propagated outcome records the **`PolicyRef`** of the RFC-0014 policy that shaped it
  (RFC-0014 §4.4) — *"which policy acted on this error, and what does it do?"* is always answerable. The
  partial accessors' refusals carry a diagnostic record. The lossy `ok` conversion is EXPLAIN-noted.
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001):** every combinator is a **pure function**
  of its inputs (the closures are the only effect surface, and effectful closures declare their effects —
  C6). `Option`/`Result` are immutable value sums; combinators return new values, never mutate. Metadata
  (the guarantee tag, the `PolicyRef`) rides the value but is **not** identity (ADR-003).
- **C5 — above the small kernel (KC-3):** the module adds **no trusted code**. Combinators are ordinary
  functions over `core`'s value sums; the `?` form elaborates to existing `Match` (no new L0 node, exactly
  as RFC-0014 §4.3 establishes); the `recover` bridge is a typed handoff to M-520, not new kernel
  machinery. No `wild`/FFI (ADR-014).
- **C6 — declared, bounded effects (RFC-0014):** the pure combinators declare **no** effects. Where a
  combinator takes a closure (`and_then`, `or_else`, `inspect`, `unwrap_or_else`), the **closure's** declared
  effects flow to the call site (RFC-0014 I3) — the combinator is transparent to effects, it does not hide
  them. The `recover` bridge surfaces the RFC-0014 policy's **declared, bounded** effects (`retry`/`cascade`
  carry budgets; an overrun is an explicit `EffectBudgetExhausted` — I4), never an undeclared or unbounded
  one.

## 6. Grounding

- The never-silent floor is **RFC-0014 §4.2 I1** (a handler/combinator either recovers explicitly or
  re-propagates; an error can *neither vanish unobserved nor* be dropped) and **RFC-0016 §4.1 C1** (G2) —
  the structural guarantee this module exists to hold at the ergonomics layer.
- Additive, opt-in recovery is **RFC-0014 §4.2/§4.5 I5** (the narrowest scope is the default; turning an
  error into a value is opt-in via an explicit named combinator or the typed `recover` bridge — never
  ambient).
- Honest tags on recovered/fallback values are **RFC-0014 I2** + **RFC-0016 C2 / VR-5** (a substituted
  fallback is at most `Declared`; recovery may only downgrade).
- The `?`-form-as-`Match` elaboration and the no-new-kernel-node posture are **RFC-0014 §4.3** + **KC-3**.
- The `recover` bridge target — reified policies, the closed v0 action set, `PolicyRef`, effect budgets —
  is **RFC-0014 §4.4/§4.5** as delivered by **`std.recover` (M-520)** (RFC-0016 §4.3).
- The module's task, ring, tier, and the §4.5 guarantee-matrix obligation are **RFC-0016 §4.2/§4.4/§4.5**;
  the C1–C6 contract is **RFC-0016 §4.1**.
- The error *representation* and lattice tags this module consumes are **RFC-0001** via **`core` (M-515)**.

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) The exact `recover`-bridge signature.** `recover : Result<τ,ε>, PolicyRef -> RecoverOutcome<τ,ε>`
  is described **abstractly** here: the precise `RecoverOutcome` shape, the `PolicyRef` resolution surface,
  and how a policy's declared effects/budgets are reflected at this call site are **owned by `std.recover`
  (M-520, RFC-0014)** and are **not yet fixed**. This spec deliberately does **not** fabricate that API
  (the house grounding rule). *Disposition:* co-design the bridge signature with M-520; this module's
  obligation is only that the outcome is `Recovered | Propagated` with no drop variant (I1) and an honest
  inherited tag (I2). Ties to RFC-0016 §8-Q1 (the v0 module set / `recover` sequencing).
- **(Q2) Should the lossy `ok` (`Result→Option`) be a *named* lossy op?** It is the one combinator that
  discards an error value (`Err→None`). It is C1-honest only because it is a **flagged, EXPLAIN-able lossy
  conversion**, not a drop — but a casual reader could mistake it for a silent swallow. *Disposition:* a
  naming/ergonomics decision — gate it behind an unmistakable name (e.g. `ok_discarding_err`) or require an
  EXPLAIN-visible lossy marker, so the loss is unmissable at the call site. Ties to **RFC-0016 §8-Q3**
  (ergonomics-vs-contract: implicit-but-inspectable vs always-explicit at the call site).
- **(Q3) `unwrap`/`expect` refusal mechanics.** The partial accessors must **refuse loudly**, but the exact
  failure mechanism (abort vs an escalating RFC-0014 `escalate` vs a `std.diag`-rendered diagnostic before
  abort) crosses into M-510/M-520 territory. *Disposition:* the *guarantee* (loud refusal, never a silent
  default) is fixed here; the *mechanism* is co-designed with `diag`/`recover`. The honest floor holds
  regardless of which mechanism is chosen.
- **(Q4) `?`-on-`Option` vs `?`-on-`Result` unification.** Whether the propagation form is one polymorphic
  operator over both sums (with a typed coercion at the boundary, cf. `ok_or`) or two surface forms is an
  ergonomics-vs-clarity call. *Disposition:* a DN-level surface decision; both satisfy C1 (each propagates,
  neither drops), so it does not affect the honesty floor. Ties to RFC-0016 §8-Q3.

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.error` / `option` / `result` ergonomics module
  (M-527, #168; Ring 2, Tier B) under RFC-0016 (Draft). Fixes the **scope + boundary** (the pure combinator
  and propagation surface, bounded against `core`'s error *representation*, `recover`'s reified *policies*, and
  `diag`'s *presentation*); the **exported-op surface** sketch (`map`/`map_err`/`and_then`/`or_else`/
  `filter`/`ok_or`/`unwrap_or*`/`?`-propagate/`unwrap`-family/`recover`-bridge); and — the load-bearing
  deliverable — the **guarantee matrix** (11 rows; mostly `Exact` pure combinators, `Declared` for
  `unwrap_or*` substitutions per I2, and an *inherited* tag for the `recover` bridge), with the explicit
  property that **no row permits a silent drop** (RFC-0014 I1 / RFC-0016 C1). States the **§4.1 conformance**
  (C1 the crux: propagation is the default, suppression structurally impossible; `unwrap` is a named partial
  that refuses loudly, never a silent default; recovery additive + opt-in per I5, honestly tagged per I2),
  the **grounding** (RFC-0014 I1/I2/I5, RFC-0016 §4.1–§4.5, RFC-0001/`core`), and **four FLAGGED questions**
  (the abstract `recover`-bridge signature owned by M-520; whether lossy `ok` needs an unmistakable name; the
  `unwrap` refusal mechanism; `?`-unification) — each tied to RFC-0016 §8 where it crosses modules. No code;
  no kernel change (KC-3 — the `?` form elaborates to an existing `Match`, no new L0 node). Append-only.
- **2026-06-19 — §7-Q1 (recover bridge) RESOLVED.** `std.error` drops its abstract stub
  `RecoverOutcome` enum + `recover` fn and re-exports the concrete
  `mycelium_std_recover::{Outcome, Resolution, RecoverOutcome, handle_classified}` (M-520). `std.error`
  is the bridge *target*, not the home of the recovery algebra (KC-3); the I1 (no drop) / I2 (tag
  inherited, never laundered) contract holds verbatim in the landed types. Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
