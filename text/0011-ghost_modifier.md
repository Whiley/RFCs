- Feature Name: `ghost_modifier`
- Start Date: `24-07-2017`
- RFC PR: https://github.com/Whiley/RFCs/pull/11

# Summary

This proposal introduces the `ghost` modifier for representing state
required purely for verification.  The intention is that such state
does not manifest itself at runtime.

# Motivation

Providing explicit support for `ghost` variables seems like a sensible
choice.  In essence, this is a modifier on variables and fields.  The
purpose of `ghost` variables are to aid verification but be omitted
from generated code.  The interesting question is what variables we
can mark as `ghost` and how exactly we are permitted to use them.

Syntax:

- **Local Variables**.  The easiest use case for `ghost`.
- **Record Fields**.  This is important to allow complex data structure invariants.
- **Parameters / Returns**.  This may be considered useful for dealing
  with termination, for example.

Expressions:

- Ghost variables can only be used in _ghost expressions_.  These are
  pure expressions which occur in a _ghost-compatible context_.
- Specification elements (i.e. `requires`, `ensures`, `where` clauses
  or `assert` / `assume` statement) provide ghost-compatible contexts.
- An assignment to a ghost variable provides a ghost-compatible
  context.  This extends to parameter arguments and return values, etc.
- Ghost variables can always be assigned from pure expressions.

Example:

```Whiley
function count(int from, int to) -> (int result)
ensures result >= from:
    //
    ghost int oFrom = from
    //
    while from < to where from >= oFrom:
        from = from + 1
    //
    return from
```

In this example, variable `oFrom` is a ghost variable.  The
expression `from >= oFrom` is a ghost-expression which is used in a
ghost-compatible context (i.e. a specification element).

# Technical Details

Following on from
[RFC#0004](https://github.com/Whiley/RFCs/blob/master/text/0004-type-modifiers.md),
the language syntax is extended as follows:

```
TypeModifier = ... | ghost
```

_(At the time of writing, there aren't any other modifiers declared.
Nevertheless, some may be added before this RFC is accepted and are
represented by `...` above)_

The syntax extension permits the `ghost` modifier in any situation
where a variable is declared.  We make some additional comments:

- **Parameters.**  Functions and methods are permitted to accept
  `ghost` parameters.  The generated code for that function or method
  will accepted one less parameter in practice as a result.  However,
  it is still required that overloading be applied to `ghost`
  parameters.  This means, for example, that name mangling should
  include any `ghost` parameters.

- **Returns.**  Functions and methods are also permitted to provide
  `ghost` return values.  Of course, any such returns must be
  themselves assigned to `ghost` variables, etc.

- **Quantifier Variable Declarations.** For a quantifier used in a
  specification element, adding the `ghost` modifier has no effect.
  However, it may still be useful to add `ghost` for an expression
  used outside of a specification element (e.g. when returning a
  `ghost` value).

- **Subtyping**.  A `ghost` type can never be a subtype of a
  non-`ghost` type.  The converse perhaps might be useful in some
  cases.  For simplicity, this proposal mandates there is no subtyping
  relationship between `ghost` and non-`ghost` types.  A later RFC may
  choose to relax this if deemed necessary, and there are no hidden
  issues.

- **Definite Assignment**.  All ghost variables must be definitely
  assigned before being referred to.

The purpose of *ghost checking* is to ensure that all ghost variables
and expressions are used appropriately.  For example, a ghost
expression cannot be used as the conditional of an `if` statement
since this is not a ghost compatible context.  Likewise, a ghost
expression cannot be assigned to a variable of non-`ghost` type, etc.

## Runtime Assertion Checking

The purpose of Runtime Assertion Checking (RAC) is to check, at
runtime, whether the given assertions or specification elements hold.
This has implications for how `ghost` variables are represented.
Consider the following hypothetical example:

```Whiley
function inc(int x, ghost int ub) -> (int r):
requires x <= ub:
   //
   return x + 1
```

Since the `requires` clause refers to the ghost variable `ub`, we
cannot simply compile away `ub` as we need its value at runtime.
Therefore, we assume a distinction between _production_ builds and
_debug_ builds (i.e. where RAC is enabled).  In this case, we would
only compile away ghost variables for production builds, but not for
debug builds.  But, the implication of this is that we cannot have
code from a debug build interoperating with code from a production
build (i.e. because in the debug build `inc()` takes two parameters,
but in the production build it takes only one).

# Terminology

- *Ghost variables*.  A variable or field whose declaration has the
`ghost` modifier.

- *Ghost Expressions*.  A pure expression involving one or more ghost
variables.

- *Specification Element*.  An expression used in a `requires`,
  `ensures`, `where` clause or `assert` / `assume` statement.

# Drawbacks and Limitations

- **Production vs Debug Builds**.  This proposal results in a strong
  distinction between production builds (where RAC is disabled) and
  debug builds (where it is enabled).

# Unresolved Issues

None.
