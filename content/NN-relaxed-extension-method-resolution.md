---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:title.html
  - /sips/:number
stage: design
status: submitted
presip-thread: https://contributors.scala-lang.org/t/relaxed-extension-methods-sip-54-are-not-relaxed-enough/6585
title: Relaxed Extension Method Resolution
---

**By: Oron Port**

## History

| Date          | Version            |
|---------------|--------------------|
| Jul 14th 2026 | Initial Draft      |

## Summary

[SIP-54 (Multi-Source Extension Overloads)](https://docs.scala-lang.org/sips/multi-source-extension-overloads.html)
relaxed the resolution of extension methods so that same-named extension methods
*imported from several sources at the same nesting level* no longer produce an
ambiguity error: the compiler tries each import and keeps the one whose receiver
type matches. This proposal takes the next step. It relaxes extension method
resolution *across nesting levels* so that an extension method visible in a
closer scope no longer **shadows** an applicable extension method of the same
name defined (or imported) in an enclosing scope.

Concretely, when a selection `e.m` is rewritten to the extension application
`m(e)`, the compiler currently resolves `m` with the classic lexical scoping
rules: the closest binding of `m` wins and everything further out is discarded
*before* the receiver type is even considered. If that closest `m` is an
extension method that does not apply to the type of `e`, resolution fails —
even though a perfectly good extension method `m` is in scope one level out.
This proposal changes extension-method resolution to fall back to the next
enclosing precedence level when no candidate at the current level applies to the
receiver, so that the *best extension method for the receiver type* is chosen,
with lexical precedence used only as a tie-breaker. This restores parity with
the pre-existing behaviour of Scala 2 `implicit class`es, which disambiguate by
the extended type rather than by the method name.

## Motivation

Extension methods were introduced, among other reasons, as the modern
replacement for the `implicit class` pattern. Users reasonably expect to be able
to migrate an `implicit class` to an `extension` and keep the same call-site
behaviour. Today that expectation is broken in a common situation: two extension
methods that share a name but extend *different* types.

### Running example

```scala
class Foo[T]
object Lib:
  extension (foo: Foo[Int])
    def bar: Unit = {}

import Lib.*
extension (foo: Foo[String])
  def bar: Unit = {}

val f = Foo[Int]()
f.bar // error: value bar is not a member of Foo[Int]
```

([scastie](https://scastie.scala-lang.org/UsJXtJ0YRYqj1i5WR4gwJA))

The library `Lib` defines `bar` on `Foo[Int]`; the user defines a *different*
`bar` on `Foo[String]`. At the call site `f.bar`, `f : Foo[Int]`. The only
extension method that could possibly apply is `Lib`'s. Yet the call fails,
because the locally-defined `bar` (on `Foo[String]`) has higher lexical
precedence than the wildcard-imported `bar` (on `Foo[Int]`) and therefore
*shadows* it. Name resolution commits to the local `bar` and never falls back to
`Lib`'s, so the receiver type `Foo[Int]` is rejected against the parameter type
`Foo[String]`.

### The same code works with `implicit class`

Replacing the extensions with the `implicit class` construct they were meant to
supersede makes the example compile and behave as expected:

```scala
class Foo[T]
object Lib:
  implicit class FooIntExt(foo: Foo[Int]):
    def bar: Unit = {}

import Lib.*
implicit class FooStringExt(foo: Foo[String]):
  def bar: Unit = {}

val f = Foo[Int]()
f.bar // works!
```

([scastie](https://scastie.scala-lang.org/j6Cc1143Q6qQw1p9NDeUOA))

The difference is fundamental to how the two features resolve. `implicit class`
augmentation is resolved through *implicit conversion search*: all candidate
conversions in scope compete, and the one selected is the one that actually
provides an applicable `bar` for the receiver `Foo[Int]`. There is no
name-based shadowing between `FooIntExt` and `FooStringExt`; they are
disambiguated by the *extended type*. Extension methods, by contrast, are
resolved through ordinary lexical *name* resolution, where a closer binding
shadows an outer one purely by name, before the receiver type is consulted.

We have grown used to `implicit class` semantics, and they should be preserved.

### Why this matters beyond a toy example

- **Standard library migration.** As the standard library migrates from
  `implicit class`es to `extension` methods, any user code that defines an
  extension method whose name collides with a standard-library extension — but
  on a different type — could start failing to compile, where the equivalent
  `implicit class` code compiled fine.

- **Named tuples.** With the [Named Tuples](https://docs.scala-lang.org/sips/named-tuples.html)
  feature, it is natural for a library to add extension methods to *its* named
  tuple shapes while a user adds identically-named extension methods to *their*
  named tuple shapes. These are different structural types, and the same
  shadowing failure appears.

- **Real-world migration (IntelliJ Scala Plugin).** During an incremental
  Scala 2 → Scala 3 migration, splitting a large module into smaller ones
  naturally places top-level extensions in several files, many sharing a name.
  The current restriction forces the introduction of artificial wrapper objects
  purely to work around resolution, enlarging the migration diff and
  complicating the code structure with no user-facing benefit.

### The status-quo workaround and why it is not good enough

The idiomatic workaround is to wrap each group of extensions in a `given`
instance (optionally over a shared trait), so that resolution goes through the
implicit scope of the receiver type instead of lexical scoping:

```scala
object Stuff:
  opaque type Height = Int
  object Height:
    given syntax: {} with
      extension (a: Height) def add(b: Height): Height = a + b

  opaque type Width = Int
  object Width:
    given syntax: {} with
      extension (a: Width) def add(b: Width): Width = a + b

import Stuff.*
def addAll(l: List[Height]): Height = l.reduce(_.add(_))
```

This works, and it is the right tool when the author is *designing* a
type-class-like API. But as a general answer to "two same-named extensions on
different types" it is unsatisfactory:

- It requires anticipating the collision and structuring the code around it.
- The `{} with` / `AnyRef with` empty-refinement syntax is obscure and widely
  considered a wart (it is explicitly flagged as "kinda weird" by those who
  recommend it).
- It obscures the author's intent behind implicit mechanics that the reader must
  reverse-engineer.
- It does not help the migration scenarios above, where the collision is
  discovered *after the fact* and the goal is to keep the diff minimal.

The goal of this proposal is that the plain, obvious `extension` code — the code
a user writes when mechanically replacing an `implicit class` — simply works.

### Scope

This proposal is limited to the resolution of **extension methods** invoked in
selection (dotted) position, `e.m`. It does not change:

- resolution of ordinary (non-extension) method overloads across imports (this
  was explicitly a non-goal of SIP-54 and remains one here);
- resolution of extension methods invoked as plain calls `m(e)` (which remain
  governed by ordinary name resolution, exactly as SIP-54 left them);
- the behaviour when the closest applicable candidate is unique — existing
  programs that resolve today keep resolving to the same method.

## Proposed solution

### High-level overview

Extension-method resolution becomes **precedence-ordered with fallback**,
mirroring what SIP-54 already does within a single nesting level, but now across
levels:

1. Group all in-scope extension methods named `m` by their lexical binding
   precedence (innermost definition, then enclosing definitions, then named
   imports, then wildcard imports — i.e. the existing precedence order used by
   name resolution).
2. Consider the groups from highest to lowest precedence. For the current group,
   try to typecheck `m(e)` for each candidate.
   - If exactly one candidate applies (its `m(e)` typechecks), pick it.
   - If more than one candidate in the *same* group applies, report an
     ambiguity error (this is the SIP-54 rule, including its "a single
     non-wildcard import beats wildcard imports" refinement).
   - If no candidate in the group applies to the receiver, discard the whole
     group and move on to the next (lower-precedence) group.
3. If no group yields an applicable candidate, resolution fails with the same
   diagnostic it would produce today for the closest candidate.

Because a closer applicable candidate is always chosen over a farther one,
**ordinary shadowing is preserved whenever it is meaningful** (i.e. when the
closer method actually applies to the receiver). Fallback only happens when the
closer candidate could not have been called anyway. Applied to the running
example:

```scala
class Foo[T]
object Lib:
  extension (foo: Foo[Int]) def bar: Unit = {}

import Lib.*
extension (foo: Foo[String]) def bar: Unit = {}

val f = Foo[Int]()
f.bar // now: resolves to Lib's bar (Foo[Int]); the local bar does not apply
val g = Foo[String]()
g.bar // resolves to the local bar (Foo[String]), which shadows as before
```

And genuine shadowing (same receiver type) is unchanged:

```scala
object Lib:
  extension (foo: Foo[Int]) def bar: Int = 1

import Lib.*
extension (foo: Foo[Int]) def bar: Int = 2

Foo[Int]().bar // 2 — the local definition shadows the import, exactly as today
```

### Specification

This proposal amends the specification text introduced by SIP-54 in the
*"Translation of Calls to Extension Methods"* section. SIP-54's Step 1 reads (as
amended by that SIP):

> 1. The selection is rewritten to `m[Ts](e)` and typechecked, using the
>    following slight modification of the name resolution rules:
>    - If `m` is imported by several imports which are all on the same nesting
>      level, try each import as an extension method instead of failing with an
>      ambiguity. If only one import leads to an expansion that typechecks
>      without errors, pick that expansion. If there are several such imports,
>      but only one import which is not a wildcard import, pick the expansion
>      from that import. Otherwise, report an ambiguous reference error.

It is replaced by:

> 1. The selection is rewritten to `m[Ts](e)` and typechecked, using the
>    following modification of the name resolution rules. The candidate
>    references named `m` that are visible at the call site are partitioned into
>    *precedence levels*, using the ordinary binding-precedence order of name
>    resolution (a definition in an enclosing scope, a definition in a more
>    deeply nested scope, a name made available by a named import, and a name
>    made available by a wildcard import, with more deeply nested bindings taking
>    precedence over less deeply nested ones). The levels are examined from
>    highest to lowest precedence:
>    - Within the current level, each candidate is tried as an extension method.
>      If exactly one candidate leads to an expansion that typechecks without
>      errors, pick that expansion. If several candidates at this level do, but
>      only one of them is not a wildcard import, pick that one. If several
>      candidates at this level do and this rule does not disambiguate them,
>      report an ambiguous reference error.
>    - If no candidate at the current level leads to an expansion that
>      typechecks, the next lower precedence level is examined in the same way.
>    - If no level yields an applicable candidate, report the error arising from
>      the highest-precedence level (preserving today's diagnostics).
>
>    Only extension methods participate in this cross-level fallback; the
>    lexical precedence of a *non-extension* member named `m` is unaffected, and
>    a non-extension member continues to shadow extension methods named `m` in
>    enclosing scopes exactly as before.

The SIP-54 same-level rule is thus a special case of the general rule applied to
a single precedence level.

### Compatibility

The change is **backward source compatible** in the same sense SIP-54 was:

- It only affects selections `e.m` that resolve, at the highest-precedence
  level, to an extension method that *does not* typecheck against the receiver.
  Such selections are rejected today. The new rule can only turn a former
  compile error into a successful resolution; it never changes the meaning of a
  program that compiles today.
- When the highest-precedence extension candidate *does* apply, resolution stops
  at that level and picks exactly the method it picks today, so shadowing among
  applicable candidates is untouched.
- Non-extension members are entirely unaffected: a value, method, or field named
  `m` in a closer scope shadows outer extension methods `m` just as it does now.

No new syntax is introduced, so there is no change to the grammar. The change is
purely in the typer's name-resolution logic and produces the same kind of typed
trees as an equivalent fully-qualified extension call, so **binary and TASTy
compatibility are preserved by construction** — every resolution the new rule
produces could already be written by hand today as a fully-qualified
`qualifierPath.m(e)` call, and elaborates identically.

### Feature interactions

- **SIP-54 (multi-source extension overloads).** This proposal strictly
  generalises SIP-54: the same-nesting-level behaviour is retained as the
  single-level case, including the "one non-wildcard import wins" tie-breaker.

- **`given` instances / implicit scope.** Extension methods brought in through
  `given` instances and the implicit scope of the receiver type are resolved by
  a *separate* mechanism (implicit search) that already disambiguates by the
  extended type. This proposal does not alter that path; it only makes the
  *lexical* path behave consistently with it. When both a lexical extension and
  a given-provided extension are available, the existing ordering between
  lexical resolution and implicit search is preserved.

- **Overloading and specificity.** Within a single precedence level, if more
  than one candidate applies and the SIP-54 tie-breaker does not resolve it, the
  result is an ambiguity error rather than a silent choice. This proposal does
  *not* introduce most-specific-overload selection across levels; the closer
  level simply wins. This keeps the rule predictable and avoids surprising
  "action at a distance".

- **Error reporting.** When resolution ultimately fails, the diagnostic is taken
  from the highest-precedence level so the reported error matches the method the
  user most likely intended, as today.

### Other concerns

- **Compile-time cost.** The fallback search is only triggered on the error path
  — when the closest extension candidate does not apply. In that case the
  compiler performs additional resolution attempts at outer levels. This cost is
  bounded by the number of enclosing precedence levels that bind `m` and is not
  paid by programs whose closest candidate already applies (the common case).

- **Tooling.** IDEs and other tools that resolve `e.m` will need to adopt the
  same fallback rule to report the same target as the compiler. The rule is
  mechanical and local, so this should be a modest change.

### Open questions

- **Cross-level ambiguity vs. specificity.** Should two *equally applicable*
  candidates at *different* levels ever be considered ambiguous, or is
  "closer wins" always the right rule? This proposal takes the simplest,
  most backward-compatible stance (closer wins, matching current shadowing), but
  the alternative of a specificity-based comparison could be explored.

- **Overloaded extension methods at the same owner.** When a single scope
  defines several same-named extension methods (true overloads), the current
  implementation treats them as one precedence level. The interaction with the
  fallback rule for the sub-case where none of the overloads applies should be
  pinned down with tests.

## Alternatives

- **Do nothing / rely on the `given` workaround.** Keep steering users toward
  wrapping extensions in `given` instances. Rejected because it fails the
  migration use cases, requires foreknowledge of collisions, and relies on
  obscure syntax (`{} with`). It also perpetuates the surprising divergence from
  `implicit class` semantics.

- **Full implicit-style unification.** Route *all* extension-method resolution
  through implicit search, discarding lexical precedence entirely. This would
  most closely match `implicit class` behaviour but is a much larger, riskier
  change with broad compatibility implications (it could change which method is
  selected for programs that compile today). The precedence-ordered-fallback
  design deliberately keeps lexical precedence authoritative whenever it is
  meaningful, changing behaviour only where the status quo is a hard error.

- **Only relax when the closest binding is an import (not a definition).** A
  narrower variant that leaves definition-level shadowing untouched. Rejected
  because the primary reported cases (top-level `extension` definitions across
  split files; a locally-defined extension shadowing an imported one) involve
  *definitions* at the closest level, so this variant would not solve them.

## Related work

- [SIP-54: Multi-Source Extension Overloads](https://docs.scala-lang.org/sips/multi-source-extension-overloads.html)
  — the predecessor this proposal generalises.
- Pre-SIP discussion:
  [Relaxed extension methods (SIP 54) are not relaxed enough](https://contributors.scala-lang.org/t/relaxed-extension-methods-sip-54-are-not-relaxed-enough/6585).
- Earlier discussion on shadowing:
  [Change shadowing mechanism of extension methods for on-par implicit class behavior](https://contributors.scala-lang.org/t/change-shadowing-mechanism-of-extension-methods-for-on-par-implicit-class-behavior/5831).
- Scala 2 `implicit class`es, which already disambiguate augmentations by the
  extended type rather than by member name — the behaviour this proposal aims to
  restore for `extension` methods.
- A proof-of-concept implementation for the Scala 3 compiler (`dotc`) accompanies
  this draft; see the *Proposed solution → Specification* section for the
  resolution rule it realises.

## FAQ

**Does this make same-named extensions on the *same* type ambiguous?**
No. When two candidates both apply to the receiver, the closer one shadows the
farther one exactly as today; ambiguity is only reported for equally-applicable
candidates at the *same* precedence level (the SIP-54 rule).

**Can this change the method a currently-compiling program calls?**
No. If a program compiles today, its call already resolved to an applicable
closest candidate, and the new rule stops at that same candidate. Only programs
that are rejected today can change — from error to success.

**Why not just tell everyone to use `given` wrappers?**
Because the plain replacement of an `implicit class` by an `extension` should
work without redesigning the API around implicit instances, and because the
migration scenarios that motivate this proposal discover the collision after the
code is already written.
