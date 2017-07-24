- Feature Name: `ghost_modifier`
- Start Date: `24-07-2017`
- RFC PR:

# Summary

This proposal introduces the `ghost` modifier for representing state
required purely for verification.  Such state does not manifest itself
at runtime.

# Motivation

Providing explicit support for `ghost` variables seems like a sensible choice.  In essence, this is a modifier on variables and fields.  The purpose of `ghost` variables it to aid verification, but to be omitted from generated code.  The interesting question, is what variables we can mark as `ghost` and how exactly we are permitted to use them.

Syntax:

- **Local Variables**.  The easiest use case for `ghost`.
- **Record Fields**.  This is important to allow complex data structure invariants.
- **Parameters**.  An interesting question as to whether or not this should be permitted?

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


# Terminology


# Drawbacks and Limitations

# Unresolved Issues
