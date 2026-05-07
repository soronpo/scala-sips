---
layout: sip
permalink: /sips/:number.html
redirect_from:
  - /sips/:number
  - /sips/:title.html
stage: design
status: under-review
number: 80
presip-thread: https://contributors.scala-lang.org/t/relative-scoping-for-hierarchical-adt-arguments/4136
title: "`#` Companion Shorthand"
---

**By: Oron Port**

## History

| Date           | Version                         |
|----------------|---------------------------------|
| May 5th 2026   | Leading `.` (Initial Draft)     |
| May 7th 2026   | Change to leading `#`           |

## Summary

We propose the `#` companion shorthand: a leading `#` followed by an identifier, used at term-level positions where the compiler knows the expected type `T`, that resolves to the named member of `T`'s companion object. When the compiler knows the expected type `T` at a given position, the user may write `#X` to mean "the term-level member `X` of `T`'s companion object."

```scala
val s = Shape(#Circle, #Red)

shape match
  case Shape(#Triangle | #Rectangle, _) => true
  case _                                => false
```

The desugaring rule is uniform: `#X` in a position with expected type `T` desugars to `T.X`, where `T`'s companion is searched for a term-level member named `X`. The resolved expression is then subject to the language's normal expected-type machinery, including implicit conversions; no constraint is placed on the candidate's declared result type. This single rule covers enums, sealed hierarchies, factory `def`s on companion objects, and opaque-type aliases.

