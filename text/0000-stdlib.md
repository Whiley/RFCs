- Feature Name: stdlib
- Start Date: `11/07/17`
- RFC PR: (leave this empty)

# Summary

This proposals a simple refactoring of the Whiley standard library to
bring it more inline with other systems languages.

# Motivation

The Whiley Standard Library currently consists of the following
modules:

```
whiley.io.File
whiley.io.Reader
whiley.io.Writer
whiley.lang.Any
whiley.lang.Array
whiley.lang.ASCII
whiley.lang.Byte
whiley.lang.Int
whiley.lang.Math
whiley.lang.Stack
whiley.lang.System
```

This package structure is designed along the lines of Java's standard
library.  However, given the focus of Whiley as a systems language, it
seems more appropriate to follow the structure of other similar
languages (e.g. Rust, C, C++, D).  Furthermore, instead of imagining a
large system library, it seems more prudent to imagine a small
library.  A "flatter"package structure would reflect this.  As such,
the proposed package structure is:

- `std.ascii` --- For all ASCII related stuff.  This is simply a
  renaming of `whiley.lang.ASCII`.

- `std.io` --- For all basic definitions of readers / writers.  This
  merges `whiley.io.Reader`, and `whiley.io.Writer`.

- `std.array` --- For all array manipulation functions.  This is simply a
  renaming of `whiley.lang.Array`.

- `std.int` --- For all integer coercion operations, and other
  functions (e.g. parsing integer from string).  This is simply a
  renaming of `whiley.lang.Int`.

- `std.math` --- For all math related functions, e.g. `abs()`,
  `min()`, `max()`, `gcd()`, etc. This is simply a renaming of
  `whiley.lang.Math`.

In addition, we would image the following additional package in the
future:

- `std.utf` --- Providing various utilities for working with UTF
  strings and encodings.

- `std.collections` --- Providing various commonly used data
  structures (e.g. `HashMap`, `ArrayList`, etc).

- `std.net` --- Providing various networking primitives, of which
  some notion of a `socket` would be most notable.

- `std.fs` --- Providing various filesystem primitives, such as for
  reading/writing files and listing directories, etc.

This package structure more closely aligns with that of
[C++](https://en.wikipedia.org/wiki/C%2B%2B_Standard_Library),
[Rust](https://doc.rust-lang.org/std/#modules) and
[D](https://dlang.org/phobos/).

What is lacking from proposed structure above is support for a wider
range of file formats.  For example, the D standard library (phobos)
includes support for `xml`, `json`, `csv`, `zip`, and more.  Whether
or not such formats should be supported in WySTD remains to considered
in the future.

# Technical Details

None.  This is a simple refactoring of the existing code base.

# Terminology

None.

# Drawbacks and Limitations

* **Backwards Compatibility.** Of course, many existing programs would
  no longer compile.

# Unresolved Issues

None.
