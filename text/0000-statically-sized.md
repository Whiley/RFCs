- Feature Name: statically_sized_types
- Start Date: 22-05-2017

# Summary

This proposal is for introducing _statically sized types_ to the
Whiley language.  Such types occupy a fixed (and known) amount of
space at runtime.  For example, a statically sized integer type might
occupy exactly 32bits at runtime.

# Motivation

The Whiley Language currently supports dynamically-sized data types.
For example, the `int` type in Whiley is unbound and can
hold any possible integer (theoretically, at least).  Such types are
naturally difficult to represent efficiently on a machine.
Furthermore, they impede communication between Whiley and foreign
code which employs fixed-sized data types (e.g. `int32` in C).

This proposal addresses the problem of specifying what underlying
representation a given type should have.  That is, providing a way for
the programming to dictate in absolute terms what representation
should be used.  This allows the programmer to optimise the underlying
representation as he/she sees fit.  It also enables them to interface
with foreign code more easily.

Several alternatives that could be considered to this proposal:

- **Library definitions of statically sized types**.  The standard
  library currently includes various definitions of statically sized
  types (e.g. `u8`,`i16`,`u32`, etc).  Knowledge of these could be
  hard-coded into the compiler.  However, this unnecessarily couples
  the compiler with the standard library.  Furthermore, there are
  certain foreign data types (e.g. C strings) which cannot be easily
  modelled in this way.

- **Range Analysis**.  Another approach previously explored was the
  use of integer range analysis (see
  [here](http://homepages.ecs.vuw.ac.nz/~djp/files/SEUS15.pdf)).
  Here, the compiler examines the invariants given on a datatype to
  determine an appropriate representation.  For example, consider a
  type `(int n)` where `n >= 0 && n <= 255`.  The range returned for
  this type would be `0 .. 255` and, hence, it could fit into a single
  `byte`.  This approach improves upon the previous in that it does
  not couple the compiler with the standard library.  However, it
  still cannot encode certain data types (e.g. C strings).
  Furthermore, the range analysis process is hard to state clearly
  because of the complex nature of invariants in Whiley.  For example,
  what range should we return for the constraint `n >= 0 && (n <= 255
  || n < 128)`?  Probably `0 .. 255` makes the most sense here, but
  invariant expressions can be arbitrarily complex and we need a clear
  and simple mechanism for specifying representation.  Finally, use
  range analysis makes interoperation with foreign code more
  challenging.  This is because we may have foreign data types whose
  representation is not minimal for their true invariants.  For
  example, an `int` parameter with only seven possible values
  (e.g. `1`..`7`).  In such case, we need to separate the
  representation from the constraints.

As a result of their limitations, the above approaches are not
considered suitable for addressing the problem.

# Technical Details

# Terminology

In changing the language, it is important to develop a suitable
"vernacular" for talking about the new aspects of the language.  For
example, if a new piece of syntax is proposed, then a standard way of
referring to this should be proposed.  Likewise, if a new phase of the
compiler is introduced, then suitable terminology for discussing the
various aspects of this should be developed.

# Drawbacks and Limitations

This should identify any potential drawbacks or limitations of the
proposed change.  For example, if the proposed change would break
existing code, this should be clearly identified.  Likewise, if the
change could impact upon other proposed changes, this should be
identified.

# Unresolved Issues

List any currently unresolved aspects of the change proposal.  These
will need to be adequately resolved before the RFC is accepted.
Identifying these issues provides a way for the author(s) of the RFC
to leverage the community in finding appropriate solutions.
