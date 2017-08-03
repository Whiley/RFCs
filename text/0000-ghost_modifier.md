- Feature Name: `ghost_modifier`
- Start Date: `24-07-2017`
- RFC PR:

# Summary

This proposal introduces the `ghost` modifier for representing state
required purely for verification.  Such state does not manifest itself
at runtime.

# Motivation

Providing explicit support for `ghost` variables seems like a sensible
choice.  In essence, this is a modifier on variables and fields.  The
purpose of `ghost` variables it to aid verification, but to be omitted
from generated code.  The interesting question, is what variables we
can mark as `ghost` and how exactly we are permitted to use them.

Syntax:

- **Local Variables**.  The easiest use case for `ghost`.
- **Record Fields**.  This is important to allow complex data structure invariants.
- **Parameters**.  This may be considered useful for dealing with
  termination, for example.

Expressions:

- Ghost variables can be used in all `requires` and `ensures` clauses.
- Ghost variables can be used in all `where` clauses.
- Ghost variables can always be assigned.
- Ghost variables can only be used in _pure expressions_ and where the lval is also `ghost`.

Example:

```
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

# Technical Details

Following on from
[RFC#0004](https://github.com/Whiley/RFCs/blob/master/text/0004-type-modifiers.md),
the language syntax is extended as follows:

```
TypeModifier = ... | ghost
```

At this stage, there aren't any other modifiers declared.
Nevertheless, some may be added before this RFC is accepted and are
represented by `...` above.

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
  cases.  For simplicity, this proposal seems mandates there is no
  subtyping relationship between `ghost` and non-`ghost` types.  A
  later RFC may choose to relax this if deemed necessary, and there
  are no hidden issues.

Finally, we note that all `ghost` variables must be definitely
assigned before being referred to.

## Ghost Checking

Variables of `ghost` type can be used in a number of different
situations:

- **Specification Elements**.  A variable of `ghost` type can be
  accessed and manipulated in any specification element
  (e.g. `requires`, `ensures`, `where` clause or `assert` / `assume`
  statement).

- **Other Ghost Expressions**.  A `ghost` expression is one which accesses
  one or more `ghost` variables.  Any `ghost` expression must be
  assigned to a variable of `ghost` type.

The purpose of *`ghost` checking* is to ensure that all `ghost`
expressions are used appropriately.  For example, a `ghost` expression
cannot be used as the conditional of an `if` statement.  Likewise, a
`ghost` expression cannot be assigned to a variable of non-`ghost` type.

## Runtime Assertion Checking

The purpose of runtime assertion checking is to check, at runtime,
whether the given assertions or specification elements hold.

# Terminology

# Drawbacks and Limitations

None.

# Unresolved Issues

- **Runtime Assertion Checking.** An interesting question is how
  `ghost` variables are represented in the presence of Runtime
  Assertion Checking (RAC).  For example, if an assertion of some kind
  refers to a `ghost` variable then we cannot simply compile away its
  value.  Furthermore, if we only compile it away when RAC is disabled
  then we cannot have RAC code interoperating with non-RAC code.  We
  could consider some implementation strategies to work around this.
