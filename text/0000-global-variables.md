- Feature Name: `global_variables`
- Start Date: 21/06/17

# Summary

This proposes the removal of the `constant` declaration and its
replacement with top-level variable declarations.

# Motivation

Currently, constants in Whiley are defined as follows:

```
constant JANUARY is 1
```

However, this syntax is rather clumsy.  Instead, this proposal is to
remove `constant` declarations and simply replace them with global
variable declarations.  For example, this would allow us to write:

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
`WhileyFile.Declaration.Variable`.  Such declarations will require, by
definition, an initialiser (a future RFC could relax this with
something equivalent to a `static` block).

# Terminology

* *Top-level Variable Declaration*.  This is a variable declaration
  which may appear at the top level (i.e. without indentation) of a
  Whiley source file.

* *Pure context*.  An expression or statement which is _functionally_
  pure.  All statements and expressions within a `function` are pure
  contexts.  Likewise, all specification elements are pure contexts.

# Drawbacks and Limitations

None.

# Unresolved Issues

Currently, there are two important questions:

* **What are the permitted forms of the right-hand side of a top-level
  variable declaration?** For example, can we make arbitrary function
  invocations?  We might choose to permit pure initialisers only.  In
  the case of `final` variables, we might need to be more restrictive.

* **What is the order of execution for top-level variable
  initialisers?** If we assume all initialisers are pure contexts then
  this is less of a problem.  However, it is possible that one
  initialiser refers to the value of another global variable if that
  is declared `final`.  In such a context, the evaluation order is
  important.  Already we have this problem in the context of
  `constant` declarations.  The requirement is only that the access
  graph is acyclic.

* **In what context can a global variable be accessed?** Initially,
  without the `final` modifier, global variables cannot be accessed
  from anywhere!  With the `final` modifier then they can be accessed
  from a pure context.
