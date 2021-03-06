# Sound non-nullable (by default) types with incremental migration

Author: leafp@google.com

Status: Draft

## CHANGELOG

2019.12.27
  - Update errors for switch statements.
  - Make it an error entirely to use the default `List` constructor in opted-in
    code.
  - Clarify that setter/getter assignability uses subtyping instead of
    assignability.

2019.12.17
  - Specify errors around definitely (un)assigned late variables.

2019.12.08
  - Allow elision of default value in abstract methods
  - **CHANGE** Allow operations on `Never` and specify the typing
  - Specify the type signature for calling Object methods on nullable types
  - Specify implicit conversion behavior
  - Allow potentially constant type variables in instance checks and casts
  - Specify the error thrown by the null check operator
  - Specify `fromEnvironment` and `Iterator.current` library breaking changes
  - Fix definition of strictly non-nullable

2019.12.03:
  - Change warnings around null aware operaters to account for legacy types.

2019.11.25:
  - Specified implicitly induced getters/setters for late variables.

2019.11.22
  - Additional errors and warnings around late variables

2019.11.21
  - Clarify runtime instance checks and casts.

2019.10.08
  - Warning to call null check operator on non-nullable expression
  - Factory constructors may not return null
  - Fix discussion of legacy `is` check
  - Specify flatten

2019.04.23:
  - Added specification of short-circuiting null
  - Added `e1?.[e2]` operator syntax

## Summary

