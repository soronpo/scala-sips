---
layout: sip
number: NN
permalink: /sips/:number.html
redirect_from:
  - /sips/:number
  - /sips/type-macros.html
stage: pre-sip
status: under-review
presip-thread: https://contributors.scala-lang.org/t/pre-sip-custom-dependent-operation-types/5236
title: Type Macros
---

**By: Oron Port and Claude AI**

## History

| Date          | Version            |
|---------------|--------------------|
| Jun 16th 2026 | Initial Draft      |

## Summary

We propose **type macros**: a way to define a parameterized type alias whose
right-hand side is a *splice* of a macro implementation that computes a type at
compile time. A type macro looks like an ordinary parameterized type, but its
definition delegates to a metaprogram:

```scala
import scala.quoted.*

type From[T] <: AnyNamedTuple = ${ fromImpl[T] }

def fromImpl[T: Type](using Quotes): Type[? <: AnyNamedTuple] = ???
```

The application `From[X]` *reduces* to whatever type `fromImpl` returns, but only
once `X` is *concrete* (fully defined). Until then, `From[X]` behaves as an
abstract type bounded by its declared upper bound (`<: AnyNamedTuple` above),
exactly like an unreduced `compiletime.ops` type or a stuck match type.

The goal is to let library authors express type-level computations that today
can only be implemented by *modifying the compiler*. The two canonical examples
are the `scala.compiletime.ops.*` operations and `scala.NamedTuple.From`, both of
which are hard-coded intrinsics today. This proposal shows how `NamedTuple.From`
can be re-expressed as an ordinary library-level type macro, with no compiler
support specific to it.

