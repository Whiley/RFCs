- Feature Name: `test_file_format`
- Start Date: `22-04-2022`
- RFC PR: (leave this empty)

# Summary

Currently, Whiley tests are either valid or invalid and consist of a
single `.whiley` (possibly along with expected output for errors).
This proposal is for a single unified format which should offer
several key advantages.

# Motivation

At the moment, Whiley test files are stored in `tests/valid` and
`tests/invalid` depending on their expected output.  For example,
files in `tests/valid` are expected to compile successfully and run
without error.  In contrast, files in `tests/invalid` are expected to
fail in some way (e.g. type error or verification error).  There are
many issues with this approach:

  1) Distinguishing between _valid_ and _invalid_ tests is ad-hoc and
  unnecessary.  Valid tests may produce e.g. compiler warnings and
  invalid tests may only fail at runtime (e.g. if verification cannot
  be applied).

  2) There is no way in the present system to have tests which employ
  multiple modules, and this leaves open a class of bugs which cannot
  be tested for.

  3) There are certain known properties about the tests which are not
  encoded consistently.  For example, some tests are known to fail to
  compile because of some outstanding issue on github.  Other tests
  may compile but fail to execute on the interpreter, and yet others
  might fail verification because of open issues.  Similarly, certain
  parameters may be required for a test to pass (e.g. a larger than
  normal `timeout` value, or specific `min` / `max` ranges for
  integers).

  4) There is no way in the present system to test incremental
  compilation.  Again, this means whole classes of bugs are invisible.
  Furthermore, when I start the push for a fully incremental compiler,
  this will be a serious problem.
  
This proposals gives a single unified file format for representing an
individual Whiley test.  The format supports multiple files in
arbitrary structures, allows properties to be specified and allows
mutations to files as well.  The format also prefers _error numbers_
over _error messages_ making it more robust to astetic changes.

# Technical Details

Each test file consists of zero or more configuration settings (of the
form e.g. `x.y.z = false`) followed by one or more _frames_ which are
applied in order of appearance.  Each frame can write or modify one or
more source files, and determines the expected errors and warnings
produced from compiling the code as it is after the frame is applied.

An example testfile is the following:

```
whiley.verify = false
boogie.timeout = 1000
================
>>> main.whiley
method main():
>>> other.whiley
import main
---
E101 main.whiley:1,2
E302 main.whiley:2,2:3
================
<<< other.whiley
>>> main.whiley:1:1
method main()
    skip
---
E202 other/main.whiley:2,2
```

Each frame begins with a line of (at least three) `===` and ends
either at EOF or the next frame.  Within each frame are various
markers, such as for signaling changes to source files:

   * '>>> filename' Indicates a write to the given file.  If no
     _range_ is given, the entire file is replaced.  A range `m:n`
     indicates the line starting at `m` and upto (but not including)
     line `n` is replaced.  Therefore, a range such as `1:1` indicates
     an _insertion_ on line `1`.  Note, the range `m` is shorthand for
     `m:m`.

  * `<<< filename` indicates the given file is deleted in its
    entirety.

   * `---` indicates the end of the change sequence.  The remainder of
     the frame contains the expected errors and warnings (if any).
     Each error line identifies the error or warning number
     (e.g. `E101`, `W202`) followed by a _coordinate_ which identifies
     the file where the error is arising, and where exactly it is
     reported.  Specifically, the source location is given by `l,m:n`
     where `l` is a line number, and `m:n` is a character range within
     that line (defined as before).

# Drawbacks and Limitations

This doesn't identify more complex refactorings, such as renaming a
file, etc.  Potentially, this might be helpful at some point.

# Unresolved Issues

None.
