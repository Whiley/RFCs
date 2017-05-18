- Feature Name: Binary Literals
- Start Date: 18/05/17
- RFC PR: (leave this empty)

# Summary

The Whiley language provides syntax for expressing binary literals
(e.g. `0101b`), how this differs from what has now become the defacto
standard (`0b0101`).  This proposal simply brings Whiley in line with
other languages.

# Motivation

The Whiley language provides syntax for expressing binary literals
(e.g. `0101b`).  This is actually a nice syntax!  However, at this
time, there are now several languages
(e.g. [Java](https://docs.oracle.com/javase/8/docs/technotes/guides/language/binary-literals.html),
[C++](http://www.informit.com/articles/article.aspx?p=2209021),
[Rust](http://rustbyexample.com/primitives/literals.html)) which
support binary literals as well.  The general consensus is that the
appropriate syntax for binary literals follows the current standard
syntax for hexadecimal literals --- namely, `0b0101` or `0B0101`.

As such, the motivation for this proposal is to bring Whiley in line
with what has become the defacto standard.  This is a relatively minor
issue, but nonetheless would benefit programmers in general.

# Technical Details

This proposal is relatively straightforward.  The language should be
updated to support binary literals of the form `0b0101` with `0b0`
being the smallest supported literal.

# Terminology

None.

# Drawbacks and Limitations

Existing programs which support the old literal syntax will need to be
updated.

# Unresolved Issues

None.
