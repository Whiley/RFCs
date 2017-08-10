- Feature Name: `qualified_names`
- Start Date: `11-08-2017`
- RFC PR: 

# Summary

Proposes a syntax change for fully qualified names to something which
is more familiar to systems programmers, and also which properly
distinguishes the "inside" from "outside" of a mode.

# Motivation

Currently, fully qualified naming in Whiley follow the syntax of
Java. Specifically, we write things like this:

```
import std.ascii
import std.io
import println from std.io

method main(ascii.string[] args):
   io.print("Hello ")
   println("World")
```

Here, the '.' operator is used to construct _fully-qualified names_ of
the form 'xxx.yyy.zzz'.  The context in which these are used
determines their meaning.  For example, in the context of an `import`
statement the path `xxx.yyy.zzz` identifies a _module_
(e.g. `std.ascii` above) .  In contrast, in the context of an function
or method invocation (e.g. `io.print()` above) the path identifies an
_entity_ (i.e. named declaration in some module, such as a `type` or
`function`).

The goal of this proposal is two-fold.  Firstly, to properly
distinguish between a module name and an entity name.  Secondly, to
provide a syntax which is more familiar to systems programmers
(e.g. from C++ or Rust backgrounds).

# Technical Details

The proposed syntax for fully qualified names is
`name/of/module::entity`.  Thus, a module name is always a collection
of names separated by `/`, whilst a (fully-qualified) entity name is
always a module name and an entity name separated by `::`.  The above
example in the proposed syntax would be:

```
import std/ascii
import std/io
import println from std/io

method main(ascii::string[] args):
   io::print("Hello ")
   println("World")
```

Here, `/` is used to separate module names over `.` to avoid
overloading this with the field access operator.  Likewise, `::` is
adopted because it is familiar to C++ and Rust programmers.
Furthermore, it does provide a useful syntactic spacing between
modules and names.

One aspect of fully-qualified names is that, in fact, there are both
_fully-qualified_ and _partially-qualified_ names.  Thus, `std/io` is
the fully qualified name of the `io` module.  However, `io::print` is
a partially qualified name for the 'print' method.  Specifically, it
is considered partial because it employs an unqualified module name
(i.e. `io` rather than `std/io`).

# Terminology

- *Module name*.  Identifies a module within the system
(e.g. `std/ascii`, `std/io`, etc)
- *Entity name*.  Identifies a named entity within a module.  This
could be, for example, a `type`, `method` or `function` declaration.
- *Fully qualified name*.  A fully-qualified module name is the
  complete system path for that module.  Likewise, a fully-qualified
  entity name employs a fully-qualified module name.
- *Partially qualified name*.  A partially-qualified entity name uses
  an unqualified module name.

# Drawbacks and Limitations

Obviously, existing code will be broken.

# Unresolved Issues

None.
