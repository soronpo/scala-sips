---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:number
  - /sips/:title.html
stage: pre-sip
status: under-review
title: Match Types — Capture Bounds, Tuple Normalization, Disjointness Refinements, and Local GADT-Aware Reduction
---

**By: (to be filled in)**

## History

| Date          | Version            |
|---------------|--------------------|
| May 2026      | Initial Draft      |

## Summary

This proposal extends [SIP-56 *Proper Specification for Match Types*](https://docs.scala-lang.org/sips/match-types-spec.html) with a bundle of five small, mechanical refinements and one substantive new capability — *local GADT-aware reduction*. The bundle is designed to close the largest cluster of currently-stuck-but-ought-to-reduce match types reported on `scala/scala3` while preserving every soundness and TASTy-stability guarantee SIP-56 was designed to deliver.

The five small refinements are:

1. **Capture bound preservation** — instantiations of named captures must satisfy their declared bounds.
2. **Tuple-shape normalization in matching** — the matcher canonicalizes `TupleN[T1, …, Tn]` to nested `*:` pairs (and vice versa).
3. **`compiletime.ops.int.S` disjointness** — `0` is provably disjoint from `S[n]`; distinct `S[…]` literals reduce by structural recursion.
4. **Multi-member refined extractor** — case 4 of *legal patterns* extends to refinements with multiple captures (`Base { type Y1 = t1; …; type Yn = tn }`).
5. **Principled invariant disjointness** — the ad-hoc `cannotBeNothing` heuristic is replaced by a "type parameter is structurally used" criterion.

The substantive extension is:

6. **Local GADT-aware reduction** — match-type reduction may consume the GADT constraints in scope at the *call* site (typically introduced by a value-level pattern match), provided the resulting reduction is conformed against the match type's declared upper bound *before* it is allowed to escape the typing branch. This admits new reductions inside dependently-typed `def`s without ever pickling a context-dependent reduction into TASTy.

Every change is a strict expansion of the set of accepted programs and the set of reductions that succeed; no previously legal pattern becomes illegal, and no reduction that succeeds today produces a different result.

## Motivation

SIP-56 stabilized match types by carving out a class of *legal patterns* whose reduction is `TypeComparer`-independent and stable across TASTy files. That stability is precious, and this proposal preserves it. But three years of bug reports against the resulting design (≈190 issues, ≈50 currently open as of May 2026) show that SIP-56's specification, taken literally, is more conservative than soundness alone requires. Six categories of complaints recur:

### 2.1 Captures lose their declared bounds

```scala
class B[U, Y <: U]

type G[U, E] <: U = E match
  case B[U, y] => y
```

`y` is declared with bound `<: U` in the case lambda's parameter list, but during reduction the spec instantiates it with `scrut.hi` (or `scrut`), discarding the constraint. As a result, code that relies on `y <: U` — including the declared upper bound `<: U` of `G` itself — fails to typecheck. Affects [#25129](https://github.com/scala/scala3/issues/25129), [#24351](https://github.com/scala/scala3/issues/24351), [#24317](https://github.com/scala/scala3/issues/24317), [#19904](https://github.com/scala/scala3/issues/19904).

### 2.2 Tuple syntactic forms are not interchangeable

```scala
type IsTypeInTuple[T, Tup <: Tuple] <: Boolean = Tup match
  case T *: ts => true
  case _ *: ts => IsTypeInTuple[T, ts]
  case EmptyTuple => false

summon[IsTypeInTuple[String, Int *: String *: EmptyTuple] =:= true]  // ok
summon[IsTypeInTuple[String, (Int, String)]              =:= true]   // stuck
```

The two scrutinees are propositionally equal, but the matcher only normalizes one direction. Affects [#9871](https://github.com/scala/scala3/issues/9871), [#16583](https://github.com/scala/scala3/issues/16583), [#17186](https://github.com/scala/scala3/issues/17186).

### 2.3 Peano-natural patterns lack disjointness

```scala
type Pred[N <: Int] = N match
  case 0    => 0
  case S[n] => n
```

`0` and `S[n]` cannot be proved disjoint by SIP-56's `provablyDisjoint`, so a recursive `Pred` cannot exhaustively dispatch over `Int` literals introduced by `compiletime.ops.int`. Affects [#14572](https://github.com/scala/scala3/issues/14572).

### 2.4 Refined-member extractor is one-capture only

SIP-56's legal-patterns case 4 admits `Base { type Y = t }` with a single capture. But a class with several abstract type members has no spec-blessed extractor:

```scala
class Pair { type A; type B }
type ExtractAB[X] = X match
  case Pair { type A = a; type B = b } => (a, b)  // illegal under SIP-56
```

Affects [#22644](https://github.com/scala/scala3/issues/22644), [#24548](https://github.com/scala/scala3/issues/24548).

### 2.5 Invariant disjointness depends on an admitted-ad-hoc rule

`provablyDisjointTypeArgs` currently invokes `cannotBeNothing` as a fallback to fix [#21295](https://github.com/scala/scala3/issues/21295) without breaking the world. The implementation comment is candid:

> sjrd: I will not be surprised when this causes further issues in the future.

It already has — the heuristic interacts surprisingly with phantom type parameters and refined classes.

### 2.6 GADT narrowing does not propagate into reduction

```scala
sealed trait Foo[T]
case class Bar[A](v: A) extends Foo[Array[A]]

type Elem[T] = T match
  case Array[x] => x

def f[A](x: Foo[A]): Elem[A] = x match
  case Bar(v) => v   // typer narrows A to Array[a]; Elem[A] is still stuck
```

The value-level pattern match introduces a GADT constraint `A =:= Array[a]`. The typer uses it to elaborate the right-hand side. But `Elem[A]`'s reduction proceeds in a `frozenConstraint` reducer that ignores the narrowing, so `Elem[A]` remains stuck, and the user sees `Found: a, Required: Elem[A]`. This is the largest cluster of the open backlog: [#25966](https://github.com/scala/scala3/issues/25966), [#21391](https://github.com/scala/scala3/issues/21391), [#21425](https://github.com/scala/scala3/issues/21425), [#18546](https://github.com/scala/scala3/issues/18546), [#16058](https://github.com/scala/scala3/issues/16058), and several smaller dependents.

### 2.7 Scope of this proposal

In scope:

- Any change that admits more reductions while preserving SIP-56's three core guarantees:
  - **(G1)** if a match type reduces to `R` in compiler version *v*, it reduces to `R` in version *v+1*;
  - **(G2)** soundness preservation along subtyping (`X <: Y` and `Y match … = R` ⇒ `X match … = R` if it reduces);
  - **(G3)** TASTy stability — pickled reductions remain consistent across recompilations.
- Refining `provablyDisjoint` where the existing rule is admitted ad hoc.

Out of scope:

- **Higher-kinded captures** (`case Aux[f] =>` with `f` of kind `* → *`). Generalizing `matchPattern` to instantiate type lambdas is higher-order unification and decidability work; left for a future SIP.
- **Cross-module override soundness** ([#20189](https://github.com/scala/scala3/issues/20189) and the cluster). Would require re-checking pickled reductions on subclass elaboration, breaking the "reduce once" invariant SIP-56 explicitly enables.
- **Visibility leakage** ([#20194](https://github.com/scala/scala3/issues/20194)). Privacy is not part of the type model match types operate over.

## Proposed solution

### 3.1 Extension 1 — Capture bound preservation

#### Specification change

In *Matching* (SIP-56 §Matching), wherever the spec currently instantiates a non-wildcard `TypeCapture` `ti` to a value `V` (one of `scrut.hi`, `scrut.lo`, or `scrut`), the operation now succeeds only if:

> `[ts := ts′] caseBounds(ti)` *contains* `V`, where `caseBounds(ti)` is the bound declared for `ti` in the case lambda's parameter list.

If containment fails, the case is *not specific*: emit a `NoInstance` failure with the offending bound (mirroring the existing not-specific path).

For wildcard captures, the existing rule (`scrut.hiBound` / `scrut.loBound` / `scrut` per variance) is unchanged — wildcards have no declared bounds.

#### Worked example

```scala
class B[U, Y <: U]
type G[U, E] <: U = E match
  case B[U, y] => y     // case lambda is [U, y <: U]
```

With `E = B[Int, String]`, the match attempts `y := String`. The capture `y` has declared bound `y <: U`. Substituting `[U := Int]` gives `y <: Int`. `String` does not satisfy `<: Int`, so the case is `NoInstance`, the user sees a clear "type capture `y` could not be instantiated; required `<: Int`, found `String`", and reduction does not silently produce a result that violates `G`'s declared upper bound.

With `E = B[Int, Byte]`, `y := Byte` does satisfy `<: Int`, so reduction produces `Byte`. The result then conforms to `G[Int, B[Int, Byte]] <: Int` as declared.

#### Soundness

The rule is strictly more restrictive than SIP-56: every successful reduction was already accepted by SIP-56, and produces the same result. The new failures (`NoInstance` instead of silent ill-typed reduction) replace currently-buggy behavior — today's reductions that violate declared bounds either (a) fail later with a confusing error or (b) leak ill-typed values. Either way, no previously *successful, well-typed* reduction changes.

(G1), (G2), (G3) preserved trivially.

### 3.2 Extension 2 — Tuple-shape normalization in matching

#### Specification change

In §Matching, before performing the `BaseTypeTest` rule for class type constructors, the matcher applies a *tuple normalization* step:

> If the scrutinee `X` is propositionally equal to a `TupleN[T1, …, Tn]` (n ≥ 1, n ≤ 22), it is treated as `T1 *: T2 *: … *: Tn *: EmptyTuple` for the purpose of pattern shape comparison. Symmetrically, a pattern shape `T1 *: T2 *: … *: Tn *: EmptyTuple` admits scrutinees of shape `TupleN[T1, …, Tn]`.

The reverse direction — a `TupleN[…]` pattern matched against a nested-pair scrutinee — applies the same normalization to the pattern.

This is the same transformation `tryConvertToSpecPattern` already performs for *patterns* in `Types.scala:5473`; we lift it into the spec and apply it symmetrically to scrutinees.

#### Worked example

```scala
type IsTypeInTuple[T, Tup <: Tuple] <: Boolean = Tup match
  case T *: ts    => true
  case _ *: ts    => IsTypeInTuple[T, ts]
  case EmptyTuple => false

summon[IsTypeInTuple[String, (Int, String)] =:= true]   // now reduces
```

`(Int, String)` is `Tuple2[Int, String]`; under normalization it is `Int *: String *: EmptyTuple`, which matches case 2 with `ts := String *: EmptyTuple`, recurses, matches case 1.

#### Soundness

`Tuple2[A, B] =:= A *: B *: EmptyTuple` is already a derived equality in the standard library. Normalization in the matcher does not change subtyping, only enables a structural match to recognize the equivalence. Because both forms reduce to the same nested-pair via existing equalities, no two reductions disagree. (G1)–(G3) preserved.

### 3.3 Extension 3 — `compiletime.ops.int.S` disjointness

#### Specification change

Augment §Disjointness with rules treating `S[…]` as a constructor disjoint from `0`:

- `0 ⋔ q.S[T]` for any `T`.
- `q.S[T] ⋔ 0`.
- `q.S[T1] ⋔ q.S[T2]` if `T1 ⋔ T2`.
- `c ⋔ q.S[T]` if `c` is a `ConstantType` of value `0`.
- `q.S[T] ⋔ c` if `c` is a `ConstantType` of value `0`.

(Distinct *non-zero* literal `Int`s are already disjoint via the existing `ConstantType` rule. Disjointness between a literal `n > 0` and `S[T]` falls out by recursion: `n` reduces to `S[…S[0]…]` via `compiletime.ops.int` evaluation, and the rules above recurse.)

#### Worked example

```scala
type Pred[N <: Int] = N match
  case 0    => 0
  case S[n] => n
```

`Pred[3]` has scrutinee `3`, which evaluates to `S[S[S[0]]]`. The first case `0` is provably disjoint from `S[S[S[0]]]`; the matcher skips it. The second case matches with `n := S[S[0]] = 2`. Reduction produces `2`.

Without the new rule, `Pred[3]` would today succeed at the second case (via legacy reducer arithmetic) but the disjointness with `0` cannot be proved, so the empty-scrutinee guard does not fire — meaning a recursive `Pred[Pred[3]]` may unexpectedly stick on a downstream subtype check.

#### Soundness

The rule is sound by injectivity of `S` and the empty intersection of `0` with the image of `S`. Both facts are already exploited by the matcher's `CompileTimeS` pattern handler (`TypeComparer.scala:3777`); this extension formalizes their consequence for `provablyDisjoint`.

### 3.4 Extension 4 — Multi-member refined extractor

#### Specification change

Replace SIP-56 *Legal patterns* case 4 (`TyconWithoutCapture` clause "It is a refined type `Base { type Y = t }`") with:

> 4. It is a refined type `Base { type Y1 = t1; type Y2 = t2; …; type Yn = tn }` (n ≥ 1) where:
>    - `Base` is a `TypeWithoutCapture`;
>    - each `Yi` is a type member already declared in `Base`;
>    - each `ti` is a `TypeCapture`;
>    - the `ti` are pairwise distinct.

The matching rule generalizes correspondingly: §Matching's "If `T` is refined type" case becomes:

> Compute `q.Yi` for each `i` simultaneously (sharing the skolem `∃α:X` if `X` is not stable). For each `i`, perform the existing single-member rule. If any single-member rule fails (no member, abstract member, skolem leakage), the whole multi-member match fails the same way it would have for that member.

#### Worked example

```scala
class Pair { type A; type B }
type ExtractAB[X] = X match
  case Pair { type A = a; type B = b } => (a, b)

type R = ExtractAB[Pair { type A = Int; type B = String }]   // = (Int, String)
```

Multi-member refinement is structurally a refinement chain `Pair { type A = a } { type B = b }`; the matcher already handles single-member refinements per SIP-56. The extension is to apply the rule recursively over the chain rather than rejecting any chain longer than 1.

#### Soundness

Each member's extraction is independent and uses the existing SIP-56 single-member extractor. The skolem-leakage check (`DropSkolemMap` in `TypeComparer.scala:3793`) applies per member as before. No member's extraction can produce an unsound binding that wasn't already possible under SIP-56 with the same single member.

(G1)–(G3) preserved: today, multi-member extractors fall through to `LegacyPatMat` under `-source:3.3` (sometimes accidentally working) and to `MatchTypeLegacyPattern` errors under `-source:3.4+`. The proposal makes them legal and reduces them deterministically; old TASTy that pickled the legacy result remains valid (legacy reduction either matched the same way or produced a stuck type; in both cases the new behavior is a strict expansion).

### 3.5 Extension 5 — Principled invariant disjointness

#### Specification change

Replace `cannotBeNothing` with a *structural use* predicate. In §Disjointness's "common base type with disjoint arguments" rule:

For an invariant type parameter `T` of a class `E`:

> Two arguments `A`, `B` are *invariantly disjoint* with respect to `T` if `A ⋔ B` and `T` is *structurally used* in `E`.

`T` is *structurally used* in class `E` iff at least one of the following holds:

1. `E` has a public or protected field whose type's free occurrences include `T` covariantly or invariantly (the existing field rule).
2. `E` has a public or protected method (including the constructor's parameter list) whose result type's free occurrences include `T` covariantly or invariantly, *or* whose parameter types' free occurrences include `T` contravariantly or invariantly.
3. `E` has a non-private abstract type member whose declared bounds reference `T`.

A type parameter that does not satisfy any of these is *phantom*; phantom invariant parameters cannot prove disjointness.

The covariant rule is unchanged (already requires "field of that type parameter").

#### Worked example

```scala
class Tag[T]                           // phantom: T not used
class Box[T](val value: T)             // T used as field
trait Reader[T] { def read: T }        // T used in method result
trait Writer[T] { def write(t: T): Unit } // T used in method parameter
class Pure[T] { type M <: T }          // T used in member bound

// Under SIP-56 + cannotBeNothing:  Tag[Int] ⋔ Tag[String] ?  Sometimes (depending on heuristic).
// Under the new rule:              Tag[Int] ⋔ Tag[String] ?  No.

// Box[Int]    ⋔ Box[String]    : yes (field rule, unchanged).
// Reader[Int] ⋔ Reader[String] : yes (method-result rule).
// Writer[Int] ⋔ Writer[String] : yes (method-parameter rule).
// Pure[Int]   ⋔ Pure[String]   : yes (member-bound rule).
```

The `Pure` case is what `cannotBeNothing` was groping at: `Pure[String]` cannot have a value because `M <: String` and `M <: Int` are incompatible *if* `M` could be summoned; but the rule doesn't depend on `M`'s instantiability, only on the structural reference.

#### Soundness

A phantom invariant parameter genuinely cannot distinguish two values: `new Tag[Int]()` *is* a valid `Tag[String]` runtime witness because there is nothing in `Tag` that depends on the choice. SIP-56's existing covariant rule already encodes this. The current `cannotBeNothing` check overshoots: it occasionally rules disjointness for phantom parameters when one of the args happens not to be `Nothing`, which is unrelated to soundness.

The new rule is *strictly stronger* than the field rule alone (admits more disjointness proofs) and *strictly weaker* than the field rule plus `cannotBeNothing` (rejects the phantom-parameter spurious disjointness). It thus closes both:

- spurious disjointness on phantoms (a soundness concern in the limit, even if no exploit is currently known);
- missed disjointness on `Reader`/`Writer`/`Pure` shapes (the conservatism #21295's fix tried to repair).

(G1) holds because every reduction that succeeded under the old rule succeeds under the new rule *or* was relying on a phantom-parameter disjointness that was unsound to begin with — those are the only reductions that change.

A backward-compatibility exemption: under `-source:3.4` through `-source:3.6`, retain the `cannotBeNothing` rule alongside the new structural-use rule (logical OR). Drop `cannotBeNothing` under `-source:future` and 3.7+. This preserves (G3) for TASTy compiled against intermediate compilers.

### 3.6 Extension 6 — Local GADT-aware reduction

#### Background

SIP-56 deliberately reduces match types under `inFrozenConstraint` (`TypeComparer.scala:4006`) so that no type-variable instantiation or GADT constraint can leak into the reduction. This is the foundation of (G1) and (G3): the reduction of `M[X]` is determined by `X` alone, not by what the typer happens to know in some calling context. Without that, two compilation units reducing the same `M[Y]` could disagree depending on whose calling context they were last seen in, and TASTy stability would collapse.

The cluster of issues in §2.6 shows the cost of that decision: dependent `def`s that pattern-match a value-level GADT cannot use the narrowing in their match-type-typed return position. The user sees a phantom mismatch and is forced to write an `asInstanceOf`.

The escape: *context-sensitive reduction is sound iff it never crosses into pickled output*. The typer already computes return types of dependent matches against an expected type derived from the declared signature. We can let reduction consume GADT constraints inside that elaboration, as long as the resulting type is conformed against the *declared upper bound* of the match type *before* it is allowed to participate in any subsequent inference, pickling, or member computation that might escape the branch.

#### Specification change

Introduce a new reduction mode, *local reduction*, which is the only mode in which GADT constraints participate. The existing reduction mode (henceforth *global reduction*) is unchanged.

##### Local reduction — when it fires

Local reduction may be performed by the typer at the following sites only:

- (LR-Match) When elaborating the body of a value-level case `case p => rhs` where `p` introduces GADT constraints `G`, while typing `rhs`. The expected type for `rhs` may be reduced under `G`.
- (LR-Inline) When reducing a transparent inline call's result type, after the call has been specialized but before it has been pickled.

Outside these sites, *global* reduction is used, exactly as in SIP-56.

##### Local reduction — algorithm

`localReduceMatchType(M, G)` where `M = X match { case P1 => R1; …; case Pn => Rn }` and `G` is the current GADT constraint set:

1. Apply `G` to `X`, producing `X'` — every type symbol with a GADT bound `>: L <: H` is replaced by an existential `_ >: L <: H` for the purpose of reduction. (This is the same widening used elsewhere when narrowing a scrutinee under GADT.)
2. Run the SIP-56 reduction algorithm on `M' = X' match { … }`, producing either `R` (reduced) or `NoType` (stuck).
3. If reduced to `R`, check `R <: B` where `B` is the declared upper bound of `M` (recall: every GADT-aware reduction site requires the match type to have a declared upper bound; see *Restriction* below). If conformance fails, treat the local reduction as if it returned `NoType`.
4. Return `R` (or `NoType`).

The *result of local reduction is not cached* on the `MatchType` instance, is not pickled, and does not participate in `MatchType.thatReducesUsingGadt` outside the LR-Match / LR-Inline scope. It is consumed by the typer, conformed against an expected type, and discarded.

##### Local reduction — restrictions

LR-Match fires for a match type `M` only if all of:

- `M` has a *declared upper bound* `B` that does not itself depend on the GADT-narrowed symbols.
- The match type's scrutinee `X` mentions at least one symbol bound in `G`.
- The reduction is consumed entirely within the typing of `rhs` — no `R` flows into a nested call's expected type unless that nested call is itself within `rhs`.

If any restriction is violated, fall back to global reduction.

##### Subtyping integration

The existing match-type subtyping rules (SIP-56 §Subtyping) all use *global* reduction, unchanged. Local reduction only feeds the typer's expected-type machinery: it can declare a value's type to be `R`, but the *type system* still sees `M` (or its declared upper bound `B`) at that position. Concretely:

- A `val x: M[A] = …` whose RHS produced `R` via local reduction is well-typed iff `R <: M[A]` *globally*. Since `R <: B` was checked in step 3 and `M[A] <: B` in general, this collapses to `R <: B`, which holds by construction.
- The pickled type of `x` is `M[A]`, not `R`. (G3) preserved.

#### Worked example

```scala
sealed trait Foo[T]
case class Bar[A](v: A) extends Foo[Array[A]]

type Elem[T] <: Any = T match
  case Array[x] => x

def f[A](x: Foo[A]): Elem[A] = x match
  case Bar(v) => v
```

Typing `case Bar(v) => v`:

1. The typer narrows `A =:= Array[a]` and introduces GADT bound `A >: Array[a] <: Array[a]`.
2. The expected type for `v` is `Elem[A]`. Under global reduction this is stuck.
3. LR-Match fires: scrutinee `A` has a GADT bound; `Elem` has declared upper bound `Any`; reduction is local.
4. Apply `G`: scrutinee becomes `Array[a]`. Run SIP-56 reduction: case `Array[x] => x` matches with `x := a`. Result `R = a`.
5. Check `a <: Any`: yes.
6. Typer accepts `v: a`, conformed to expected `Elem[A]` *via the declared upper bound `Any`* — `a <: Any`.

The pickled signature of `f` remains `def f[A](x: Foo[A]): Elem[A]`. A separate compilation unit calling `f(someBar)` sees `Elem[A]` and reduces it *globally* with whatever `A` it knows; it never observes `R = a`.

#### Soundness

The result of LR is consumed only by the typer's "is this RHS acceptable for this expected type?" check, and is bounded by the declared `B` before being released. Because:

- the typer already trusts GADT narrowing to type `rhs` at value level (this is how value-level GADT typing has always worked);
- the bound `B` does not depend on GADT-narrowed symbols (restriction);
- the pickled type is the un-narrowed `M`, not `R`;

no two compilation units can ever observe disagreeing reductions of the same `M[X]`. (G1) and (G3) preserved.

(G2) is preserved because LR is only ever a *more permissive* result than global reduction at the same site: anywhere global reduction succeeded with `R`, local reduction will also reach `R` (it is a strict superset of the search space). The conformance check `R <: B` holds for global reductions by SIP-56's `<: B` invariant. So no acceptance flips to a rejection, and no rejection of an unsound reduction flips to acceptance.

#### What this is *not*

It is not GADT-aware reduction inside arbitrary type expressions. It is not pickled. It does not change the subtyping relation. It does not let `M[A]` reduce *globally* differently because some caller's typer happens to know `A =:= Array[a]`. It is, deliberately, the smallest possible relaxation that closes the §2.6 cluster.

### 3.7 Implementation summary

| Extension | Files most affected | Rough size |
|---|---|---|
| 1. Capture bound preservation | `Types.scala` (`MatchTypeCaseSpec.analyze` / capture lambda), `TypeComparer.scala` (`matchSpeccedPatMat` capture instantiation) | ~30 lines |
| 2. Tuple normalization in scrutinee | `Types.scala` (`MatchType.normalized` or scrutinee preprocessor) | ~20 lines |
| 3. `S` disjointness | `TypeComparer.scala` (`provablyDisjoint`) | ~30 lines |
| 4. Multi-member extractor | `Types.scala` (`tryConvertToSpecPattern` refinement case), `TypeComparer.scala` (`TypeMemberExtractor` rule loop) | ~60 lines |
| 5. Principled invariant disjointness | `TypeComparer.scala` (`provablyDisjointTypeArgs`, `cannotBeNothing` removal, new `isStructurallyUsed`) | ~50 lines |
| 6. Local GADT reduction | `Typer.scala` (LR-Match call site, LR-Inline call site), `TypeComparer.scala` (`MatchReducer` GADT-mode flag), `Types.scala` (no caching of LR results) | ~150 lines |

Total estimated diff: ~340 lines plus tests.

## Compatibility

### Backward source compatibility

Every extension is a *strict expansion* of the set of programs that compile and the set of match-type reductions that succeed:

- Extension 1 changes some currently-successful-but-ill-typed reductions into *errors* — but the affected reductions today fail anyway with a downstream "found X, required Y" error. The new error message is more local and specific.
- Extensions 2, 3, 4, 6 turn previously-stuck match types into reducing ones. No previously-reducing type changes its result.
- Extension 5: under `-source:3.4`–`3.6`, the new rule is `OR`-ed with the legacy `cannotBeNothing`, so no reduction that previously succeeded fails. Under `-source:future`+, programs that relied on phantom-parameter disjointness — *none of which are currently known to exist in published libraries* — would need to add a real structural use of the parameter (e.g., a tag method).

A targeted Maven-Central re-typecheck (mirroring SIP-56's compatibility analysis) should be run before final acceptance. SIP-56 found 8/779 cases needed adjustment; we expect this proposal's number to be 0 (since each extension only *adds* reductions or changes ad-hoc heuristics whose breakage is not surfaced in published code).

### Backward TASTy compatibility

Pickled match types continue to be reduced by the *global* algorithm. Extensions 1–5 modify the global algorithm but only in ways that turn "stuck" into "reduces to R". A TASTy file pickled by an older compiler that recorded `M[X]` as stuck remains valid: the newer compiler may now reduce it to `R`, which is consistent with the older observation (a stuck match type `M` participates in subtyping via its declared upper bound, and `R <: B` by SIP-56's well-formedness rule, so the new behavior is a refinement of the old).

A TASTy file pickled by an older compiler that recorded `M[X]` as reducing to `R` will continue to reduce to the same `R` — none of the extensions change the result of a previously-successful reduction.

Extension 6 does not affect TASTy at all, by construction.

### Backward binary compatibility

Match-type reduction does not directly produce bytecode; binary compatibility is mediated by the erasure of the resulting type. Because extensions 1–5 do not change any *successful* reduction's result, erasure is unchanged for any program that previously compiled. Extensions 2, 3, 4, 6 may erase newly-reducing match types differently from their declared upper bounds, but only at sites that previously did not compile — there is no previously-shipped binary to compare against.

### Forward TASTy compatibility

A TASTy file pickled by a newer compiler may reference a match type whose reduction the older compiler does not know about (e.g., extension 4's multi-member extractor). This is the same forward-compatibility profile as any non-trivial type-system extension. The standard Scala 3 forward-compatibility window applies.

## Feature Interactions

### Opaque types

Local reduction (extension 6) does not look through opaque types any more than global reduction does. Inside the defining module, opaque types unfold; outside, they don't. GADT narrowing of an opaque-typed parameter produces a narrowed *opaque* type, and the narrowing is consumed by LR exactly like any other narrowing. No interaction beyond what SIP-56 already specifies.

### Transparent inline

LR-Inline (extension 6) overlaps with `transparent inline`'s existing ad-hoc narrowing. Today, a transparent inline `def`'s return type can be more specific than its declared signature; this is a form of localized type-level computation. LR-Inline formalizes the part of that computation that goes through match types: the inline call's result type is reduced under the (effectively GADT) constraints inferred from the call's actual arguments, and the result is conformed against the declared signature before being pickled into the caller's TASTy. This subsumes some currently-fragile patterns around `transparent inline` + match types ([#13250](https://github.com/scala/scala3/issues/13250), [#19857](https://github.com/scala/scala3/issues/19857), [#21015](https://github.com/scala/scala3/issues/21015)).

### Dependent value-level matches

Extension 6 makes dependent-typed matches "just work" in the most-asked-about cases (§2.6). The existing `typedDependentMatchFinish` (`Typer.scala:2293`) is the natural call site for LR-Match; the patch is to invoke `localReduceMatchType` instead of going through global `MatchType.reduced` for the per-case expected type.

### Implicit search

Implicit search is unchanged. Extensions 1–5 may make new givens findable by reducing previously-stuck result types ([#17395](https://github.com/scala/scala3/issues/17395), [#17907](https://github.com/scala/scala3/issues/17907), [#18211](https://github.com/scala/scala3/issues/18211)), but the search algorithm itself is not modified. Implicit search continues to use *global* reduction; LR is purely a typer-elaboration tool.

### Capture checking

Capture-checking ([#25843](https://github.com/scala/scala3/issues/25843)) interacts with match types via the capture-set machinery. None of the proposed extensions touches `CaptureSet`; the surface for new capture-checking interactions is zero. Existing capture-checking bugs are out of scope.

## Other concerns

### Compilation performance

Extensions 1, 3, 4 each add a localized check; impact on compile time should be negligible. Extension 2's tuple normalization is an O(n) preprocessing step on the scrutinee, only fired when the scrutinee is an `AppliedType` to a `TupleN` constructor. Extension 5 replaces one heuristic with another of the same algorithmic class. Extension 6's LR adds one extra reduction attempt per dependent-match case; in the common-case fast path (no GADT constraints in scope) it short-circuits before the SIP-56 algorithm is even invoked.

A microbenchmark on `community-build` is recommended before final acceptance.

### Error messages

Extension 1 produces a new error: "type capture `y` could not be instantiated; required `y <: Int`, found `String`". Extensions 2, 3, 4, 6 are silent in the success case; in the failure case they leave the existing SIP-56 error machinery (`MatchTypeTrace.stuck`, `noMatches`, `noInstance`) unchanged. Extension 5 changes which type pairs are reported as disjoint; the user-facing message ("selector is uninhabited" / "selector matches none of the cases") is unchanged.

### Tooling

The presentation compiler, IDE hover, and Scaladoc all consume `MatchType.reduced` (global reduction). None observes LR. Tools should continue to work without changes.

### Open questions

1. **Does LR-Match need a way to *explicitly opt out*?** Some users may prefer a stuck match type's "fail loudly" behavior to LR's "silently reduce locally". A `// scalac:` directive or annotation could disable LR per-call. The proposal currently does not include this.
2. **Should extension 4's multi-member rule require the captures to be *ordered* the same way as the parent's declarations?** The current draft does not; refinement order is reordered freely. This could be surprising in pretty-printed errors but is consistent with how Scala already treats refinement order.
3. **Is the `cannotBeNothing` retention window** (`-source:3.4`–`3.6`) **correct?** A more aggressive removal (3.5+) is plausible but riskier; a more conservative retention (3.7) would delay closing the soundness gap on phantoms.
4. **Should extension 3 generalize to any `compiletime.ops` constructor**, or stay specific to `S`? Generalizing to e.g. `+`, `*` would require encoding their algebraic identities (`a + b ⋔ a + c` if `b ⋔ c`); useful but a much larger surface and arguably its own SIP.

## Alternatives

### A1. Status quo (do nothing)

The cluster of §2.6 issues stays open indefinitely. Users continue to use `asInstanceOf` in dependent matches. The ad-hoc `cannotBeNothing` rule remains. Extensions 1–4 each have to be filed as separate one-off SIPs or compiler PRs. Cost-of-deferral is high because each individual extension is small but the bundle has shared reviewer cost.

### A2. Six separate proposals

Each extension as its own PR / mini-SIP. Pros: smaller review surface per change. Cons: the soundness arguments overlap (extensions 1, 5, 6 all rely on the same TASTy-stability framing), so reviewing them together is more efficient; and extensions 1 and 6 *interact* (extension 1's bound preservation feeds extension 6's `R <: B` conformance), so splitting them makes the LR specification harder to phrase correctly.

### A3. Full GADT-aware global reduction

Rather than restrict GADT awareness to local reduction, allow it everywhere — pickle GADT-narrowed reductions in TASTy. This is the most powerful design but breaks (G3) outright: two compilation units reducing the same `M[X]` could disagree based on their independent inference contexts. Rejected for the same reasons SIP-56 rejected the pre-SIP-56 status quo.

### A4. GADT awareness as a compiler-flag opt-in (`-Xexperimental:gadt-match-types`)

Allow A3-style global GADT reduction behind a flag, accepting the (G3) breakage as the price. Useful as a research vehicle but not as a stable language feature. Rejected for the same reason as A3.

### A5. Higher-kinded captures included in the bundle

Include `case Aux[f] =>` with HK `f`. Rejected because (a) HK matching is higher-order unification and admits no obvious decidable restriction; (b) it would balloon the proposal's review surface; (c) it does not depend on any of extensions 1–6 and can be filed independently when ready.

## Related work

- [SIP-56: Proper Specification for Match Types](https://docs.scala-lang.org/sips/match-types-spec.html) — the foundation this proposal extends. Every change is consistent with SIP-56's framing of *legal patterns*, *matching*, *disjointness*, and *subtyping*.
- Olivier Blanvillain, *Abstractions for Type-Level Programming*, Chapter 4 (Match Types). The academic foundation; specifies a calculus close to SIP-56 but without GADT awareness.
- TypeScript's *conditional types* with `extends`-clause inference (`T extends Array<infer X> ? X : never`) are the closest cousin in another language. TS conditional types reduce on non-ground types (similar to LR) and *do* propagate through unions; they do not face Scala's TASTy-stability constraint.
- Haskell's closed type families with [non-linear matching and `~` constraints](https://gitlab.haskell.org/ghc/ghc/-/wikis/closed-type-families) are related but operate over a fundamentally different elaboration model (no separate compilation of closed type family equations across packages).
- Scala 3 issue clusters cited in §2.

## FAQ

### Why not just remove `inFrozenConstraint` from `MatchReducer`?

Because that would silently break (G3): the reduction of `M[X]` would depend on whatever GADT or type-variable state happens to be live when the reduction is *first cached*, and subsequent references in other compilation units would be inconsistent. LR (extension 6) is what you get when you take "remove `inFrozenConstraint`" and add the minimum guard rails to keep (G3): result not cached, not pickled, conformed against declared upper bound.

### Will this break my library that uses match types?

Almost certainly not. Extensions 1–4, 6 only ever turn "stuck" into "reduces" — code that compiled today continues to compile, and most users will see *fewer* "could not be reduced" errors. Extension 5 changes which type pairs are disjoint; the affected programs are those that relied on phantom-parameter disjointness, which we believe to be empty in published code (subject to a Maven-Central re-typecheck).

### Does this enable `Tuple.InverseMap` to handle named tuples?

Partially. Extension 4 (multi-member extractor) is the spec hook needed for refined-member based `Tuple.InverseMap` variants over named-tuple-shaped scrutinees ([#22422](https://github.com/scala/scala3/issues/22422)). Whether the standard library exposes a new combinator is library design, out of scope here.

### What about `tests/neg/6570.scala`?

Untouched. The empty-scrutinee soundness rule (`MatchResult.ReducedAndDisjoint`) is the bedrock that prevents the `ClassCastException` exploit, and none of the extensions weaken it. Extension 5 actually *tightens* it by preventing phantom-parameter false-positive disjointness from making the rule fire spuriously.

### Can I get Extension 6 (LR) without the others?

Technically yes; LR is the most independent of the six. Practically no, because LR's `R <: B` conformance step is most useful when `R` itself has accurate captured-type bounds (Extension 1) and when more match types reduce in the first place (Extensions 2–5). The bundle is sized so LR has the most material to work on.