This document is the result of the Pre-SIP discussion
["Custom Dependent Operation Types"][presip], in particular
[Jeremy Smith's suggestion (post #7)][post7] to expose type computation directly
through quotes and splices rather than through an indirection over `given`
instances.

## Motivation

Scala 3 already has rich type-level computation: match types, singleton types,
and the `scala.compiletime.ops.*` operations. These cover many cases, but they
share a common ceiling: **anything they cannot express must be added to the
compiler itself.**

### The status quo for `compiletime.ops`

The standard library declares operations such as:

```scala
// library/src/scala/compiletime/ops/int.scala
object int:
  infix type +[X <: Int, Y <: Int] <: Int
  infix type -[X <: Int, Y <: Int] <: Int
  infix type *[X <: Int, Y <: Int] <: Int
  // ...
```

These are *abstract* type aliases — they have no right-hand side. The actual
meaning lives in the compiler. When the type comparer normalizes a type, it
routes applied types through `TypeEval.tryCompiletimeConstantFold`, which
recognizes the operation by symbol and folds the arguments to a `ConstantType`:

```scala
// compiler/src/dotty/tools/dotc/core/TypeEval.scala
val constantType =
  if defn.isCompiletime_S(sym) then
    constantFold1(natValue, _ + 1)
  else if defn.isNamedTuple_From(sym) then
    fieldsOf
  else if owner == defn.CompiletimeOpsIntModuleClass then name match
    case tpnme.Plus  => constantFold2(intValue, _ + _)
    case tpnme.Minus => constantFold2(intValue, _ - _)
    case tpnme.Times => constantFold2(intValue, _ * _)
    // ...one case per operation...
```

Every operation must additionally be registered as an "intrinsic" by name and
owner in `Definitions.scala`:

```scala
// compiler/src/dotty/tools/dotc/core/Definitions.scala
final def isCompiletimeAppliedType(sym: Symbol)(using Context): Boolean =
  compiletimePackageOpTypes.contains(sym.name)
  && ( isCompiletime_S(sym)
    || isNamedTuple_From(sym)
    || sym.owner == CompiletimeOpsIntModuleClass    && compiletimePackageIntTypes.contains(sym.name)
    || sym.owner == CompiletimeOpsStringModuleClass && compiletimePackageStringTypes.contains(sym.name)
    // ...
  )
```

The consequence is that a library author who needs, say, a `Gcd[A, B]`,
`Sqrt[N]`, or a domain-specific bit-width computation has only two options:

1. Encode it laboriously with match types and recursion (when it is even
   expressible — many operations on `String`, `Double`, or structural
   reflection are not), or
2. Convince the compiler team to add a new intrinsic and ship a new compiler.

### The status quo for `NamedTuple.From`

`NamedTuple.From` is the clearest illustration that *structural* type computation
is not expressible in user code at all today. Its library declaration is, again,
an abstract alias:

```scala
// library/src/scala/NamedTuple.scala
/** A type specially treated by the compiler to represent all fields of a
 *  class argument `T` as a named tuple. Or, if `T` is already a named tuple,
 *  `From[T]` is the same as `T`.
 */
type From[T] <: AnyNamedTuple
```

and the whole behavior is a compiler intrinsic in `TypeEval`:

```scala
// compiler/src/dotty/tools/dotc/core/TypeEval.scala
def fieldsOf: Option[Type] =
  expectArgsNum(1)
  val arg = tp.args.head
  val cls = arg.classSymbol
  if MatchTypes.isConcrete(arg) && cls.is(CaseClass) then
    val fields = cls.caseAccessors
    val fieldLabels = fields.map: field =>
      ConstantType(Constant(field.name.toString))
    val fieldTypes = fields.map(arg.memberInfo)
    Some:
      defn.NamedTupleTypeRef.appliedTo:
        nestedPairs(fieldLabels) :: nestedPairs(fieldTypes) :: Nil
  else arg.widenDealias match
    case arg @ defn.NamedTuple(_, _) => Some(arg)
    case arg if arg.derivesFrom(defn.TupleClass) => /* label as _1, _2, ... */
    case _ => None
```

This computation — *given a concrete case class, reflect on its fields and build
a named tuple of (labels, types)* — is exactly the kind of thing the `Quotes`
reflection API can already do at the term level. The only reason it must live in
the compiler is that there is no surface mechanism to run a metaprogram *to
produce a type*. Note also the recurring pattern `MatchTypes.isConcrete(arg)`:
the intrinsic is careful to reduce **only when its argument is concrete**, which
is precisely the timing concern Jeremy Smith raised in the Pre-SIP.

### Why match types and term macros are not enough

- **Match types** can deconstruct types, but they are restricted by design
  (see [SIP-56][sip56]): the reduction must be specifiable through `baseType`
  and subtyping, so unrestrained computation (string manipulation, arithmetic,
  reflection over arbitrary members, calling into a solver) is out of scope.
- **Whitebox term macros** can compute a *value* of a refined type, but they
  cannot be used to name a type in a signature, a bound, or another type
  definition. You cannot write `def f(x: From[Person]): ...` by means of a term
  macro.

The missing capability is a *type-returning* metaprogram that participates in
type normalization. That is what this proposal adds.

#### Running example

Throughout the proposal we use the de-facto requirement that motivated the
Pre-SIP: a user-defined operation that the compiler does not know about. We use
two:

```scala
// 1. an arithmetic op the compiler does not provide
type Gcd[A <: Int, B <: Int] <: Int = ${ gcdImpl[A, B] }

// 2. structural reflection, i.e. NamedTuple.From re-implemented in a library
type From[T] <: AnyNamedTuple = ${ fromImpl[T] }
```

Today the first is impossible without a new compiler intrinsic, and the second
*is* a compiler intrinsic. The goal is to make both ordinary library code.

## Proposed solution

### High-level overview

A **type macro** is an upper-bounded parameterized type whose right-hand side is a
single top-level splice `${ impl[...] }`, where `impl` is a macro implementation
that returns a `scala.quoted.Type`. The mandatory bound (`<: Int` below) is what
licenses the splice syntax and types the unreduced form:

```scala
import scala.quoted.*

// the type macro definition
type Gcd[A <: Int, B <: Int] <: Int = ${ gcdImpl[A, B] }

// the implementation, compiled in a prior compilation unit (like any macro)
def gcdImpl[A <: Int : Type, B <: Int : Type](using Quotes): Type[? <: Int] =
  import quotes.reflect.*
  (TypeRepr.of[A], TypeRepr.of[B]) match
    case (ConstantType(IntConstant(a)), ConstantType(IntConstant(b))) =>
      def gcd(x: Int, y: Int): Int = if y == 0 then x else gcd(y, x % y)
      ConstantType(IntConstant(gcd(a, b))).asType
        .asInstanceOf[Type[? <: Int]]
```

Then, in user code:

```scala
val x: Gcd[12, 18] = 6   // Gcd[12, 18] reduces to 6
summon[Gcd[12, 18] =:= 6] // ok
```

The application `Gcd[12, 18]` **reduces** to the type returned by `gcdImpl`. The
type macro is just a type, so it composes with everything else:

```scala
type LcmViaGcd[A <: Int, B <: Int] = A * B / Gcd[A, B]   // uses ops + macro
val y: LcmViaGcd[4, 6] = 12
```

#### Lazy reduction

Reduction is **lazy / by-need**, mirroring `compiletime.ops` and match types: a
type macro application reduces only when *all type arguments it inspects are
concrete*. When arguments are still abstract, the application stays unreduced and
is treated as an abstract type with the **declared upper bound**:

```scala
def f[A <: Int, B <: Int]: Gcd[A, B] = ???
//                          ^^^^^^^^^ does not reduce here (A, B abstract);
//                                    typed as `<: Int`
val r = f[12, 18]   // now reduces to 6
```

This directly answers the timing concern from [post #7][post7]: *"you'd only
want to do that when the type argument becomes concrete."* The declared bound
(`<: Int`, `<: AnyNamedTuple`, ...) is mandatory precisely so that the
unreduced form is still usable in a signature.

### `NamedTuple.From` as a type macro

The point of the proposal is that `From` need not be a compiler intrinsic. Here
is the same behavior as a plain library type macro. It mirrors the compiler's
`fieldsOf` step-for-step, but uses the public reflection API:

```scala
import scala.quoted.*
import scala.NamedTuple.{NamedTuple, AnyNamedTuple}

object structural:

  /** All fields of a class `T` as a named tuple; identity on named tuples;
   *  positional labels (_1, _2, ...) for plain tuples. */
  type From[T] <: AnyNamedTuple = ${ fromImpl[T] }

  def fromImpl[T: Type](using Quotes): Type[? <: AnyNamedTuple] =
    import quotes.reflect.*
    val arg = TypeRepr.of[T].dealias

    // Build the type `NamedTuple[(labels...), (types...)]` from two lists.
    def named(labels: List[String], types: List[TypeRepr]): Type[? <: AnyNamedTuple] =
      val labelTuple = tupleOf(labels.map(l => ConstantType(StringConstant(l))))
      val typeTuple  = tupleOf(types)
      (labelTuple.asType, typeTuple.asType) match
        case ('[type ns <: Tuple; ns], '[type vs <: Tuple; vs]) =>
          Type.of[NamedTuple[ns, vs]]

    // right-nested pairs: T1 *: T2 *: ... *: EmptyTuple  (cf. compiler `nestedPairs`)
    def tupleOf(ts: List[TypeRepr]): TypeRepr =
      ts.foldRight(TypeRepr.of[EmptyTuple])((t, acc) => TypeRepr.of[*:].appliedTo(List(t, acc)))

    val sym = arg.typeSymbol
    if arg.isConcrete && sym.flags.is(Flags.Case) then        // a concrete case class
      val fields = sym.caseFields
      named(fields.map(_.name), fields.map(f => arg.memberType(f)))
    else arg.widen.dealias.asType match
      case '[type n <: Tuple; type v <: Tuple; NamedTuple[n, v]] => // already named
        Type.of[NamedTuple[n, v]]
      case _ if arg <:< TypeRepr.of[Tuple] =>                  // a plain tuple
        val elems = tupleElementTypes(arg)
        named(elems.indices.map(i => s"_${i + 1}").toList, elems)
      case _ =>
        report.errorAndAbort(s"Cannot derive NamedTuple.From for ${arg.show}")
```

Compared with the intrinsic in `TypeEval.fieldsOf`, the correspondence is exact:

| Compiler intrinsic (`TypeEval`)        | Type macro (`Quotes`)                |
|----------------------------------------|--------------------------------------|
| `MatchTypes.isConcrete(arg)`           | `arg.isConcrete`                     |
| `cls.is(CaseClass)`                    | `sym.flags.is(Flags.Case)`           |
| `cls.caseAccessors`                    | `sym.caseFields`                     |
| `arg.memberInfo(field)`                | `arg.memberType(field)`              |
| `nestedPairs(...)`                     | `tupleOf(...)` (right-nested `*:`)   |
| `defn.NamedTupleTypeRef.appliedTo(...)`| `Type.of[NamedTuple[ns, vs]]`        |
| returns `Option[Type]` (None = stuck)  | does not reduce when `!isConcrete`   |

With this, the special cases `isNamedTuple_From` in `Definitions.scala` and the
`fieldsOf` branch in `TypeEval.scala` would no longer be needed; `From` becomes
an ordinary entry in the standard library, indistinguishable from user code.

### Specification

#### Syntax

A type macro is a *bounded* type definition whose right-hand side is a single
splice naming a macro implementation. The splice right-hand side is licensed by
the **mandatory upper bound**: it is legal *only* when an explicit `<: U` is
present, and is rejected otherwise. This makes a type macro a direct sibling of a
match type, which already shares the very same `<: U = …` shape:

```scala
type Elem[X] <: Bound = X match { … }       // match type
type From[T] <: AnyNamedTuple = ${ impl[T] } // type macro
```

We extend the grammar of type definitions so that, *when an upper bound is
present*, the right-hand side may be a splice:

```
TypeDef            ::=  id [TypeParamClause] ‘<:’ Type ‘=’ TypeMacroSplice
TypeMacroSplice    ::=  ‘$’ ‘{’ Expr ‘}’
```

Constraints on a well-formed type macro definition:

1. The right-hand side must be exactly a single splice `${ implCall }`, and the
   spliced expression must be a single call to a macro-implementation method —
   *no surrounding type and no further composition*. This mirrors term macros,
   whose body is `inline def f = ${ impl(...) }` and not an arbitrary expression
   built around a splice. Splices nested inside a larger type expression (e.g.
   `List[${ impl[A] }]`) are **not** permitted.
2. An explicit upper bound `<: U` is **mandatory**: it both licenses the splice
   syntax and is the type of every *unreduced* application — what the rest of the
   program type-checks against before reduction. A splice right-hand side without
   a declared upper bound is a compile error. (`compiletime.ops` and `From` carry
   such a bound for the same reason.)
3. The spliced call must have type `scala.quoted.Type[? <: U]`; its witnessed
   bound must conform to the declared bound `U`.
4. The implementation is subject to the same staging rules as a term macro: it
   must be defined in a *previous* compilation unit/run and be available on the
   classpath when the type macro is reduced.

The implementation's signature has the shape:

```scala
def impl[T1 <: B1 : Type, ..., Tn <: Bn : Type](using Quotes): Type[? <: U]
```

i.e. one `Type[_]` context bound per type parameter of the macro, an
implicit/`using` `Quotes`, and a `Type[? <: U]` result whose witnessed bound
conforms to the declared bound `U`.

#### Reduction

A type macro application `M[A_1, ..., A_n]` participates in type normalization
exactly where `compiletime.ops` and match types do — through `Type#tryNormalize`
/ the `TypeComparer`:

```scala
// compiler/src/dotty/tools/dotc/core/Types.scala (existing dispatch)
def tryNormalize(using Context): Type = underlyingNormalizable match
  case mt: MatchType => mt.reduced.normalized
  case tp: AppliedType => tp.tryCompiletimeConstantFold   // <- type macros plug in here
  case _ => NoType
```

The reduction of `M[A_1, ..., A_n]` is defined as:

1. **Concreteness gate.** Let `args` be the type arguments that the macro
   inspects. If any required argument is not *concrete* (in the sense of
   `MatchTypes.isConcrete` — no abstract type members, type variables, or
   unreduced applications remain), the application **does not reduce**; it is
   treated as an abstract type with bounds `Nothing <: _ <: U`. (The compiler
   cannot know in advance which arguments the macro inspects, so in practice the
   gate requires *all* arguments to be concrete before invoking the macro; this
   is the conservative, deterministic choice.)
2. **Invocation.** Otherwise, the compiler synthesizes `Type.of[A_i]` for each
   argument and invokes `impl` through the existing macro interpreter
   (`Splicer` / `SpliceInterpreter`), supplying a `QuotesImpl`. This is the same
   machinery that runs term macros; the only difference is that the result is
   read back as a `Type` (a `TypeImpl` wrapping a `tpd.Tree` in type position)
   rather than an `Expr`.
3. **Result.** The witnessed type `R` of the returned `Type[R]` becomes the
   reduction of `M[A_1, ..., A_n]`. The compiler checks `R <: U`; if not, it is
   a compile error at the macro definition's contract.
4. **Caching.** The result is memoized per `runId` on the `AppliedType`, exactly
   as `tryCompiletimeConstantFold` already does, so a given application reduces
   at most once per run.

If the implementation throws or calls `report.errorAndAbort`, reduction fails
with that error, analogous to a `TypeError` thrown from constant folding.

#### Typing of unreduced applications

Before reduction (and permanently, if an argument is abstract), `M[A...]` is an
abstract type with upper bound `U`. Subtyping, member selection, and inference
all use `U`. This is identical to how an unreduced `1 + N` (with `N` abstract) is
treated as `<: Int`. Reduction is *monotone*: if `M[A...]` reduces to `R` for
concrete `A`, then it must continue to reduce to `R` for any more specific
arguments — a contract the macro author must uphold (see Compatibility).

#### The `Type`-returning macro API

No new library type is needed; `scala.quoted.Type[T]` already exists and
`quotes.reflect.TypeRepr#asType` already produces one. The proposal only requires
that the macro *runner* accept a `Type[_]`-typed result in type-definition
position. Concretely:

- The typer type-checks the bounded splice RHS, expecting `Type[? <: U]` instead
  of `Expr[T]`, and marks the implementation method as a macro.
- A `Splicer.spliceType` runs the interpreter expecting a
  `Quotes => scala.quoted.Type[?]` and reads the result back into a `TypeTree`
  via `PickledQuotes.quotedTypeToTree`; the resulting type is the reduction. (Both
  of these exist in the proof-of-concept implementation.)

### Compatibility

**Backward source compatibility.** The feature is purely additive: a splice in
type-definition position is currently a syntax error, so no existing program
changes meaning. Re-expressing `compiletime.ops` and `NamedTuple.From` as type
macros (if ever done) would be an *internal* refactor with no observable change
to user programs, since the reduced types are identical.

**Binary compatibility.** Type macros generate no new runtime artifacts; like
match types and `compiletime.ops`, they are fully erased. The macro
implementation is ordinary compiled code that already exists on the classpath.

**TASTy compatibility — the central concern.** Reduction of a type macro spans
TASTy files in the same way match-type reduction does (see [SIP-56][sip56]): a
type macro application may be pickled unreduced (when its arguments are abstract)
and reduced later, in a *different* compilation, once arguments become concrete.
For TASTy to remain stable we need:

* *Determinism*: the same concrete arguments must always yield the same result
  type, across compiler versions and runs.
* *Monotonicity*: a more specific (still concrete) argument must not change an
  already-established result.

Unlike match types, **the compiler cannot verify these properties**, because a
type macro runs arbitrary code. This is the same trust model as whitebox term
macros, whose pickled, refined result types already depend on the macro behaving
deterministically. We therefore make determinism and monotonicity a documented
**contract** of type macro authors, and recommend (see Open Questions) that the
*reduced* form be preferred in pickling wherever possible so downstream
compilations need not re-run the macro. When an unreduced application must be
pickled, the macro implementation must be on the downstream classpath, exactly as
for `inline`/macro dependencies today.

### Feature Interactions

- **Match types.** A type macro can appear in a match-type scrutinee or body;
  the macro reduces first (when concrete), then the match type reduces. A type
  macro is *not* subject to the match-type legality rules of SIP-56 because it is
  not a match type; its "specification" is its implementation. A useful idiom is a
  type macro with a `<: Nothing` bound used as a match-type branch to emit a
  custom compile error, the type-level analogue of `compiletime.error`:

  ```scala
  type Error[Msg <: String] <: Nothing = ${ errorImpl[Msg] } // aborts with Msg

  type AnInt[T] <: Int = T match
    case Int => T & Int
    case _   => Error["This is not an Int"]

  val ok:  AnInt[Int]    = 5    // reduces to Int; the Error branch is never taken
  val bad: AnInt[String] = ???  // error: "This is not an Int"
  ```

  Because the `case _` branch is only selected (and only then forced) for a
  non-`Int` scrutinee, `AnInt[Int]` reduces cleanly while `AnInt[String]` triggers
  the macro abort with a domain-specific message.
- **`compiletime.ops`.** Fully composable, since both reduce in the same
  `tryNormalize` path. `A * B / Gcd[A, B]` mixes intrinsics and a type macro.
- **`inline` / term macros.** A `transparent inline def` may return a value whose
  type mentions a type macro; the type macro reduces during the same
  post-inlining normalization. The staging requirement (impl in a prior run) is
  shared with term macros.
- **Implicit search / type class derivation.** Because `From[T]` reduces to a
  concrete `NamedTuple[...]`, given instances can be summoned for the reduced
  form, enabling structural derivation without the `From` intrinsic.
- **Variance & GADTs.** Type macro parameters are invariant for reduction
  purposes (the macro decides how arguments are used). An unreduced application
  uses its declared bound, so variance checking sees only `U`.
- **Separate compilation.** Identical model to macros: a type macro cannot be
  defined and used within the same compilation unit.

### Other concerns

- **Soundness.** Reduction must be a (partial) function of concrete arguments. A
  non-deterministic macro can break subtyping and erasure across TASTy
  boundaries, just as a misbehaving whitebox macro can. The contract is the
  mitigation; tooling could offer an opt-in determinism check by running the
  macro twice in debug builds.
- **Compile-time cost & termination.** A type macro can loop or be expensive.
  The same `-Xmacro-settings`/stack-guard protections that bound term-macro
  execution apply. Reduction is cached per `runId`.
- **Security.** Type macros run arbitrary code at compile time. This is not a new
  capability — term macros already do — but the surface for it grows. The
  existing `-Yno-...`/sandbox considerations for macros carry over unchanged.
- **Error reporting & IDE.** Because reduction calls user code, error positions
  should point at the *use site* with a macro-expansion trace, mirroring term
  macro diagnostics. Presentation-compiler reductions must be side-effect-free.

### Resolved decisions

- **No nested splices; no composition.** A type macro right-hand side is exactly
  one splice naming one macro-implementation call — `type M[A] <: U = ${ impl[A] }`
  — never a splice embedded in a larger type such as `List[${ impl[A] }]`. This
  matches term macros, whose body is `${ impl(...) }` and not an arbitrary
  expression around a splice, and it avoids the normalization-ordering questions
  that composed splices would raise. Composition is recovered the ordinary way, by
  *using* the type macro inside another type (`List[M[A]]`, `M[A] *: T`, …).

### Open questions

1. **Pickling strategy.** Should unreduced type-macro applications be picklable at
   all, or should the compiler require reduction at definition site (banning
   abstract-argument occurrences in pickled signatures)? Banning them sidesteps
   the cross-TASTy determinism risk entirely, at the cost of expressiveness.
2. **Which arguments gate reduction.** Can a macro declare that only *some*
   parameters must be concrete (e.g. via an annotation), to reduce earlier?
3. **Kind polymorphism.** `From[T]` takes a proper type; should type macros
   accept higher-kinded arguments (`T <: AnyKind`), and how does `Type[_]` carry
   the kind?

## Alternatives

### `@customOp` / `CustomOp` given instances (the original Pre-SIP)

The [Pre-SIP][presip] originally proposed:

```scala
final class customOp extends StaticAnnotation
erased trait CustomOp[T <: AnyKind, Args <: Tuple]:
  type Out

@customOp type Plus[A <: Int, B <: Int] <: Int
transparent inline given [A <: Int, B <: Int]: CustomOp[Plus, (A, B)] = ${ simplerMacro }
```

Here the compiler, on seeing an `@customOp` type, searches for a `CustomOp` given
and reads its `Out`. **Pros:** reuses implicit search; the `given` can be
constrained/overloaded per argument shape. **Cons:** an extra layer of
indirection (annotation + trait + given) for what is conceptually a function from
types to a type; the computation still ultimately lives in a `transparent inline`
macro, so it does not avoid macros — it wraps them. Post #7 explicitly proposes
type macros as the more direct alternative: *"this eliminates the need to do it
through a given."* This SIP follows that direction.

### Pure match types + `compiletime.ops`

Encode everything with match types. **Pros:** specifiable, no arbitrary code.
**Cons:** cannot express arithmetic beyond what `compiletime.ops` provides,
string/structural operations, or reflection over arbitrary class members; `From`
is provably not expressible (you cannot enumerate a case class's fields with a
match type). This is *why* `From` is a compiler intrinsic today.

### Whitebox term macros + opaque/abstract types

Compute a value of a refined type with a whitebox macro and project a type member
out of it. **Cons:** cannot be used to *name* a type in arbitrary positions
(bounds, other type definitions, parameter types) and forces a value-level
detour; ergonomics are poor and composition with `compiletime.ops` is lost.

### Keep adding compiler intrinsics

The status quo. **Cons:** does not scale; every new operation is a compiler
change, a new release, and a new entry in `TypeEval`/`Definitions`. The Pre-SIP
exists precisely because this does not scale.

## Related work

- **Pre-SIP discussion** that led to this proposal:
  [Custom Dependent Operation Types][presip], in particular
  [post #7 by Jeremy Smith][post7] proposing the
  `type MyType[A] = ${ myTypeImpl[A] }` syntax adopted here.
- **[SIP-56: Proper Specification for Match Types][sip56].** The cross-TASTy
  determinism analysis there is directly relevant; type macros face the same
  reduction-spans-TASTy problem but, unlike match types, cannot be made
  *specifiable* — only *contractual*.
- **[SIP-58: Named Tuples][sip58].** Introduced `NamedTuple` and the `From`
  intrinsic that this proposal re-expresses as a library type macro.
- **`scala.compiletime.ops`** — the existing intrinsic-based type computation
  this proposal generalizes (`compiler/.../core/TypeEval.scala`).
- **Scala 2 "type macros"** (Eugene Burmako, *macro paradise*) — an experimental,
  later-removed feature allowing macros in type position. This proposal is
  narrower: it is anchored on the existing quotes/splices staging model and on
  the `tryNormalize` reduction pipeline, and it is lazy (concreteness-gated)
  rather than eager.
- **Quotes & splices reflection API** (`scala.quoted.Type`, `Quotes`,
  `TypeRepr#asType`) — the machinery the implementation reuses without extension.
- **Proof-of-concept implementation.** A working prototype exists on the
  `claude/peaceful-pascal-0uy3li` branch of the compiler. It implements the heart
  of the proposal — the reduction engine — by wiring type-macro reduction into the
  existing `TypeEval`/`tryNormalize` pipeline (gated on `MatchTypes.isConcrete`)
  and reusing the term-macro interpreter (`Splicer`) to run a metaprogram that
  returns a `scala.quoted.Type`, read back as a type via
  `PickledQuotes.quotedTypeToTree`. With it, `From[Person]` reduces — entirely in
  library code — to `NamedTuple[("name", "age"), (String, Int)]`, demonstrating
  the SIP's central claim that `NamedTuple.From` need not be a compiler intrinsic;
  a companion `Gcd[A, B]` example shows a `compiletime.ops`-style operation the
  compiler does not provide, composing with the built-in ops (`A * B / Gcd[A, B]`).
  The reduction engine is surface-syntax-agnostic; the remaining work is the
  `type M[A] <: U = ${ impl[A] }` parser/typer plumbing, the TASTy pickling
  strategy, and the determinism safeguards discussed above.

## FAQ

**Is this not just match types with arbitrary code?**
Yes, deliberately. Match types are restricted so their reduction is specifiable
and TASTy-stable by construction. Type macros trade that guarantee for
expressiveness, putting the determinism burden on the author — the same trade-off
Scala already makes for whitebox term macros.

**Why require an explicit upper bound, and why does it gate the syntax?**
Two reasons, one practical and one syntactic. Practically, an application must be
typeable *before* it reduces (e.g. inside a generic method whose type parameters
are abstract); the bound is the type the rest of the program sees until concrete
arguments arrive, exactly as for `compiletime.ops` and `From`. Syntactically, the
bound is also what *licenses* the splice right-hand side: a `${ … }` type
right-hand side is only accepted when an upper bound is declared, and rejected
otherwise. This keeps the feature from claiming a keyword such as `inline` (which
is a term/tree concept, out of place on a type) and makes a type macro read as a
natural sibling of a match type, which already has the same `<: U = …` shape.

**Why not allow `${ … }` nested inside a larger type, or compose splices?**
For the same reason term macros don't: the body is a single macro call, not an
arbitrary expression around a splice. You compose by *using* the type macro inside
another type (`List[M[A]]`), not by embedding a splice inside a type.

**Does the macro re-run in every downstream project?**
Only when a downstream compilation encounters an *unreduced* application whose
arguments have just become concrete — the same situation in which match types and
`compiletime.ops` re-reduce across TASTy. Fully-reduced results are pickled as
ordinary types and never re-run.

[presip]: https://contributors.scala-lang.org/t/pre-sip-custom-dependent-operation-types/5236
[post7]: https://contributors.scala-lang.org/t/pre-sip-custom-dependent-operation-types/5236/7
[sip56]: https://docs.scala-lang.org/sips/match-types-spec.html
[sip58]: https://docs.scala-lang.org/sips/named-tuples.html