This is the proposed specification for [sound non-nullable by default types](http://github.com/dart-lang/language/issues/110).
Discussion of this proposal should take place in [Issue 110](http://github.com/dart-lang/language/issues/110).

Discussion issues on specific topics related to this proposal are [here](https://github.com/dart-lang/language/issues?utf8=%E2%9C%93&q=is%3Aissue+label%3Annbd+)

The motivations for the feature along with the migration plan and strategy are
discussed in more detail in
the
[roadmap](https://github.com/dart-lang/language/blob/master/working/0110-incremental-sound-nnbd/roadmap.md).

This proposal draws on the proposal that Patrice Chalin wrote
up [here](https://github.com/dart-archive/dart_enhancement_proposals/issues/30),
and on the proposal that Bob Nystrom wrote
up
[here](https://github.com/dart-lang/language/blob/master/resources/old-non-nullable-types.md).


## Syntax

The precise changes to the syntax are given in an accompanying set of
modifications to the grammar in the formal specification.  This section
summarizes in prose the grammar changes associated with this feature.

The grammar of types is extended to allow any type to be suffixed with a `?`
(e.g. `int?`) indicating the nullable version of that type.

A new primitive type `Never`.  This type is denoted by the built-in type
declaration `Never` declared in `dart:core`.

The grammar of expressions is extended to allow any expression to be suffixed
with a `!`.

The modifier `late` is added as a built-in identifier.  The grammar of top level
variables, static fields, instance fields, and local variables is extended to
allow any declaration to include the modifer `late`.

The modifier `required` is added as a built-in identifier.  The grammar of
function types is extended to allow any named parameter declaration to be
prefixed by the `required` modifier (e.g. `int Function(int, {int?  y, required
int z})`.

The grammar of selectors is extended to allow null-aware subscripting using the
syntax `e1?.[e2]` which evaluates to `null` if `e1` evaluates to `null` and
otherwise evaluates as `e1[e2]`.

The grammar of cascade sequences is extended to allow the first cascade of a
sequence to be written as `?..` indicating that the cascade is null-shorting.

All of the syntax changes for this feature have been incorporated into
the
[formal grammar](https://github.com/dart-lang/language/blob/master/specification/dartLangSpec.tex),
which serves as the canonical reference for the grammatical changes.

### Grammatical ambiguities and clarifications.

#### Nested nullable types

The grammar for types does not allow multiple successive `?` operators on a
type.  That is, the grammar for types is nominally equivalent to:

```
type' ::= functionType
          | qualified typeArguments?

type ::= type' `?`?
```

#### Conditional expression ambiguities

Conditional expressions inside of braces are ambiguous between sets and maps.
That is, `{ a as bool ? - 3 : 3 }` can be parsed as a set literal `{ (a as bool)
? - 3 : 3 }` or as a map literal `{ (a as bool ?) - 3 : 3 }`.  Parsers will
prefer the former parse over the latter.

The same is true for `{ a is int ? - 3 : 3 }`.

The same is true for `{ int ? - 3 : 3 }` if we allow this.


## Static semantics

### Legacy types

The internal representation of types is extended with a type `T*` for every type
`T` to represent legacy pre-NNBD types.  This is discussed further in the legacy
library section below.

### Subtyping

We modify the subtyping rules to account for nullability and legacy types as
specified
[here](https://github.com/dart-lang/language/blob/master/resources/type-system/subtyping.md).
We write `S <: T` to mean that the type `S` is a subtype of `T` according to the
rules specified there.


We define `LEGACY_SUBTYPE(S, T)` to be true iff `S` would be a subtype of `T`
in a modification of the rules above in which all `?` on types were ignored, `*`
was added to each type, and `required` parameters were treated as optional.
This has the effect of treating `Never` as equivalent to `Null`, restoring
`Null` to the bottom of the type hierarchy, treating `Object` as nullable, and
ignoring `required` on named parameters.  This is intended to provide the same
subtyping results as pre-nnbd Dart.

Where potentially ambiguous, we sometimes write `NNBD_SUBTYPE(S, T)` to mean
the full subtyping relation without the legacy exceptions defined in the
previous paragraph.

### Upper and lower bounds

We modify the upper and lower bound rules to account for nullability and legacy
types as
specified
[here](https://github.com/dart-lang/language/blob/master/resources/type-system/upper-lower-bounds.md).  

### Type normalization

We define a normalization procedure on types which defines a canonical
representation for otherwise equivalent
types
[here](https://github.com/dart-lang/language/blob/master/resources/type-system/normalization.md).
This defines a procedure **NORM(`T`)** such that **NORM(`T`)** is syntactically
equal to **NORM(`S`)** modulo replacement of primitive top types iff `S <: T`
and `T <: S`.

### Future flattening

The **flatten** function is modified as follows:

**flatten**(`T`) is defined by cases on `T`:
  - if `T` is `S?` then **flatten**(`T`) = **flatten**(`S`)`?`
  - otherwise if `T` is `S*` then **flatten**(`T`) = **flatten**(`S`)`*`
  - otherwise if `T` is `FutureOr<S>` then **flatten**(`T`) = `S`
  - otherwise if `T <: Future` then let `S` be a type such that `T <: Future<S>`
and for all `R`, if `T <: Future<R>` then `S <: R`; then **flatten**('T') = `S`
  - otherwise **flatten**('T') = `T`

### Static errors
#### Nullability definitions

We say that a type `T` is **nullable** if `Null <: T` and not `T <: Object`.
This is equivalent to the syntactic criterion that `T` is any of:
  - `Null`
  - `S?` for some `S`
  - `S*` for some `S` where `S` is nullable
  - `FutureOr<S>` for some `S` where `S` is nullable
  - `dynamic`
  - `void`

Nullable types are types which are definitively known to be nullable, regardless
of instantiation of type variables, and regardless of any choice of replacement
for the `*` positions (with `?` or nothing).

We say that a type `T` is **non-nullable** if `T <: Object`.
This is equivalent to the syntactic criterion that `T` is any of:
  - `Never`
  - Any function type (including `Function`)
  - Any interface type except `Null`.
  - `S*` for some `S` where `S` is non-nullable
  - `FutureOr<S>` where `S` is non-nullable
  - `X extends S` where `S` is non-nullable
  - `X & S` where `S` is non-nullable

Non-nullable types are types which are either definitively known to be
non-nullable regardless of instantiation of type variables, or for which
replacing the `*` positions with nothing will result in a non-nullable type.

Note that there are types which are neither nullable nor non-nullable.  For
example `X extends T` where `T` is nullable is neither nullable nor
non-nullable.

We say that a type `T` is **strictly non-nullable** if `T <: Object` and not
`Null <: T`.  This is equivalent to the syntactic criterion that `T` is any of:
  - `Never`
  - Any function type (including `Function`)
  - Any interface type except `Null`.
  - `FutureOr<S>` where `S` is strictly non-nullable
  - `X extends S` where `S` is strictly non-nullable
  - `X & S` where `S` is strictly non-nullable

We say that a type `T` is **potentially nullable** if `T` is not non-nullable.
Note that this is different from saying that `T` is nullable.  For example, a
type variable `X extends Object?` is a type which is potentially nullable but
not nullable.  Note that `T*` is potentially nullable by this definition if `T`
is potentially nullable - so `int*` is not potentially nullable, but `X*` where
`X extends int?` is.  The potentially nullable types include all of the types
which are either definitely nullable, potentially instantiable to a nullable
type, or for which any migration results in a potentially nullable type.

We say that a type `T` is **potentially non-nullable** if `T` is not nullable.
Note that this is different from saying that `T` is non-nullable.  For example,
a type variable `X extends Object?` is a type which is potentially non-nullable
but not non-nullable.  Note that `T*` is potentially non-nullable by this
definition if `T` is potentially non-nullable.


#### Reachability

A number of errors and warnings are updated to take reachability of statements
into account.  Computation of code reachability
is
[specified separately](https://github.com/dart-lang/language/blob/master/resources/type-system/flow-analysis.md).

We say that a statement **may complete normally** if the specified control flow
analysis determines that any control flow path may reach the end of the
statement without returning, throwing an exception not caught within the
statement, breaking to a location outside of the statement, or continuing to a
location outside of the statement.

#### Errors and Warnings

It is an error to call a method, setter, getter or operator on an expression
whose type is potentially nullable and not `dynamic`, except for the methods,
setters, getters, and operators on `Object`.

It is an error to read a field or tear off a method from an expression whose
type is potentially nullable and not `dynamic`, except for the methods and
fields on `Object`.

It is an error to call an expression whose type is potentially nullable and not
`dynamic`.

It is an error if a top level variable, static variable, or instance field with
potentially non-nullable type has no initializer expression and is not
initialized in a constructor via an initializing formal or an initializer list
entry, unless the variable or field is marked with the `late` modifier.

It is an error if a potentially non-nullable local variable which has no
initializer expression and is not marked `late` is used before it is definitely
assigned (see Definite Assignment below).

It is an error if the body of a method, function, getter, or function expression
with a potentially non-nullable return type **may completely normally**.

It is an error if an optional parameter (named or otherwise) with no default
value has a potentially non-nullable type **except** in the parameter list of an
abstract method declaration.

It is an error if a required named parameter has a default value.

It is an error if a named parameter that is part of a `required` group is not
bound to an argument at a call site.

It is an error to call the default `List` constructor.

For the purposes of errors and warnings, the null aware operators `?.`, `?..`,
and `?.[]` are checked as if the receiver of the operator had non-nullable type.
More specifically, if the type of the receiver of a null aware operator is `T`,
then the operator is checked as if the receiver had type **NonNull**(`T`) (see
definition below).

It is an error for a class to extend, implement, or mixin a type of the form
`T?` for any `T`.

It is an error for a class to extend, implement, or mixin the type `Never`.

It is not an error to call or tear-off a method, setter, or getter, or to read
or write a field, on a receiver of static type `Never`.  Implementations that
provide feedback about dead or unreachable code are encouraged to indicate that
any arguments to the invocation are unreachable.

It is not an error to apply an expression of type `Never` in the function
position of a function call. Implementations that provide feedback about dead or
unreachable code are encouraged to indicate that any arguments to the call are
unreachable.

It is an error if the static type of `e` in the expression `throw e` is not
assignable to `Object`.

It is not an error for the body of a `late` field to reference `this`.

It is an error for a variable to be declared as `late` in any of the following
positions: in a formal parameter list of any kind; in a catch clause; in the
variable binding section of a c-style `for` loop, a `for in` loop, an `await
for` loop, or a `for element` in a collection literal.

It is an error for the initializer expression of a `late` local variable to use
a prefix `await` expression that is not nested inside of another function
expression.

It is an error for a class with a `const` constructor to have a `late final`
instance variable.

It is not a compile time error to write to a `final` variable if that variable
is declared `late` and does not have an initializer.

It is a compile time error to assign a value to a local variable marked `late`
and `final` when the variable is **definitely assigned**.  This includes all
forms of assignments, including assignments via the composite assignment
operators as well as pre and post-fix operators.

It is a compile time error to read a local variable marked `late` when the
variable is **definitely unassigned**. This includes all forms of reads,
including implicit reads via the composite assignment operators as well as pre
and post-fix operators.

It is an error if the object being iterated over by a `for-in` loop has a static
type which is not `dynamic`, and is not a subtype of `Iterable<dynamic>`.

It is an error if the type of the value returned from a factory constructor is
not a subtype of the class type associated with the class in which it is defined
(specifically, it is an error to return a nullable type from a factory
constructor for any class other than `Null`).

It is an error if any case of a switch statement except the last case (the
default case if present) **may complete normally**.  The previous syntactic
restriction requiring the last statement of each case to be one of an enumerated
list of statements (break, continue, return, throw, or rethrow) is removed.

Given a switch statement which switches over an expression `e` of type `T`,
where the cases are dispatched based on expressions `e0`...`ek`:
  - It is no longer required that the `ei` evaluate to instances of the same
    class.
  - It is an error if any of the `ei` evaluate to a value whose static type is
    not a subtype of `T`.
  - It is an error if any of the `ei` evaluate to constants for which equality
    is not primitive.
  - If `T` is an enum type, it is a warning if the switch does not handle all
    enum cases, either explicitly or via a default.
  - If `T` is `Q?` where `Q` is an enum type, it is a warning if the switch does
    not handle all enum cases and `null`, either explicitly or via a default.
  - It is a warning if a switch over a nullable type does not handle `null`
    either explicitly or via a default.

It is an error if a class has a setter and a getter with the same basename where
the return type of the getter is not a subtype of the argument type of the
setter.  Note that this error specifically requires subtyping and not
assignability and hence makes no exception for `dynamic`.

It is a warning to use a null aware operator (`?.`, `?..`, `??`, `??=`, or
`...?`) on an expression of type `T` if `T` is **strictly non-nullable**.

It is a warning to use the null check operator (`!`) on an expression of type
`T` if `T` is **strictly non-nullable** .

### Expression typing

It is permitted to invoke or tear-off a method, setter, getter, or operator that
is defined on `Object` on potentially nullable type.  The type used for static
analysis of such an invocation or tear-off shall be the type declared on the
relevant member on `Object`.  For example, given a receiver `o` of type `T?`,
invoking an `Object` member on `o` shall use the type of the member as declared
on `Object`, regardless of the type of the member as declared on `T` (note that
the type as declared on `T` must be a subtype of the type on `Object`, and so
choosing the `Object` type is a sound choice.  The opposite choice is not
sound).

Calling a method (including an operator) or getter on a receiver of static type
`Never` is treated by static analysis as producing a result of type `Never`.
Tearing off a method from a receiver of static type `Never` produces a value of
type `Never`.  Applying an expression of type `Never` in the function position
of a function call produces a result of type `Never`.


### Assignability

The definition of assignability is changed as follows.

A type `T` is **assignable** to a type `S` if `T` is `dynamic`, or if `S` is a
subtype of `T`.

### Generics

The default bound of generic type parameters is treated as `Object?`.

### Implicit conversions

The implicit conversion of integer literals to double literals is performed when
the context type is `double` or `double?`.

The implicit tear-off conversion which converts uses of instances of classes
with call methods to the tear-off of their `.call` method when the context type
is a function type is performed when the context type is a function type, or the
nullable version of a context type.

Implicit tear-off conversion is *not* performed on objects of nullable type,
regardless of the context type.  For example:

```dart
class C {
  int call() {}
}
void main() {
  int Function()? c0 = new C(); // Ok
  int Function()? c0 = (null as C?); // static error
  int Function()  c1 = (null as C?); // static error
}
```

### Const objects

The definition of potentially constant expressions is extended to include type
casts and instance checks on potentially constant types, as follows.

We change the following specification text:

```
\item An expression of the form \code{$e$\,\,as\,\,$T$} is potentially constant
  if $e$ is a potentially constant expression
  and $T$ is a constant type expression,
  and it is further constant if $e$ is constant.
```

to

```
\item An expression of the form \code{$e$\,\,as\,\,$T$} is potentially constant
  if $e$ is a potentially constant expression
  and $T$ is a potentially constant type expression,
  and it is further constant if $e$ is constant.
```

where the definition of a "potentially constant type expression" is the same as
the current definition for a "constant type expression" with the addition that a
type variable is allowed as a "potentially constant type expression".

This is motivated by the requirement to make downcasts explicit as part of the
NNBD release.  Current constant evaluation is permitted to evaluate implicit
downcasts involving type variables.  Without this change, it is difficult to
change such implicit downcasts to an explicit form.  For example this class is
currently valid Dart code, but is invalid after the NNBD restriction on implicit
downcasts because of the implied downcast on the initialization of `w`:


```dart
const num three = 3;

class ConstantClass<T extends num> {
  final T w;
  const ConstantClass() : w = three /* as T */;
}

void main() {
  print(const ConstantClass<int>());
}
```

With this change, the following is a valid migration of this code:

```dart
const num three = 3;

class ConstantClass<T extends num> {
  final T w;
  const ConstantClass() : w = three as T;
}

void main() {
  print(const ConstantClass<int>());
}
```

### Null promotion

The machinery of type promotion is extended to promote the type of variables
based on nullability checks subject to the same set of restrictions as normal
promotion.  The relevant checks and the types they are considered to promote to
are as follows.

A check of the form `e == null` or of the form `e is Null` where `e` has static
type `T` promotes the type of `e` to `Null` in the `true` continuation, and to
**NonNull**(`T`) in the
`false` continuation.

A check of the form `e != null` or of the form `e is T` where `e` has static
type `T?` promotes the type of `e` to `T` in the `true` continuation, and to
`Null` in the `false` continuation.

The static type of an expression `e!` is **NonNull**(`T`) where `T` is the
static type of `e`.

The **NonNull** function defines the null-promoted version of a type, and is
defined as follows.

- **NonNull**(Null) = Never
- **NonNull**(_C_<_T_<sub>1</sub>, ... , _T_<sub>_n_</sub>>) = _C_<_T_<sub>1</sub>, ... , _T_<sub>_n_</sub>>  for class *C* other than Null (including Object).
- **NonNull**(FutureOr<_T_>) = FutureOr<_T_>
- **NonNull**(_T_<sub>0</sub> Function(...)) = _T_<sub>0</sub> Function(...)
- **NonNull**(Function) = Function
- **NonNull**(Never) = Never
- **NonNull**(dynamic) = dynamic
- **NonNull**(void) = void
- **NonNull**(_X_) = X & **NonNull**(B), where B is the bound of X.
- **NonNull**(_X_ & T) = X & **NonNull**(T)
- **NonNull**(_T_?) = **NonNull**(_T_)
- **NonNull**(_T_\*) = **NonNull**(_T_)

#### Extended Type promotion, Definite Assignment, and Reachability

These are extended as
per
[separate proposal](https://github.com/dart-lang/language/blob/master/resources/type-system/flow-analysis.md).

## Runtime semantics

### Runtime type equality operator

Two objects `T1` and `T2` which are instances of `Type` (that is, runtime type
objects) are considered equal if and only if the runtime type objects `T1` and
`T2` corresponds to the types `S1` and `S2` respectively, and the normal forms
**NORM(`S1`)** and **NORM(`S2`)** are syntactically equal up to equivalence of
bound variables and **ignoring `*` modifiers on types**.  So for example, the
runtime type objects corresponding to `List<int>` and `List<int*>` are
considered equal.  Note that we do not equate primitive top types.  `List<void>`
and `List<dynamic>` are still considered distinct runtime type objects.  Note
that we also do not equate `Never` and `Null`, and we do not equate function
types which differ in the placement of `required` on parameter types.  Because
of this, the equality described here is not equivalent to syntactic equality on
the `LEGACY_ERASURE` of the types.


### Const evaluation and canonicalization

In weak checking mode, all generic const constructors and generic const literals
are treated as if all type arguments passed to them were legacy types (both at
the top level, and recursively over the structure of the types), regardless of
whether the constructed class is defined in a legacy library or not, and
regardless of whether the constructor invocation or literal occurs in a legacy
library or not.  This ensures that const objects continue to canonicalize
consistently across legacy and opted-in libraries.

In strong checking mode, all generic const constructors and generic const
literals are evaluated using the actual type arguments provided, whether legacy
or non-legacy.  This ensures that in strong checking mode, the final consistent
semantics are obeyed.

### Null check operator

An expression of the form `e!` evaluates `e` to a value `v`, throws a runtime
error which is an instance of `CastError` if `v` is `null`, and otherwise
evaluates to `v`.

### Null aware operator

The semantics of the null aware operator `?.` are defined via a source to source
translation of expressions into Dart code extended with a let binding construct.
The translation is defined using meta-level functions over syntax.  We use the
notation `fn[x : Exp] : Exp => E` to define a meta-level function of type `Exp
-> Exp` (that is, a function from expressions to expressions), and similarly
`fn[k : Exp -> Exp] : Exp => E` to define a meta-level function of type `Exp ->
Exp -> Exp`.  Where obvious from context, we elide the parameter and return
types on the meta-level functions.  The meta-variables `F` and `G` are used to
range over meta-level functions.  Application of a meta-level function is
written as `F[p]` where `p` is the argument.

The null-shorting translation of an expression `e` is meta-level function `F` of
type `(Exp -> Exp) -> Exp` which takes as an argument the continuation of `e` and
produces an expression semantically equivalent to `e` with all occurrences of
`?.` eliminated in favor of explicit sequencing using a `let` construct.

Let `ID` be the identity function `fn[x : Exp] : Exp => x`.

The expression translation of an expression `e` is the result of applying the
null-shorting translation of `e` to `ID`.  That is, if `e` translates to `F`,
then `F[ID]` is the expression translation of `e`.

We use `EXP(e)` as a shorthand for the expression translation of `e`.  That is,
if the null-shorting translation of `e` is `F`, then `EXP(e)` is `F[ID]`.

We extend the expression translation to argument lists in the obvious way, using
`ARGS(args)` to denote the result of applying the expression translation
pointwise to the arguments in the argument list `args`.

We use three combinators to express the translation.

The null-aware shorting combinator `SHORT` is defined as:
```
  SHORT = fn[r : Exp, c : Exp -> Exp] =>
              fn[k : Exp -> Exp] : Exp =>
                let x = r in x == null ? null : k[c[x]]
```

where `x` is a fresh object level variable.  The `SHORT` combinator is used to
give semantics to uses of the `?.` operator.  It is parameterized over the
receiver of the conditional property access (`r`) and a meta-level function
(`c`) which given an object-level variable (`x`) bound to the result of
evaluating the receiver, produces the final expression.  The result is
parameterized over the continuation of the expression being translated.  The
continuation is only called in the case that the result of evaluating the
receiver is non-null.

The shorting propagation combinator `PASSTHRU` is defined as:
```
  PASSTHRU = fn[F : (Exp -> Exp) -> Exp, c : Exp -> Exp] =>
               fn[k : Exp -> Exp] : Exp => F[fn[x] => k[c[x]]]
```

The `PASSTHRU` combinator is used to give semantics to expression forms which
propagate null-shorting behavior.  It is parameterized over the translation `F`
of the potentially null-shorting expression, and over a meta-level function `c`
which given an expression which denotes the value of the translated
null-shorting expression produces the final expression being translated.  The
result is parameterized over the continuation of the expression being
translated, which is called unconditionally.

The null-shorting termination combinator TERM is defined as:
```
  TERM = fn[r : Exp] => fn[k : Exp -> Exp] : Exp => k[r]
```

The `TERM` combinator is used to give semantics to expressions which neither
short-circuit nor propagate null-shorting behavior.  It is parameterized over
the translated expression, and simply passes on the expression to its
continuation.

- A property access `e?.f` translates to:
  - `SHORT[EXP(e), fn[x] => x.f]`
- If `e` translates to `F` then `e.f` translates to:
  - `PASSTHRU[F, fn[x] => x.f]`
- A null aware method call `e?.m(args)` translates to:
  - `SHORT[EXP(e), fn[x] => x.m(ARGS(args))]`
- If `e` translates to `F` then `e.m(args)` translates to:
  - `PASSTHRU[F, fn[x] => x.m(ARGS(args))]`
- If `e` translates to `F` then `e(args)` translates to:
  - `PASSTHRU[F, fn[x] => x(ARGS(args))]`
- If `e1` translates to `F` then `e1?.[e2]` translates to:
  - `SHORT[EXP(e1), fn[x] => x[EXP(e2)]]`
- If `e1` translates to `F` then `e1[e2]` translates to:
  - `PASSTHRU[F, fn[x] => x[EXP(e2)]]`
- The assignment `e1?.f = e2` translates to:
  - `SHORT[EXP(e1), fn[x] => x.f = EXP(e2)]`
- The other assignment operators are handled equivalently.
- If `e1` translates to `F` then `e1.f = e2` translates to:
  - `PASSTHRU[F, fn[x] => x.f = EXP(e2)]`
- The other assignment operators are handled equivalently.
- If `e1` translates to `F` then `e1?.[e2] = e3` translates to:
  - `SHORT[EXP(e1), fn[x] => x[EXP(e2)] = EXP(e3)]`
- The other assignment operators are handled equivalently.
- If `e1` translates to `F` then `e1[e2] = e3` translates to:
  - `PASSTHRU[F, fn[x] => x[EXP(e2)] = EXP(e3)]`
- The other assignment operators are handled equivalently.
- A cascade expression `e..s` translates as follows, where `F` is the
    translation of `e` and  `x` and `y` are fresh object level variables:
    ```
        fn[k : Exp -> Exp] : Exp =>
           F[fn[r : Exp] : Exp => let x = r in
                                  let y = EXP(x.s)
                                  in k[x]
           ]
    ```
- A null-shorting cascade expression `e?..s` translates as follows, where `x`
    and `y` are fresh object level variables.
    ```
       fn[k : Exp -> Exp] : Exp =>
           let x = EXP(e) in x == null ? null : let y = EXP(x.s) in k(x)
    ```
- All other expressions are translated compositionally using the `TERM`
  combinator.  Examples:
  - An identifier `x` translates to `TERM[x]`
  - A list literal `[e1, ..., en]` translates to `TERM[ [EXP(e1), ..., EXP(en)] ]`
  - A parenthesized expression `(e)` translates to `TERM[(EXP(e))]`

### Late fields and variables

A non-local `late` variable declaration _D_ implicitly induces a getter
into the enclosing scope.  It also induces an implicit setter iff one of the
following conditions is satisfied:

  - _D_ is non-final.
  - _D_ is late, final, and has no initializing expression.

The late final variable declaration with no initializer is special in that it
is the only final variable which can be the target of an assignment.  It
can only be assigned once, but this is enforced dynamically rather than
statically.

A read of a field or variable which is marked as `late` which has not yet been
written to causes the initializer expression of the variable to be evaluated to
a value, assigned to the variable or field, and returned as the value of the
read.
  - If there is no initializer expression, the read causes a runtime error to be
    thrown which is an instance of `LateInitializationError`.
  - Evaluating the initializer expression may validly cause a write to the field
    or variable, assuming that the field or variable is not final.  In this
    case, the variable assumes the written value.  The final value of the
    initializer expression overwrites any intermediate written values.
  - Evaluating the initializer expression may cause an exception to be thrown.
    If the variable was written to before the exception was thrown, the value of
    the variable on subsequent reads is the last written value.  If the variable
    was not written before the exception was thrown, then the next read attempts
    to evaluate the initializer expression again.
  - If a variable or field is read from during the process of evaluating its own
    initializer expression, and no write to the variable has occurred, the read
    is treated as a first read and the initializer expression is evaluated
    again.

Let _D_ be a `late` and `final` non-local variable declaration named `v`
without an initializing expression.  It is a run-time error, throwing an
instance of `LateInitializationError`, to invoke the setter `v=` which is
implicitly induced by _D_ if a value has previously been assigned to `v`
(which could be due to an initializing formal or a constructor initializer
list, or due to an invocation of the setter).

Let _D_ be a `late` and `final` local variable declaration named `v`
without an initializing expression.  It is a run-time error, throwing an
instance of `LateInitializationError`, to assign a value to `v` if a value
has previously been assigned to `v`.

Note that this includes the implicit initializing writes induced by
evaluating the initializer during a read.  Hence, the following program
terminates with a `LateInitializationError` exception.

```dart
int i = 0;
late final int x = i++ == 0 ? x + 1 : 0;
void main() {
  print(x);
}
```

A toplevel or static variable with an initializer is evaluated as if it
was marked `late`.  Note that this is a change from pre-NNBD semantics in that:
  - Throwing an exception during initializer evaluation no longer sets the
    variable to `null`
  - Reading the variable during initializer evaluation is no longer checked for,
    and does not cause an error.


## Core library changes

Certain core libraries APIs will have a change in specified behavior only when
interacting with opted in code.  These changes are as follows.

Calling the `.length` setter on a `List` with element type `E` with an argument
greater than the current length of the list is a runtime error unless `Null <:
E`.

The `Iterator.current` getter is given an non-nullable return type, and is
changed such that the behavior if it is called before calling
`Iterator.moveNext` or after `Iterator.moveNext` has returned `false` is
unspecified and implementation defined.  In most core library implementations,
the implemented behavior will to return `null` if the element type is
`nullable`, and otherwise to throw an error.

### Legacy breaking changes

We will make a small set of minimally breaking changes to the core library APIs
that apply to legacy code as well.  These changes are as follows.

The `String.fromEnvironment` and `int.fromEnvironment` contructors have default
values for their optional parameters.

## Migration features

For migration, we support incremental adoption of non-nullability as described
at a high level in
the
[roadmap](https://github.com/dart-lang/language/blob/master/accepted/future-releases/nnbd/roadmap.md).

### Opted in libraries.

Libraries and packages must opt into the feature as described elsewhere.  An
opted-in library may depend on un-opted-in libraries, and vice versa.

### Errors as warnings

Weak null checking is enabled as soon as a package or library opts into this
feature.  When weak null checking is enabled, all errors specified this
proposal (that is, all errors that arise only out of the new features of
this proposal) shall be treated as warnings.

Strong null checking is enabled by running the compilation or execution
environment with the appropriate flags.  When strong null checking is enabled,
errors specified in this proposal shall be treated as errors.

### Legacy libraries

Static checking for a library which has not opted into this feature (a *legacy*
library) is done using the semantics as of the last version of the language
before this feature ships (or the last version to which it has opted in, if that
is different).  All opted-in libraries upstream from the legacy library are
viewed by the legacy library with nullability related features erased from their
APIs.  In particular:
  - All types of the form `T?` in the opted-in API are treated as `T`.
  - All required named parameters are treated as optional named parameters.
  - The type `Never` is treated as the type `Null`

In a legacy library, none of the new syntax introduced by this proposal is
available, and it is a static error if it is used.

### Importing legacy libraries from opted-in libraries

The type system is extended with a notion of a legacy type operator.  For every
type `T`, there is an additional type `T*` which is the legacy version of the
type.  There is no surface syntax for legacy types, and implementations should
display the legacy type `T*` in the same way that they would display the type
`T`, except in so far as it is useful to communicate to programmers for the
purposes of error messages that the type originates in legacy code.

When static checking is done in an opted-in library, types which are imported
from legacy libraries are seen as legacy types.  However, type inference in
the opted-in library "erases" legacy types.  That is, if a missing type
parameter, local variable type, or closure type is inferred to be a type `T`,
all occurrences of `S*` in `T` shall be replaced with `S`.  As a result, legacy
types will never appear as type annotations in opted-in libraries, nor will they
appear in reified positions.

### Exports

If a legacy library re-exports an opted-in library, the re-exported symbols
retain their opted-in status (that is, downstream migrated libraries will see
their nnbd-aware types).

It is an error for an opted-in library to re-export symbols which are defined in
a legacy library (note that a symbol which is defined in an opted-in library and
then exported from a legacy library is accepted for re-export from a third
opted-in library since the symbol is not **defined** in the legacy library which
first exports it).

### Super-interface and member type computation with legacy types.

A class defined in a legacy library may have in its set of super-interfaces both
legacy and opted-in interfaces, and hence may have members which are derived
from either, or both.  Similarly, a class defined in an opted-in library may
have in its set of super-interfaces both legacy and opted-in interfaces, and
hence may have members which are derived from either, or both.  We define the
super-interface and member signature computation for such classes as follows.

#### Classes defined in legacy libraries

The legacy erasure of a type `T` denoted `LEGACY_ERASURE(T)` is `T` with all
occurrences of `?` removed, `Never` replaced with `Null`, `required` removed
from all parameters, and all types marked as legacy types.

A direct super-interface of a class defined in a legacy library (that is, an
interface which is listed in the `extends`, `implements` or `with` clauses of
the class) has all generic arguments (and all sub-components of the generic
arguments) marked as legacy types.

If a class `C` in a legacy library implements the same generic class `I` more
than once, it is an error if the `LEGACY_ERASURE` of all such super-interfaces
are not all syntactically equal.  For the purposes of runtime subtyping checks,
`C` is considered to implement the canonical `LEGACY_ERASURE` of the
super-interfaces in question.

A member which is defined in a class in a legacy library (whether concrete or
abstract), is given a signature in which every type is a legacy type.  It is an
error if the signature of a member is not a correct override of all members of
the same name in super-interfaces of the class, using the legacy subtyping
rules.

Using the legacy erasure for checking super-interfaces accounts for opted-out
classes which depend on both opted-in and opted-out versions of the same generic
interface.

```dart
//opted in
class I<T> {}

// opted in
class A implements I<int?> {}

// opted out
class B implements I<int> {}

// opted out
class C extends A implements B {}
```

The class `C` is not considered erroneous, despite implementing both `I<int?>`
and `I<int*>`, since legacy erasure makes both of those interfaces equal.  The
canonical interface which `C` is chosen to implement for the purposes of runtime
type checks is `I<int*>`.


#### Classes defined in legacy libraries as seen from opted-in libraries

Members inherited in a class in an opted-in library, which are inherited via a
class or mixin defined in a legacy library are viewed with their erased legacy
signature, even if they were original defined in an opted-in library.  Note that
if a class which is defined in a legacy library inherits a member with the same
name from multiple super-interfaces, then error checking is done as usual using
the legacy typing rules which ignore nullability.  This means that it is valid
for a legacy class to inherit the same member signature with contradictory
nullability information. For the purposes of member lookup within a legacy
library, nullability information is ignored, and so it is valid to simply erase
the nullability information within the legacy library. When referenced from an
opted-in library, the same erasure is performed, and the member is seen at its
legacy type.

We use legacy subtyping when checking inherited member signature coherence in
classes because opted out libraries may bring together otherwise incompatible
member signatures without causing an error.

```dart
// opted_in.dart
class A {
  int? foo(int? x) {}
}
class B {
  int foo(int x) {}
}
```
```dart
// opted_out.dart
// @dart = 2.6
import 'opted_in.dart';

class C extends A implements B {}
```

The class `C` is accepted, since the versions of `foo` inherited from `A` and
`B` are compatible.

If the class `C` is now used within an opted-in library, we must decide what
signature to ascribe to `foo`.  The `LEGACY_ERASURE` function computes a legacy
signature for `foo` which drops the nullability information producing a single
signature, in this case `int* Function(int*)`.  Consequently, the following code
is accepted:

```dart
//opted in
import 'opted_out.dart';
void test() {
  new C().foo(null).isEven;
}
```

#### Classes defined in opted-in libraries

The `NNBD_TOP_MERGE` of two types `T` and `S` is the unique type `R` defined
as:
 - `NNBD_TOP_MERGE(Object?, Object?)  = Object?`
 - `NNBD_TOP_MERGE(dynamic, dynamic)  = dynamic`
 - `NNBD_TOP_MERGE(void, void)  = void`
 - `NNBD_TOP_MERGE(Object?, void)  = void`
   - And the reverse
 - `NNBD_TOP_MERGE(dynamic, void)  = void`
   - And the reverse
 - `NNBD_TOP_MERGE(Object?, dynamic)  = Object?`
   - And the reverse
 - `NNBD_TOP_MERGE(Never*, Null)  = Null`
   - And the reverse
 - `NNBD_TOP_MERGE(T?, S?) = NNBD_TOP_MERGE(T, S)?`
 - `NNBD_TOP_MERGE(T?, S*) = NNBD_TOP_MERGE(T, S)?`
 - `NNBD_TOP_MERGE(T*, S?) = NNBD_TOP_MERGE(T, S)?`
 - `NNBD_TOP_MERGE(T*, S*) = NNBD_TOP_MERGE(T, S)*`
 - `NNBD_TOP_MERGE(T*, S)  = NNBD_TOP_MERGE(T, S)`
 - `NNBD_TOP_MERGE(T, S*)  = NNBD_TOP_MERGE(T, S)`

 - And for all other types, recursively applying the transformation over the
   structure of the type
   - e.g. `NNBD_TOP_MERGE(C<T>, C<S>)  = C<NNBD_TOP_MERGE(T, S)>`

In other words, `NNBD_TOP_MERGE` takes two types which are structurally equal
except for the placement `*` types, and the particular choice of top types, and
finds a single canonical type to represent them by replacing `?` with `*` or
adding `*` as required.. The `NNBD_TOP_MERGE` of two types is not defined for
types which are not otherwise structurally equal.

The `NNBD_TOP_MERGE` of more than two types is defined by taking the
`NNBD_TOP_MERGE` of the first two, and then recursively taking the
`NNBD_TOP_MERGE` of the rest.

A direct super-interface of a class defined in an opted-in library (that is, an
interface which is listed in the `extends`, `implements` or `with` clauses of
the class) has all generic arguments (and all sub-components of the generic
arguments) marked as nullable or non-nullable as written.

If a class `C` in an opted-in library implements the same generic class `I` more
than once as `I0, .., In`, and at least one of the `Ii` is not syntactically
equal to the others, then it is an error if `NNBD_TOP_MERGE(S0, ..., Sn)` is not
defined where `Si` is **NORM(`Ii`)**.  Otherwise, for the purposes of runtime
subtyping checks, `C` is considered to implement the canonical interface given
by `NNBD_TOP_MERGE(S0, ..., Sn)`.

If a class `C` in an opted-in library overrides a member, it is an error if its
signature is not a subtype of the types of all overriden members from all
super-interfaces (whether legacy or opted-in).  For the purposes of override
checking, members which are inherited from opted-in classes through legacy
classes are still checked against each original declaration at its opted-in
type.  For example, the following override is considered an error.

```dart
// opted_in.dart
class A {
  int foo(int? x) {}
}
```
```dart
// opted_out.dart
// @dart = 2.6
import 'opted_in.dart';

class B extends A {}
```

```dart
// opted_in.dart
class C extends B {
  // Override checking is done against the opted-in signature of A.foo
  int? foo(int x) {}
}
```


If a class `C` in an opted-in library inherits a member `m` with the same name
from multiple super-interfaces (whether legacy or opted-in), let `T0, ..., Tn`
be the signatures of the inherited members.  If there is exactly one `Ti` such
that `NNBD_SUBTYPE(Ti, Tk)` for all `k` in `0...n`, then the signature of `m` is
considered to be `Ti`.  If there are more than one such `Ti`, then it is an
error if the `NNBD_TOP_MERGE` of `S0, ..., Sn` does not exist, where `Si` is
**NORM(`Ti`)**.  Otherwise, the signature of `m` for the purposes of member
lookup is the `NNBD_TOP_MERGE` of the `Si`.


### Type reification

All types reified in legacy libraries are reified as legacy types.  Runtime
subtyping checks treat them according to the subtyping rules specified
separately.

### Runtime checks and weak checking

When weak checking is enabled, runtime type tests (including explicit and
implicit casts) shall succeed whenever the runtime type test would have
succeeded if all `?` on types were ignored, `*` was added to each type, and
`required` parameters were treated as optional.  This has the effect of treating
`Never` as equivalent to `Null`, restoring `Null` to the bottom of the type
hierarchy, treating `Object` as nullable, and ignoring `required` on named
parameters.  This is intended to provide the same subtyping results as pre-nnbd
Dart.

Instance checks (`e is T`) and casts (`e as T`) behave differently when run in
strong vs weak checking mode.

We define the weak checking and strong checking mode instance tests as follows:

**In weak checking mode**: if `e` evaluates to a value `v` and `v` has runtime
type `S`, an instance check `e is T` occurring in a **legacy library** is
evaluated as follows:
  - If `S` is `Null` return `LEGACY_SUBTYPE(T, NULL) || LEGACY_SUBTYPE(Object,
    T)`
  - Otherwise return `LEGACY_SUBTYPE(S, T)`

**In weak checking mode**: if `e` evaluates to a value `v` and `v` has runtime
type `S`, an instance check `e is T` occurring in an **opted-in library** is
evaluated as follows:
  - If `S` is `Null` return `NNBD_SUBTYPE(NULL, T)`
  - Otherwise return `LEGACY_SUBTYPE(S, T)`

**In strong checking mode**: if `e` evaluates to a value `v` and `v` has runtime
type `S`, an instance check `e is T` textually occurring in a **legacy library**
is evaluated as follows:
  - If `S` is `Null` return `NNBD_SUBTYPE(T, NULL) || NNBD_SUBTYPE(Object, T)`
  - Otherwise return `NNBD_SUBTYPE(S, T)`

**In strong checking mode**: if `e` evaluates to a value `v` and `v` has runtime
type `S`, an instance check `e is T` textually occurring in an **opted-in
library** is evaluated as follows:
  - return `NNBD_SUBTYPE(S, T)`

We define the weak checking and strong checking mode casts as follows:

**In weak checking mode**: if `e` evaluates to a value `v` and `v` has runtime
type `S`, a cast `e as T` **whether textually occurring in a legacy or opted-in
library** is evaluated as follows:
  - if `LEGACY_SUBTYPE(S, T)` then `e as T` evaluates to `v`.  Otherwise a
    `CastError` is thrown.

**In strong checking mode**: if `e` evaluates to a value `v` and `v` has runtime
type `S`, a cast `e as T` **whether textually occurring in a legacy or opted-in
library** is evaluated as follows:
  - if `NNBD_SUBTYPE(S, T)` then `e as T` evaluates to `v`.  Otherwise a
    `CastError` is thrown.


In weak checking mode, we ensure that opted-in libraries do not break downstream
clients by continuing to evaluate instance checks and casts with the same
semantics as in pre-nnbd Dart.  All runtime subtype checks are done using the
legacy subtyping, and instance checks maintain the pre-nnbd behavior on `null`
instances.  In strong checking mode, we use the specified nnbd subtyping for all
instance checks and casts.  However, in legacy libraries, we continue to
specifically reject instance tests on `null` instances unless the tested type is
a bottom or top type.  The rationale for this is that type tests performed in a
legacy library will generally be performed with a legacy type as the tested
type.  Without specifically rejecting `null` instances, successful instance
checks in legacy libraries would no longer guarantee that the tested object is
not `null` - a regression relative to the weak checking.

When developers enable strong checking in their tests and applications, new
runtime cast failures may arise.  The process of migrating libraries and
applications will require users to track down these changes in behavior.
Development platforms are encouraged to provide facilities to help users
understand these changes: for example, by providing a debugging option in which
instance checks or casts which would result in a different outcome if run in
strong checking mode vs weak checking mode are flagged for the developer by
logging a warning or breaking to the debugger.