This proposal is the outcome of an extended pre-SIP discussion ([Scala Contributors thread #4136](https://contributors.scala-lang.org/t/relative-scoping-for-hierarchical-adt-arguments/4136)) spanning 2020–2025.

## Motivation

When an algebraic data type is organised hierarchically — to keep the top-level namespace clean and group related cases — every construction and pattern match becomes verbose. Consider:

```scala
final case class Shape(geometry: Shape.Geometry, color: Shape.Color)
object Shape:
  sealed trait Geometry
  object Geometry:
    case object Triangle  extends Geometry
    case object Rectangle extends Geometry
    case object Circle    extends Geometry
  sealed trait Color
  object Color:
    case object Red   extends Color
    case object Green extends Color
    case object Blue  extends Color
```

Today, building or matching a `Shape` requires fully qualified paths:

```scala
val redCircle = Shape(Shape.Geometry.Circle, Shape.Color.Red)

shape match
  case Shape(Shape.Geometry.Triangle | Shape.Geometry.Rectangle, _) => true
  case _                                                            => false
```

There are three workarounds in current Scala, none satisfactory:

1. **Fully qualified names.** Verbose at every call site; the verbosity scales with hierarchy depth and is paid by every user, every time.
2. **Wildcard imports** (`import Shape.Geometry.*, Shape.Color.*`). These re-introduce exactly the namespace pollution the hierarchical organisation was designed to avoid, and they pollute *the rest of the enclosing block* — not just the one expression that needs them.
3. **Localised scoped imports.** Cleaner than top-level wildcard imports, but they don't compose inside argument-only or pattern-only positions: there is no way to scope an import to "just this argument" or "just this case alternative."

To make the cost of the localised-import workaround concrete, compare it side-by-side with the proposal:

```scala
// Localised imports — the closest current workaround:
shape match
  case s @ Shape(_, _) =>
    import Shape.Geometry.*, Shape.Color.*
    s match
      case Shape(Triangle | Rectangle, _) => true
      case _                              => false
  case _ => false
// Imports persist for the rest of this case body, polluting that scope.
// They cannot be scoped to a single | alternative or a single argument.

// With the `#` companion shorthand:
shape match
  case Shape(#Triangle | #Rectangle, _) => true
  case _                                => false
// No imports. The `#X` form is naturally scoped to the position it appears in.
```

The same pain appears beyond ADTs whenever a companion object holds factory members — for example HTML / DSL builder libraries that keep tag attributes, colours, and other constants on the companion object of the result type. In all of these cases, the call site already knows the expected type; what it lacks is a way to *reuse* that knowledge to look up the member by short name.

The goal of this proposal is to add exactly that: a syntactic affordance for "the `X` of the expected type." It is *not* a form of ambient name injection, and it does *not* introduce new name-resolution ambiguity in any expression that is currently valid.

## Proposed solution

### High-level overview

A leading `#` followed by an identifier `X` (no whitespace between `#` and `X`), appearing in a position where the compiler knows the expected type `T`, desugars to `T.X`. The resolved expression is then type-checked normally.

```scala
def describe(shape: Shape): String = ???

describe(Shape(#Circle, #Red))    // #Circle is resolved against the expected
                                  // type of Shape.apply's first parameter
                                  // (Shape.Geometry), so it desugars to
                                  // Shape.Geometry.Circle. Likewise #Red
                                  // desugars to Shape.Color.Red.

val c: Shape.Color = #Red         // #Red desugars to Shape.Color.Red

shape.color match
  case #Red | #Blue => "primary-ish"
  case _            => "other"
```

The mental model is uniform: *"`#X` is the `X` of whatever type is expected here."*

The feature applies to:

- **Enum cases** (the central motivating use case): `val c: Color = #Red`.
- **Plain companion-object members** — `val`s, `object`s, `def`s, anything term-level: `val o: Option[Int] = #empty`.
- **Factory methods with arguments**: `render(#color("red"))`. See *Application form* below.
- **Pattern matching** at every position where the expected type is known: `case #Red => ...`.
- **Transparent type aliases, `import`s, and `export`s** — the resolution rule dealiases before looking up the companion: `val c: MyColor = #Red` (where `type MyColor = Color`).
- **Opaque type aliases** (from outside the defining module): `val l: Level = #Info`.
- **`using` clauses**: `f(using #Red)`.
- **Companion `apply` factory methods via the `#(args)` shorthand** (optional extension — see *Optional extension: bare apply form*): `val s: Shape = #(#Circle, #Red)`.

It does *not* apply to:

- Positions where the expected type is unknown — a fresh, unconstrained type variable, or a bare expression statement.
- Expected types whose principal class component has no companion module — for example general union types `A | B` (where neither side is `Null`), function types `A => B`, or bare traits without companions. The form `T | Null` is the one supported union: it reduces to `T`.
- Type-argument positions.
- Anonymous given instances on the companion (these remain reachable through normal given resolution).

### Specification

#### Triggering positions

A leading `#` followed immediately (with no intervening whitespace) by an identifier `X` is parsed as `#`-companion-shorthand syntax wherever `#X` would otherwise be a syntax error — that is, anywhere in expression or pattern position. There is no lexical-context gate: `#X` has no current meaning at expression position in Scala (the only existing use of `#` is the type-level projection `T#X`), so the rule fires uniformly.

Triggering is determined solely by whether type inference establishes a non-trivial expected type `T` for the position. The feature applies in:

- arguments to an application (including arguments inside a `using` clause);
- the right-hand side of a `val`, `var`, or `def` with an explicit declared type;
- the right-hand side of an assignment to a typed location;
- an arm of a `match`, `if`, or `try` whose target type is known;
- an element of a typed collection literal;
- a pattern whose expected type (the scrutinee or extractor parameter type) is known.

If the expected type is unknown the use site is an error (see *Resolution rule*).

#### Resolution rule

Given an expected type `T` and identifier `X`, the compiler computes the *target type* by reducing `T` as follows, in order:

1. Strip prototype layers from `T` (i.e. unwrap any wrappers introduced by the type-checker for inference purposes — `T` becomes its underlying expected type).
2. Dealias transparent type aliases. (See *Aliases, imports, and exports* below.)
3. Drop dependent refinements.
4. Take the principal class component, if `T` is a refined or intersection type.
5. **Drop `Null` arms from `T | Null` / `Null | T` unions** (recursively). This is the only union form that reduces to a single principal class; `T | Null` is the explicit-nulls idiom for "nullable T," and `Null` has no companion members of its own. So `Color | Null`, `Null | Color`, and `(Color | Null) | Null` all reduce to `Color`.

The resulting type is the *target* `T'`. Resolution then proceeds:

- The compiler looks up `X` as a *term-level* member of `T'`'s companion object. For an opaque type alias, the alias's own companion is searched (the underlying type's companion is *not* consulted from outside the module that defines the alias).
- **No constraint is placed on the candidate member's declared result type.** The desugared form `T'.X` is then re-typed by the language's regular `Select` machinery, so:
  - implicit conversions are inserted to bridge the candidate's type to the surrounding expected type;
  - if `X` is overloaded in the companion, normal overload resolution applies at the use site after the desugaring;
  - the resolved member must not be an *anonymous* given (named givens are eligible).

  This is the key design point: the rule does not pre-filter by conformance, because the resolved member's type may differ from `T` and a `given Conversion` (or other conversion) is expected to bridge the gap. See the *implicit-conversion bridging* worked example below.
- **Only the static expected type's companion is searched, not its supertypes' companions.** This keeps resolution local and predictable. A user who needs a member from a supertype writes the chained form (see the *Animal* worked example).
- If `T'` has no companion module — for example an unconstrained abstract type, a type parameter, a bare trait without a companion, a function/SAM type whose `Function1` companion has no useful members, or a non-`Null` union type `A | B` (which has no principal class) — `#X` is a hard error at the use site, with a diagnostic pointing at the missing companion. There is no silent fallback to outer scope.

##### Nullable expected types

The `T | Null` carve-out is the only union form for which the shorthand fires. It exists because `T | Null` is the canonical representation of nullable references under explicit-nulls, and there is no ambiguity about which side carries the companion — `Null` has none. Other unions remain hard errors:

```scala
val c1: Color | Null   = #Red          // OK — reduces to Color
val c2: Null | Color   = #Red          // OK — order does not matter
def paint(c: Color | Null): Unit = ???
paint(#Red)                            // OK
paint(null)                            // also OK (existing rule)

val u: Color | Int     = #Red          // ERROR — no principal class
```

#### Aliases, imports, and exports

The dealiasing step ensures that the shorthand works through transparent type aliases, `import`-introduced type names, and `export`-introduced type names — including combinations:

```scala
object Lib:
  sealed trait Color
  object Color:
    case object Red   extends Color
    case object Blue  extends Color

object AliasUser:
  type MyColor = Lib.Color           // transparent type alias
  def foo(c: MyColor): Unit = ()

  val c1: MyColor = #Red             // dealias MyColor → Lib.Color → Lib.Color.Red
  foo(#Blue)
  foo(c = #Blue)                     // named argument; expected type is MyColor

object ImportUser:
  import Lib.Color                   // import the type into scope
  val c: Color = #Red                // resolves through Lib.Color's companion

object Facade:
  export Lib.Color                   // re-export

object FacadeClient:
  import Facade.Color
  val c: Color = #Red                // resolves the same way
  type Hue = Color                   // alias on top of an exported type
  val h: Hue = #Blue                 // ok
```

For *opaque* type aliases the rule is different: from outside the defining module, the alias is abstract, and `#X` resolves against the *alias's own* companion, not the underlying type's companion. This is shown in the *Conversion* example below.

#### Pattern matching

In a pattern position, `#X` is interpreted as `T.X` where `T` is the scrutinee type for that position (or, inside an extractor pattern, the corresponding extractor parameter type). It works inside `|` alternatives, nested patterns, and typed patterns:

```scala
val c: Color = ???
c match
  case #Red | #Blue => "primary"
  case #Green       => "secondary"
```

`case #Red` is shorthand for `case Color.Red` only when the pattern's expected type is statically `Color`.

#### Application form: `#X(args)` and `#X[T](args)`

When `#X` resolves to a callable companion member, parentheses (and optional type arguments) follow naturally: `#X(args)` desugars to `T.X(args)`. The natural example is a factory `def` on the companion:

```scala
trait Frag
object Frag:
  def color(name: String): Frag = ???
  def text(s: String): Frag     = ???

def render(frag: Frag): String = ???

render(#color("red"))    // desugars to: render(Frag.color("red"))
render(#text("hi"))      // desugars to: render(Frag.text("hi"))
```

#### Chaining: `#X.Y`

Permitted. Only the leading `#` triggers the shorthand; everything to its right is ordinary path selection on the resulting value. This naturally supports navigating through intermediate levels of a hierarchy when the expected type's companion holds an intermediate class. See the *Animal* worked example below.

#### `using` clauses

For consistency with regular argument clauses, the shorthand is allowed inside `using` argument clauses:

```scala
def f(using c: Color): Unit = ???
f(using #Red)               // #Red desugars against the using parameter's expected type
```

#### Type-argument position

Not supported in this SIP. `f[#IntList]` remains invalid. Type-position shorthand is left for a future proposal.

#### Overload resolution

At a `#X` position the rule is:

> After overload arity narrowing — which may leave multiple candidates when default arguments are involved — if all remaining candidates have the **same parameter type** at the `#X` position, resolve `#X` against that shared type and let normal overload selection disambiguate using the other arguments. If the candidates have **different** parameter types at that position, the use site is an error and the user disambiguates with the fully qualified name, a type ascription, or a non-overloaded wrapper.

This is the same flavour of rule that already governs target-typed `_` in Scala: `foo.bar(_ + 1)` resolves cleanly when `bar` has multiple overloads that all take a `Int => Int` as their first parameter, regardless of how the overloads differ elsewhere. The shorthand inherits the principle — shared-signature overloads are not ambiguous from the use site's perspective.

##### Examples

**Different declared arities, only one matches.** Arity narrowing already picks the unique candidate; nothing further is needed.

```scala
def bar(a: Animal): Unit            = ???
def bar(a: Animal, b: Animal): Unit = ???

bar(#Cat)              // OK — only the 1-arg overload matches.
bar(#Cat, #Dog)        // OK — only the 2-arg overload matches.
```

**Different declared arities, multiple match because of defaults.** When a default argument makes a higher-arity overload also viable for the call's actual argument count, the parameter types at the `#X` position must agree. They do here, so resolution proceeds:

```scala
def bar(a: Animal): Unit                           = ???
def bar(a: Animal, b: Animal = Animal.Dog): Unit   = ???

bar(#Cat)              // OK — both candidates have parameter 0 type Animal;
                       //      #Cat resolves to Animal.Cat. The two overloads
                       //      then disambiguate by argument count.
```

This pattern is common when an API evolves while maintaining binary compatibility (a single-parameter method gains a defaulted second parameter), and the shorthand should not break at the call site when that happens.

**Same-arity overloads with shared parameter type at the `#X` position.** No ambiguity at `#X` — overload selection happens via the other arguments:

```scala
def f(c: Color, x: Int):    Unit = ???
def f(c: Color, x: String): Unit = ???

f(#Red, 0)             // OK — both overloads have parameter 0 type Color;
                       //      #Red → Color.Red. Disambiguation happens
                       //      on the second argument.
f(#Red, "x")           // OK.
```

**Same-arity overloads with different parameter types at the `#X` position — error.** The user disambiguates:

```scala
def foo(a: Animal): Unit = ???
def foo(a: Color):  Unit = ???

foo(#Red)              // ERROR: candidates differ at the #X position.
foo((#Red: Color))     // OK — type ascription gives #Red an unambiguous expected type.

def fc(c: Color): Unit = foo(c)
fc(#Red)               // OK — non-overloaded wrapper.
```

#### Varargs

Varargs parameters work straightforwardly when the element type is known at the call site:

```scala
def palette(colors: Color*): Unit = ???

palette(#Red, #Green)                     // each #X has expected type Color.

def labelled(name: String, colors: Color*): Unit = ???

labelled("primaries", #Red, #Green)       // OK — the first argument's String type
                                          //      binds the first parameter, leaving
                                          //      the varargs to be typed as Color.
```

In pattern position, varargs extractors carry the element type from the scrutinee:

```scala
val cs: List[Color] = ???
cs match
  case List(#Red, _*)         => "starts with red"
  case List(#Red, #Green, _*) => "red then green"
  case _                      => "other"
```

#### Polymorphic inference

`#X` resolution **does not** contribute back to type-parameter inference, but it does *consume* a type parameter that has already been fixed by other means. So `#X` works inside any polymorphic call whose type parameter is constrained by:

- an explicit type argument,
- the surrounding expected type (for example a typed `val`, `def`, or argument-position parameter type),
- a sibling argument that already pins the parameter.

```scala
Seq[Color](#Red, #Green)                    // OK — explicit type argument.
List[Color](#Red, #Green)                   // OK.

val cs:  Seq[Color]            = Seq(#Red, #Green)    // OK — outer expected type
                                                      //      Seq[Color] pins A first.
val ls:  List[Color]           = List(#Red, #Green)   // OK — same.
val opt: Option[Color]         = Some(#Red)           // OK — Some.apply[A] picks
                                                      //      A = Color from the
                                                      //      outer expected type.
val e:   Either[String, Color] = Right(#Red)          // OK — same.
val o:   Option[Int]           = #empty               // OK — Option.empty[A] picks
                                                      //      A from the outer
                                                      //      expected type; #empty
                                                      //      is just a direct
                                                      //      companion lookup.

Seq(#Red, #Green)                           // ERROR — Seq.apply[A](xs: A*) has
                                            //         no other source of A; #X
                                            //         cannot drive inference.
```

The remaining failure case is the truly unconstrained one: a polymorphic call where `#X` would be the *sole* source of information about the type parameter. The inference machinery sees the `#X` position as having an unknown expected type and stops. Lifting this is future work and would require `#X` resolution to contribute candidate types back to inference rather than just consume an established expected type. It is the same underlying entanglement that the same-arity overloading restriction sidesteps.

#### Optional extension: bare apply form `#(args)`

This subsection describes a designated-optional addition to the core proposal. The core `#X` rule does *not* depend on it; the committee may accept the rest of SIP-80 without this extension. It is included here because it follows naturally from the `#` sigil and unlocks a high-frequency factory-method use case.

When the expected type is `T` and the user writes `#(args)`, this desugars to `T.apply(args)`. This covers the most common factory-method case where the companion's `apply` would otherwise have to be spelled out. It is the natural complement of `#X` — the unnamed-member case.

```scala
final case class Shape(geometry: Geometry, color: Color)

val s: Shape = #(#Circle, #Red)         // ⤳ Shape.apply(Geometry.Circle, Color.Red)

def render(shape: Shape): String = ???
render(#(#Circle, #Red))                // ⤳ render(Shape.apply(...))
```

Pattern form: `case #(geom, color)` desugars to `case Shape(geom, color)` when the scrutinee type is statically `Shape`.

The same overload-resolution / target-type rules apply: the expected type must be a known type with a companion that has a callable `apply`. Anonymous givens are still excluded.

**Why optional.** The core `#X` proposal is a single, narrow rule (target-typed name lookup on a companion). `#(args)` is a related but separate rule — same rationale, different syntax position — and reviewers may want to evaluate it on its own merits. Marking it optional lets the committee:

- accept both as a coordinated package (the recommended path), or
- accept only `#X` and defer `#(args)` to a follow-up SIP, or
- reject `#(args)` while keeping `#X`.

If the committee rejects the optional extension, only this subsection is removed; the rest of the SIP is unchanged. Worked examples that use `#(args)` should be edited to fall back to the explicit form (`Shape(#Circle, #Red)`).

#### Infix applications

Because `#` is a fresh starter token at expression position, the bare infix form `a op #X` parses cleanly:

```scala
sealed trait Color
object Color:
  case object Red  extends Color
  case object Blue extends Color

extension (c: Color)
  infix def mix(other: Color): Color = ???
  def |+|(other: Color): Color       = ???

val c: Color = ???

c mix #Red                // OK — #Red has expected type Color.
c |+| #Red                // OK — same.
c.mix(#Red)               // OK — method-call form.
```

No parens are required. This is one of the structural advantages of `#` over leading-dot variants discussed in *Alternatives*: `.X` after an operator collides with method-chain continuation, but `#X` does not.

#### Equality and inequality

`==` and `!=` are defined on `Any`, so the right operand's expected type is `Any` — which has no companion members named `Red`, `Blue`, etc. The shorthand therefore does not help with equality comparisons:

```scala
c == #Red                 // ERROR — typer: Any has no member Red.
c == Color.Red            // OK — the standard form.
```

This is a consequence of the resolution rule, not a special-case carve-out: `==` simply does not propagate a useful expected type to its right operand. Users continue to write the fully qualified form for equality. Library authors who want comparison-style use sites to participate in the shorthand can provide typed methods (for example `def matches(other: Color): Boolean`), which then behave like any other parameter:

```scala
extension (c: Color)
  infix def matches(other: Color): Boolean = c == other

c matches #Red            // OK — RHS expected type is Color.
c.matches(#Red)           // OK.
```

#### Grammar

The following productions are added (`SimpleExpr1` and `Pattern1` use the existing Scala 3 grammar names):

```
SimpleExpr1 ::= ... | '#' id [TypeArgs] [ArgumentExprs]
Pattern1    ::= ... | '#' id [TypeArgs] [ArgumentPatterns]
```

If the optional `#(args)` extension is accepted, also add:

```
SimpleExpr1 ::= ... | '#' ArgumentExprs                -- bare apply
Pattern1    ::= ... | '#' '(' [Patterns] ')'           -- bare apply pattern
```

No proviso. The parser sees `#` as the trigger directly; there is no chain-continuation collision to resolve.

#### Worked examples

**Example 1 — `Shape` ADT (the running motivation).**

```scala
final case class Shape(geometry: Shape.Geometry, color: Shape.Color)
object Shape:
  sealed trait Geometry
  object Geometry:
    case object Triangle  extends Geometry
    case object Rectangle extends Geometry
    case object Circle    extends Geometry
  sealed trait Color
  object Color:
    case object Red   extends Color
    case object Green extends Color
    case object Blue  extends Color

val redCircle: Shape = Shape(#Circle, #Red)

redCircle match
  case Shape(#Triangle | #Rectangle, _) => "edged"
  case Shape(#Circle, #Red)             => "stop sign"
  case _                                => "other"
```

The first parameter of `Shape.apply` has expected type `Shape.Geometry`, so `#Circle` desugars to `Shape.Geometry.Circle`. The second has expected type `Shape.Color`, so `#Red` desugars to `Shape.Color.Red`.

**Example 2 — chaining through nested levels.** When the expected type's companion holds an *intermediate* level of the hierarchy, only the first segment is resolved by the shorthand; subsequent segments are plain member selection on the value that segment yields.

```scala
sealed trait Animal
object Animal:
  sealed trait Mammal extends Animal
  sealed trait Bird   extends Animal
  object Mammal:
    case object Dog extends Mammal
    case object Cat extends Mammal
  object Bird:
    case object Parrot extends Bird
    case object Eagle  extends Bird

def describe(a: Animal): String = ???

// Construction at the Animal level — chain through Mammal:
describe(#Mammal.Dog)
// #Mammal desugars to Animal.Mammal (the companion object of the Mammal trait,
// resolved via Animal's companion). .Dog is then plain member selection on that
// companion object. Final: Animal.Mammal.Dog.

// What does NOT work:
describe(#Dog)
// ERROR: Dog is a member of Animal.Mammal's companion, not Animal's.
// The expected type is Animal, so the lookup is in Animal's companion only.
// Either chain explicitly: describe(#Mammal.Dog)
// or qualify:              describe(Animal.Mammal.Dog)

// Pattern matching at the Animal level:
val a: Animal = ???
a match
  case #Mammal.Dog => "woof"
  case #Bird.Eagle => "screech"
  case _           => "other"

// Pattern matching at the Mammal level — direct, no chaining needed:
val m: Mammal = ???
m match
  case #Dog | #Cat => "mammal noise"
```

The rule is uniform: only the leading `#` triggers the shorthand; everything to its right is ordinary path selection. The expected type at the use site determines *which* companion is searched.

**Example 3 — implicit-conversion bridging.** The resolution rule does not require the candidate's declared result type to equal the expected type, because the language already has machinery for converting between types. A typical case:

```scala
enum LogLevel:
  case INFO, WARN, ERROR

opaque type ParserLogLevel = LogLevel
object ParserLogLevel:
  export LogLevel.{INFO, WARN, ERROR}              // re-exports of type LogLevel
  given Conversion[LogLevel, ParserLogLevel] = identity

def parse(level: ParserLogLevel): Unit = ???

// Status quo:
parse(ParserLogLevel.INFO)
// ParserLogLevel.INFO has declared type LogLevel (not ParserLogLevel);
// the given Conversion[LogLevel, ParserLogLevel] coerces it.

// With the shorthand:
parse(#INFO)
// #INFO desugars to ParserLogLevel.INFO (still of type LogLevel),
// then the same given Conversion fires.
```

If the rule had instead required the candidate's result type to conform to `ParserLogLevel`, this case would silently fail and force the user back to fully qualified names — defeating the feature in exactly the case where opaque types and conversions matter most. Dropping the conformance restriction lets the existing conversion machinery do its job.

### Compatibility

#### Source compatibility

A leading `#` followed immediately by an identifier (or `(args)`, under the optional extension) is currently a parse error in expression and pattern position. The proposed rule is therefore strictly additive: no existing program changes meaning. There is no chain-continuation collision to resolve — `#` has no role in expression-position selection in current Scala (its only existing meaning is type-level projection `T#X`, which is a binary operator between two types and never appears as a leading token).

#### Binary and TASTy compatibility

The feature is a pure desugaring: `#X` becomes `T.X`. Emitted bytecode and TASTy are identical to writing the qualified form by hand. The desugaring may be performed at parse time or at typer time; doing it at typer time is recommended because it allows higher-quality error messages that refer back to the surface `#X` form. There are no new AST-node requirements that affect serialisation.

#### Migration

No migration is needed. Existing code continues to compile with identical semantics. New code may opt into the shorthand at any call site.

### Feature interactions

- **Implicit / given resolution.** `#X` resolves *before* implicit search. Givens cannot shadow companion members for the purposes of this rule. Anonymous givens are not eligible candidates (see Specification); named givens are.
- **Overload resolution.** `#X` resolves against the shared parameter type at its position across all overload candidates remaining after arity narrowing — including candidates that are only viable because of default arguments. If candidates disagree on the parameter type at the `#X` position, the use site is an error; the user disambiguates with the qualified name, a type ascription, or a non-overloaded wrapper. See *Overload resolution* in the Specification for examples.
- **Default and named arguments.** `f(color = #Red)` works because the named argument fixes the expected type to that parameter's type before `#X` resolution runs.
- **Polymorphic methods and type-parameter inference.** `#X` resolution consumes an established expected type but does not contribute back to type inference. If a type parameter is otherwise unconstrained, the user supplies an explicit type argument or annotates the surrounding expression. See *Varargs* and *Polymorphic inference* in the Specification.
- **Opaque types.** Explicitly supported. From outside the defining module the alias is abstract, so `#X` resolves against the alias's *own* companion (not the underlying type's companion). See the *Conversion* worked example.
- **Transparent type aliases, `import`, `export`.** Resolution dealiases before looking up the companion, so all three transparently participate. See *Aliases, imports, and exports* in the Specification.
- **Union types.** `Color | Int` has no principal class component, so `#X` is a hard error: there is no single companion to search. The user spells out the qualified name (or refines the expected type before assignment). The single exception is `T | Null` (in either order, including nested), which reduces to `T` — see *Nullable expected types* in the Specification.
- **SAM and function types.** `#X` of expected type `A => B` falls through to `Function1`'s companion, which has no useful members. The diagnostic is the standard "no member named `X`" error.
- **Path-dependent types.** For an expected type `e.T` where `T` is path-dependent, the companion of `e.T` is searched if one exists. Behaviour is consistent with non-path-dependent types.

### Other concerns

- **Reference implementation.** A working implementation is available as scala/scala3 PR [#25998](https://github.com/scala/scala3/pull/25998), gated behind the experimental language import `scala.language.experimental.hashCompanionShorthand` while the proposal is under review.
- **Implementation surface.** The change is small. The grammar gains one production each in expression and pattern positions (plus two more if the optional extension is accepted); the parser sees `#` as a fresh starter token; the typer gains a single resolution pass that reduces the expected type (prototype stripping, dealiasing, refinement dropping, principal-class extraction, `T | Null` reduction) and rewrites `#X` to a `Select` on the resulting type. There are no new TASTy nodes and no changes to encoding.
- **Tooling.** The same expected-type machinery that drives editors' "show inferred type" also resolves `#X` on hover; IDE features such as "go to definition" and "find usages" work on the desugared form. The presentation compiler offers completions for `#X` at any triggering position whose expected type has a companion: typing `#r⟨TAB⟩` in `val c: Color = #r⟨TAB⟩` lists `Red` and similar candidates. Bare `#⟨TAB⟩` (no identifier character yet) does not parse and does not produce SIP-80 suggestions; this matches Swift and Dart's behaviour for their analogous shorthand.
- **Cross-platform.** Pure desugaring; no JVM, JS, or Native specifics.

## Empirical analysis: how often would `#X` fire?

A common question for any language-feature SIP is *"how much existing Scala code would actually benefit from this?"* For SIP-80 we answer that quantitatively, by scanning real compiled code and counting, per call site, how many positions the proposed `#X` rule would fire at.

### Methodology

A scanner was built in two stages and is published as a working artefact at [`soronpo/scala3` — `claude/scala-repo-scanner-script-FXi3I`, directory `sip80-scanner/`](https://github.com/soronpo/scala3/tree/claude/scala-repo-scanner-script-FXi3I/sip80-scanner). The whole tool is reproducible: any reader can re-run it on any published Scala 3 jar.

- **Stage A — syntactic baseline.** A Python regex-based scanner that walks `.scala` source files and counts surface patterns where `T.X` *appears* as a fully qualified prefix in a position the SIP would target. Cheap, and useful as a sanity check, but it has known false positives (e.g. `Arbitrary(Arbitrary.arbitrary[Int])` looks like a `T.X` shortening but the parameter type is `Gen[Int]`, not `Arbitrary`) and false negatives (it cannot see bare identifiers brought in by wildcard imports, nor expected types resolved by inference rather than written in the source).

- **Stage B — TASTy-based scanner.** A Scala 3 program built on `scala.tasty.inspector.TastyInspector` that walks compiled `.tasty` files and, for each `Select(qual, name)` and `Ident(name)` tree, asks the precise question: *"is this position target-typed by a context whose principal class component (after SIP-80's reduction rules) has a companion module that owns this symbol?"* Stage B walks the typed tree, applies SIP-80's reduction rules (strip prototypes, dealias, drop `Null` arms, take principal class), and only counts a position as an incident when the rule would actually fire. It catches the patterns Stage A misses — bare identifiers resolved through wildcard imports, ascriptions, `if`/`else` branches typed by context, nested extractor patterns, method-call args whose param type isn't visible from a regex — and rejects Stage A's false positives.

The scanner records, for each incident:
- the source file and line,
- whether the call site is *prefix-based* (the user wrote `T.X`) or *import-based* (the user wrote bare `X` with a wildcard import in scope),
- the `chars_saved` figure: `text.length - (memberName.length + 1)` for prefix-based; `0` for import-based (SIP-80 would shorten the *file* by removing the import, not the use site itself).

Walked target-typed positions:

| Category | Source |
|---|---|
| `typed_decl`     | `val`/`var`/`def` with an explicit declared type, RHS |
| `default_arg`    | parameter default value (Scala 3 emits these in `*$default$N` synthetic getters) |
| `call_arg`       | method/constructor argument; the param's expected type drives resolution |
| `ascription`     | `(expr : T)` |
| `if_branch`      | the two branches of an `if`/`else` whose enclosing context provides the expected type |
| `match_case`     | top-level `case T.X` with the scrutinee type |
| `nested_pattern` | `Some(T.X)`, tuple patterns, etc.; component types are read from the unapply's signature |

The scanner has a 39-incident self-test fixture (`fixtures/expected.json`) that pins per-category and per-file counts; running `bash build.sh fixtures && python3 check.py` validates that the inspector's behaviour matches the spec before any real measurement is taken.

### Results — Scala 3 itself (3.8.3)

| Module                                   | Incidents | Chars saved | Import-based |
|------------------------------------------|----------:|------------:|-------------:|
| `scala-library:3.8.3`                    |       290 |       1,688 |           98 |
| `scala3-compiler_3:3.8.3`                |     1,010 |      12,295 |          343 |
| `scala3-presentation-compiler_3:3.8.3`   |       177 |       1,817 |            2 |
| `scaladoc_3:3.8.3`                       |        89 |         630 |            1 |
| `scala3-staging_3:3.8.3`                 |         2 |           4 |            1 |
| `tasty-core_3:3.8.3`                     |         1 |          15 |            0 |
| `scala3-tasty-inspector_3:3.8.3`         |         0 |           0 |            0 |
| **Scala 3 total**                        | **1,569** |  **16,449** |      **445** |

The compiler dwarfs the rest because it is a 100 kLOC codebase rich in ADT pattern matching, `Set.empty` / `List.empty` factory calls, and `Mode` / `CompileMode`-style flag enums — all positions where SIP-80 fires. A non-trivial fraction of `scala3-compiler`'s 343 import-based hits come from generated `semanticdb` and `scalajs-ir` code that wildcard-imports the case-object members of large enums.

### Results — DFiantHDL (a heavy DSL user)

DFiantHDL is the kind of project SIP-80 was designed for: a Scala 3 hardware-description language whose intermediate representation is a large, deeply nested algebraic data type.

| Module                            | Version | Incidents | Chars saved | Import-based |
|-----------------------------------|---------|----------:|------------:|-------------:|
| `dfhdl-core_3`                    | 0.17.0  |       223 |       1,823 |           44 |
| `dfhdl-internals_3`               | 0.17.0  |        17 |         126 |            0 |
| `dfhdl-compiler-ir_3`             | 0.17.0  |       296 |       2,295 |           39 |
| `dfhdl-compiler-stages_3`         | 0.17.0  |       216 |       2,138 |            2 |
| `dfhdl-platforms_3`               | 0.17.0  |        55 |         357 |           13 |
| `dfhdl-devices_3`                 | 0.12.0  |         5 |          79 |            0 |
| **DFHDL total**                   |         |   **812** |   **6,818** |       **98** |

The `dfhdl-compiler-ir` module alone has 90 top-level `case T.X` arms and 81 nested patterns that SIP-80 would shorten.

### Results — curated Scala 3 community build

The community-build run covers 49 active Scala 3 projects, 62 modules, in three categories (A: small libraries, B: medium, C: large / framework-scale).

| Category | Active projects | Modules | Incidents | Chars saved | Import-based |
|----------|----------------:|--------:|----------:|------------:|-------------:|
| A        |               4 |      11 |       335 |       2,111 |           54 |
| B        |              15 |      18 |     1,798 |       9,836 |          499 |
| C        |              30 |      33 |     3,826 |      31,128 |        1,613 |
| **Total** |        **49** |  **62** | **5,959** |  **43,075** |    **2,166** |

Roughly 6,000 SIP-80 firing positions across the curated Scala 3 community build, with **~36 % via bare identifiers visible only because a wildcard import is open** — exactly the namespace-pollution pattern this proposal lets users avoid. Top contributors include `scalaz-core` (1,483 incidents, 904 import-based — pervasive `import Foo._` typeclass-syntax patterns), `scalapb-runtime` (571 incidents, 16,295 chars saved — generated protobuf code with explicit types everywhere), `scalacheck` (353), `sconfig` (232), and `libretto-core` (219).

### Caveats

- **Chars-saved is a lower bound.** The figure measures the bytes removed from source per incident; it does not capture the corresponding `import` lines that become unnecessary. For import-based incidents (~36 % of the total) the use-site saving is recorded as 0 because the use site is already short — but the *file* gets shorter when the wildcard import is no longer needed.
- **TASTy-version coverage.** The scanner auto-detects each jar's TASTy major.minor (`detect-tasty-version.py` reads the first `.tasty` entry's header) and runs against the matching `scala3-tasty-inspector_3:3.<minor>.x`. Some libraries' transitive deps are not auto-resolvable on Maven Central in a TASTy-walkable form (e.g. some `cats-core_3` references); those are skipped or run with an extended classpath.
- **Source-position recovery.** Source paths are recovered via `pos.sourceFile.path` and a basename-fallback index. Atypical layouts can produce empty `source` columns (still counted as incidents but without per-line char savings).
- **Volume vs. density.** These figures count *absolute* incidents. A single user does not look at all 6,000 incidents at once — they look at one block of code. The density of incidents within hot files (DSL builders, ADT pattern-matchers, configuration enums) is the relevant measure of perceived noise reduction. The compiler's 1,010 incidents are spread across hundreds of files; DFHDL's 296 in `dfhdl-compiler-ir` are concentrated in a much smaller surface.

All findings are committed under `sip80-scanner/tasty/results-jars/` in the linked repository, with `findings-*.tsv` files containing `file:line:before:after` for every incident, so the numbers above are independently verifiable.

## Alternatives

The pre-SIP discussion and the open review on this proposal considered a wide range of alternative syntaxes and semantics. Each is summarised below with the reason it is not the recommended primary form.

### `.X` (leading dot)

```scala
val s = Shape(.Circle, .Red)
```

This was the original form proposed and is the syntax used by Swift and Dart 3. It is rejected as primary for three reasons specific to Scala:

1. **Method-chain continuation collision.** Scala 3's existing chain-continuation rule reads `someExpr` ↵ `  .Red` as `someExpr.Red`. Distinguishing relative-scope `.X` from chain-continuation `.X` requires a non-trivial lexical-context rule (a list of preceding tokens that can permit `.X` to start a fresh expression). This is the "one-pixel difference radically changes the meaning of the program" concern.
2. **No clean apply shorthand.** `.()` is meaningless, so the natural complement `*.apply(...)` cannot be reduced to a single sigil with `.`. The `#(args)` form (under the optional extension) cleanly extends `#X` to factories without any extra ceremony.
3. **Visual fragility.** A bare period at column 1 is hard to spot in review and is easily confused with end-of-sentence prose punctuation in inline documentation.

The chain-continuation problem alone is sufficient to favour `#`. `.X` remains a defensible secondary choice if the committee is willing to accept the lexical-context complexity.

### `*.X` (asterisk-dot)

```scala
val s = Shape(*.Circle, *.Red)
```

Proposed during review as a way to "represent the receiver" — read as "the implicit `*`-receiver, dot something." Rejected because `*` is overloaded with too many existing meanings (multiplication operator, `import foo.*`, varargs `xs*`, glob patterns), the form is three keystrokes instead of one (`#X`), and `*(args)` is already a valid expression so the bare-apply form is unavailable.

### `..X` (leading double-dot)

```scala
val s = Shape(..Circle, ..Red)
```

Also fully unambiguous (no current Scala expression begins with `..`). Rejected for visual heaviness and for the connotation of "ellipsis / more here," which is the wrong mental model for what is in fact a *narrowing* operation: "the `X` member of the expected type."

### Bare identifier `X` with priority-ranked companion fallback

```scala
val s = Shape(Circle, Red)        // would search Shape's apply parameter types
```

The most ergonomic option, but rejected on two grounds:

1. **Silent breakage on library evolution.** Adding a member to a sealed hierarchy's companion can change the meaning of downstream call sites that bind a same-named local, or introduce ambiguity errors at separate-compilation time.
2. **Priority-ranked fallback rules are a known footgun.** Scala's existing implicit-priority machinery has a long history of hard-to-diagnose bugs; importing the same style into ordinary name resolution would amplify the problem.

A sigil signals to the reader that the name is being resolved relative to the expected type. At scale this is worth one keystroke.

### `'X` (leading apostrophe)

Collides with prior `Symbol` literal idioms (Scala 2) and with quoted-syntax in metaprogramming.

### Opt-in modifier (e.g. `export`-style)

A scheme in which library authors must mark companion members as eligible for the shorthand.

Rejected because it fragments the ecosystem: if some libraries forget to mark members, the feature works inconsistently and users avoid relying on it. The motivating use case is making *existing* hierarchical ADTs ergonomic at the call site, including ADTs in libraries the consumer doesn't control. Requiring library cooperation defeats this. The feature should be governed by the consumer's explicit sigil at the use site, not by an upstream flag.

### Localised imports (status quo)

```scala
val s = { import Shape.Geometry.*, Shape.Color.*; Shape(Circle, Red) }
```

Insufficient: imports do not compose inside argument-only or pattern-only positions, they pollute the rest of the enclosing block, and they add ceremony at every site. See the side-by-side comparison in *Motivation*.

## Related work

The pre-SIP discussion ([Scala Contributors thread #4136](https://contributors.scala-lang.org/t/relative-scoping-for-hierarchical-adt-arguments/4136)) is the primary source of design rationale, alternatives considered, and community feedback. Two prior Scala SIPs that drive syntax from the target type — [SIP-58 (Named Tuples)](https://docs.scala-lang.org/sips/named-tuples.html) and [SIP-60 (Bind variables within alternative patterns)](https://docs.scala-lang.org/sips/alternative-bind-variables.html) — are useful style references.

A `#`-as-companion-placeholder design was previously floated within Scala's own pre-SIP discussions in 2024: see [aggregate-literals pre-SIP, post #98](https://contributors.scala-lang.org/t/pre-sip-a-syntax-for-aggregate-literals/6697/98). That discussion is in progress and no official SIP has been submitted as of yet; it is referenced here only to note that the choice of `#` for this kind of resolution has been independently considered within the Scala community.

### Cross-language survey

Target-typed enum / companion shorthand is a feature that has independently appeared, been adopted, or been formally proposed across a wide range of programming languages. The convergent design — *"in a position whose expected type is `T`, a leading sigil followed by a name resolves to a member of `T`'s static scope"* — is now a recognisable cross-language pattern.

The table below summarises the languages currently in scope, with status and the form they use; each is detailed below. "Production" means the feature ships in the released language; "proposal" means it has a public RFC / KEEP / pre-SIP that has not yet shipped.

| Language     | Form                       | Status                         | Notes |
|--------------|----------------------------|--------------------------------|-------|
| Swift        | `.case` / `.factory(args)` | Production (since Swift 1.0)   | Closest direct precedent; over a decade of use. |
| Dart 3.x     | `.case` / `.factory(args)` | Production (Dart 3.0+)         | Officially called *dot shorthands*. |
| Zig          | `.case` / `.{ … }`         | Production                     | *Enum literals* and *decl literals*; `.{...}` is the bare-apply analogue. |
| C# 12+       | `[ … ]`                    | Production                     | *Collection expressions*; target-typed sibling form. |
| Kotlin       | `[ … ]`                    | Proposed (KEEP-0416)           | Collection literals proposal; same target-typed apply spirit. |
| Rust         | `.case` / `.case(args)`    | Proposed (RFC #3444)           | Open RFC; pattern-match and expression positions. |
| OCaml        | `` `Tag x ``               | Production                     | Polymorphic variants — different surface, similar context-driven tag resolution. |
| Java         | (none)                     | —                              | Sealed types + pattern matching ship in modern Java; no target-typed shorthand exists or is proposed. |

The language-by-language details follow.

#### Swift — `.value`

Swift has had the implicit-member-expression form since the first public release; in target-typed contexts a leading `.` selects from the static scope of the expected type. It applies to enum cases, type-level constants, and factory members alike.

```swift
view.backgroundColor = .systemBackground
let publisher = URLSession.shared.dataTaskPublisher(for: .swiftBySundell)

let x: SomeClass = .shared.a.f()
let y: SomeClass? = .shared
let z: SomeClass = .sharedSubclass

enum NetworkResult {
    case success(data: Data)
    case failure(error: Error)
    case pending
}

switch result {
case .success(let data): handle(data)
case .failure(let error): log(error)
case .pending: break
}
```

Reference: [The Swift Programming Language — Implicit Member Expression](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions/#Implicit-Member-Expression).

#### Dart 3.x — *dot shorthands*

Dart 3.0 (2025) adopted the same leading-dot form, naming it *dot shorthands*. It works for enum values, named constructors (`.origin()`), and inside collection literals.

```dart
enum Status { none, running, stopped, paused }

class Point {
  final double x, y;
  const Point(this.x, this.y);
  const Point.origin() : x = 0.0, y = 0.0;
}

const Status defaultStatus  = .running;        // Instead of Status.running
const Point  myOrigin       = .origin();       // Instead of Point.origin()
const List<Point> keyPoints = [.origin(), .new(1.0, 1.0)];

enum LogLevel { debug, info, warning, error }

String colorCode(LogLevel level) =>
  switch (level) {
    .debug   => 'gray',
    .info    => 'blue',
    .warning => 'orange',
    .error   => 'red',
  };
```

Reference: [Dart language tour — Dot shorthands](https://dart.dev/language/dot-shorthands).

#### Zig — *enum literals* and *decl literals*

Zig combines the named-member shorthand (`.case`) and a bare-apply analogue (`.{ … }`):

```zig
const dir: Direction = .north;
var gpa: std.heap.GeneralPurposeAllocator(.{}) = .init;

const color: Color = .auto;
const result = switch (color) {
    .auto => false,
    .on   => true,
    .off  => false,
};

var arr: []bool = .{ true, false };
pub const highway: DeloreanOptions = .{
    .enable_flux_capacitor = false,
    .target_speed_mph      = 60,
};
```

Zig's `.{...}` is the closest precedent for SIP-80's *optional* `#(args)` apply form: in both, a sigil with no name reduces to "construct/apply against the expected type."

Reference: [Zig Language Reference — Enum Literals](https://ziglang.org/documentation/master/#Enum-Literals); [Zig issue #9938 (decl literals)](https://github.com/ziglang/zig/issues/9938).

#### C# — collection expressions and target-typed expressions

C# 12 (2023) introduced *collection expressions*: `[ ... ]` syntax desugars to a target-typed factory call.

```csharp
public IEnumerable<int> MaxDays =>
    [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];
```

C# also already has unqualified flag-enum members in flags-attribute positions (`type.GetMethod("Name", .Public | .Instance | .DeclaredOnly)`-style usage in newer style guides) and target-typed conditional expressions (`Option<int> opt = condition ? .None : .Some(42);`). The same general direction — let the expected type drive the constructor / member lookup — is consistently visible in C#'s evolution.

Reference: [C# collection expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions).

#### Kotlin — *collection literals* (proposal)

Kotlin's [KEEP-0416](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0416-collection-literals.md) proposes target-typed collection literals:

```kotlin
val list: MyCustomList<Int> = [1, 2]   // desugars to a target-typed factory call
```

Same family as Zig's `.{...}` and C#'s `[...]`: a sigilled (or fixed-bracket) form whose desugaring is determined by the expected type.

Reference: [Kotlin KEEP-0416 — Collection literals](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0416-collection-literals.md).

#### Rust — RFC #3444 (proposal)

Rust [RFC #3444](https://github.com/rust-lang/rfcs/pull/3444) proposes a leading-`.`-prefixed shorthand for `Self`-resolved variants in expression and pattern positions:

```rust
let s: Status = .Running;

match status {
    .Pending(progress) => ...,
    .Complete { data } => ...,
    .Failed            => ...,
}
```

The RFC is open at the time of writing. The status-quo Rust idiom is `Status::Running`, with `use Status::*;` as the wildcard-import workaround that mirrors Scala's `import Color.*`.

Reference: [Rust RFC #3444 — Member access shorthand](https://github.com/rust-lang/rfcs/pull/3444).

#### OCaml — polymorphic variants

OCaml's polymorphic variants use a different mechanism (the tag carries no namespace at all and is unified structurally), but share the spirit of inferring the tag's namespace from context:

```ocaml
let f x = match x with
  | `Red   -> 1
  | `Blue  -> 2
  | `Green -> 3
```

The reader does not see `Color.Red`; the namespace is implicit in the type that `x` flows into. The mechanism is more general than SIP-80 (any tag works against any context that admits it) but the cognitive ergonomics are the same.

Reference: [Real World OCaml — Polymorphic Variants](https://dev.realworldocaml.org/variants.html#polymorphic-variants).

#### Java — no comparable feature

Java has shipped sealed classes (JEP 409) and pattern matching for `switch` (JEP 441), which together cover the *use case* SIP-80 addresses — building and matching on hierarchical ADTs — but Java has not introduced any target-typed shorthand for enum constants or factory members. The idiomatic form remains fully qualified (`Status.RUNNING`) or wildcard-imported.

#### Convergent design

Across the languages above, three observations emerge:

1. **The user-facing surface is consistently `.case` (Swift, Dart, Rust RFC, Zig)** when a sigilled form is chosen for the named-member case. The leading-dot convention is the *de facto* standard. Scala's choice of `#` rather than `.` is driven by the chain-continuation collision specific to Scala 3's grammar (see *Alternatives → `.X`*); the underlying mental model is identical to the leading-dot family.
2. **The bare-apply form (`#(args)` in this proposal) has direct cousins** — Zig's `.{...}`, C#'s `[...]`, Kotlin's proposed `[...]`. Each language picks brackets/braces shaped by its own grammar; SIP-80's `#(args)` is the natural Scala-shaped variant.
3. **The semantics are stable across languages**: a sigil at a position whose expected type is `T` resolves to a member of `T`'s static scope, with the language's normal conversion / overload machinery applied afterwards. SIP-80's resolution rule is in this family.

This convergence is itself an argument that the feature has earned its place: distinct language design teams, working independently and with very different surrounding type systems, have repeatedly arrived at the same shorthand for the same recurring usability problem.

## FAQ

**Why not just `import T.*`?** Imports do not compose inside argument-only or pattern-only positions, and they pollute the rest of the enclosing block. The motivation for hierarchical ADTs is precisely to avoid this pollution.

**What about library evolution?** Adding a member to a companion is source-compatible unless it introduces a same-named candidate at an existing `#X` use site, in which case normal overload resolution applies (or the use site fails with a clear "ambiguous" error). This is rare and immediately diagnosable, unlike the bare-identifier alternative which can silently change meaning.

**Why not also look up `given` instances?** Anonymous givens are out of scope; this proposal is about static name lookup of *explicitly named* companion members. Named givens, like any other named term-level member, are eligible.

**Why a sigil rather than an opt-in modifier?** The feature must work uniformly across libraries the consumer doesn't control; site-explicit sigils achieve this, while a library-author flag would fragment the ecosystem.

**Why `#X` rather than `.X` or `..X`?** `#` is rare in Scala (only used at the type level for projection), so the term-level meaning is free; `#X` is the same length as `.X` (no verbosity downside); there is no chain-continuation collision (so no complex lexical-context rule); and `#` cleanly extends to a bare apply form `#(args)` (under the optional extension), which `.` cannot offer because `.()` is meaningless.

**Does this work with opaque types?** Yes — the alias's own companion is searched (from outside the defining module), and the implicit-conversion bridging example above shows the interaction with `given Conversion`.

**Does this work in `using` clauses?** Yes — `f(using #Red)` resolves `#Red` against the expected type of the `using` parameter, exactly as for a regular argument.
