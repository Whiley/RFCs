- Feature Name: `global_variables`
- Start Date: 21/06/17
- PR: https://github.com/Whiley/RFCs/pull/8

# Summary

This proposes the removal of the `constant` declaration and its
replacement with top-level --- or *static* --- variable declarations.

# Motivation

Currently, constants in Whiley are defined as follows:

```
constant JANUARY is 1
```

However, this syntax is rather clumsy.  Instead, this proposal is to
remove `constant` declarations and simply replace them with top-level
variable declarations.  Such top-level variable declarations are
referred to as _static variable declarations_.  For example, this
would allow us to write:

```
int JANUARY = 1
```

This is not a strict replacement for the original `constant`
declaration because it represents _mutable state_.  However, through
a separate (and complimentary) proposal we could write the following:

```
final int JANUARY = 1
```

This is now a proper replacement for `constant` declarations.  

# Technical Details

The syntactical details are relatively straightforward.  The top-level
declaration `WhileyFile.Declaration.Constant` will be replaced with
`WhileyFile.Declaration.StaticVariable`.  Such declarations will
require, by definition, an initialiser (though a future RFC could
relax this with something equivalent to a `static` block).

## Initialisers

An important question is: _what form are static initialisers permitted
to take?_  For this purpose, the concept of a _constant expression_ is
introduced.  That is an expression which can be evaluated at compile
time.  A constant expression is an expression with the following
constraints:

- **Purity.** A constant expression must be pure and is not permitted
  to have side-effects.  This means, amongst other things, that they
  cannot invoke `method`s, or create state through `new`.

- **Function Invocation.** A constant expression is also not permitted
  to invoke a `function`.  This is to ensure it can be efficiently
  evaluated at compile time.
  
- **Static Access.** A constant expression may only access static
  variables declared as `final` and/or take the address of a
  `function` or `method`.  Note that cyclic dependencies are not
  permitted.

We note that future RFCs may choose to further relax these
restrictions as necessary.  For example, by introducing the concept of
compile-time functions or similar.

# Side effects

Another important question is: _in what context can a static variable be
accessed?_  This proposes the following:

- **Final Statics**.  These are effectively compile-time constants and
  can be accessed from anywhere.  This includes within `functions` and
  also as `case` labels.

- **Non-Final Statics**.  These represent global state and, hence,
  cannot be accessed in any pure context (e.g. within a `function`).
  In principle, these should be accessible from within a _non-pure_
  context (e.g. within a `method`).  However, at this time, there is
  no syntax for expressing the necessary `modifies` clause of a method
  (this will be added separately later).

- **Volatile Statics**.  These represent global state which may be
  accessed concurrently.  At this time, it is impossible to declare
  such a static variable (though in the future this may be relaxed).
  Nevertheless, this proposal states that a non-`final` static cannot
  be used in a concurrent context unless this is explicitly stated
  through an appropriate modifier (e.g. `volatile`).

An important observation from the above is that, under this proposal,
non-`final` static variables cannot be accessed at all!  In the
future, this restriction will be relaxed through the use of a
`modifies` clause or similar.

# Terminology

* *Static Variable Declaration*.  This is a variable declaration which
  may appear at the top level (i.e. without indentation) of a Whiley
  source file.

* *Pure Context*.  An expression or statement which is _functionally_ pure.
  All statements and expressions within a `function` are pure
  contexts.  Likewise, all specification elements are pure contexts.

* *Constant Expression*.  An expression which can be completely
  evaluated at compile time.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
