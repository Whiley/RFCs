- Feature Name: `qualified_names`
- Start Date: `11-08-2017`
- RFC PR: 

# Summary

This proposal is for a syntax change to fully-qualified names to
something more familiar to systems programmers, and to clarify the
meanings of different qualified names.

# Motivation

Currently, fully qualified naming in Whiley follows the syntax of
Java. Specifically, we write things like this:

```
import std.ascii
import std.io
import println from std.io

method main(ascii.string[] args):
   io.print("Hello ")
   println("World")
```

Here, the `.` operator is used to construct _fully-qualified names_ of
the form `xxx.yyy.zzz`.  The context in which these are used
determines their meaning.  For example, in the context of an `import`
statement the path `xxx.yyy.zzz` identifies a _module_
(e.g. `std.ascii` above).  However, in the context of a function or
method invocation (e.g. `io.print()` above) the path identifies a
_symbol_ (i.e. a named declaration in some module, such as a `type` or
`function`).

The goal of this proposal is two-fold.  Firstly, to clarify the
distinction between module names and symbol names.  Secondly, to
provide a syntax which is more familiar to systems programmers
(e.g. from C++ or Rust backgrounds).

# Technical Details

A _qualified name_ is a sequence of identifiers separated by `::`
(e.g. `std::ascii`, `std::ascii::to_string`, etc).  A _qualified
module name_ is a qualified name where the last component identifies
the _name_ and the preceding components (if any) constitute the
_path_.  For example, in the module name `std::ascii` it follows that
`std` is the path, and `ascii` the name.  A _qualified symbol name_ is
a qualified name where the last component identifies the symbol and
the preceding components (if any) constitute a module name.  For
example, in the symbol name `std::ascii::to_string` it follows that
`std::ascii` is the module name, and `to_string` the symbol name.
The above example in the proposed syntax would be:

```
import std::ascii
import std::io
import println from std::io

method main(ascii::string[] args):
   io::print("Hello ")
   println("World")
```

**NOTES:** The separator `::` is used for module names over `.` to
avoid overloading this with the field access operator.  Likewise, `::`
is adopted because it is familiar to C++ and Rust programmers.
Furthermore, it does provide a useful syntactic spacing between
modules and names.

Qualified names can be used in a number of different contexts:

- **Imports**.  When used in `import` statements, qualified names are
always module names (e.g. in `import std::ascii`, it follows that
`std::ascii` is a module name).
- **Expressions**.  When used in expressions, qualified names are
always symbols names (e.g.`std::ascii::to_string()` or
`std::ascii::DEL`).
- **Types**.  When used in a type, qualified names are always symbol
  names (e.g. `ascii:string` and `null|(utf8::char)`.

Another aspect of qualified naming is that a qualified name can be
_fully-qualified_ or _partially-qualified_.  For example, `std::io` is
the fully qualified name of the `io` module.  In contrast, `io::print`
is a partially qualified name for the `print` method.  This name is
considered partial because it employs an _unqualified_ module name
(i.e. `io` rather than `std::io`).  It follows from this that one can
choose to use fully qualified names to avoid `import` statements.  For
example, we could rewrite the above without `import` statements as
follows:

```
method main(std/ascii::string[] args):
   std::io::print("Hello ")
   std::io::println("World")
```

In general, however, the use of fully-qualified names should be
discouraged in favour of partially qualified names.

# Terminology

- *Module name*.  Identifies a module within the system
(e.g. `std::ascii`, `std::io`, etc)
- *Symbol name*.  Identifies a named entity within a module.  This
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
