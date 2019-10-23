- Feature Name: `import_with`
- Start Date: `24-10-2019`
- RFC PR: (leave this empty)
- See also: [RFC#53](https://github.com/Whiley/RFCs/blob/master/text/0053-imports.md)

# Summary

A common pattern arising is that one wants to import a module
(e.g. `std::vector`), and one or more specific items from that module
(e.g. `Vector`).  This RFC proposes a specific syntax for this.

# Motivation

At this stage, `import` statements in Whiley are a bit cumbersome.  For example, I end up writing stuff like this a lot:

```
import std::vector
import Vector from std::vector
```

This allows me to write something like this:

```
Vector<int> vec = Vector([1,2,3])
vec = vector::push(vec,4)
```

Which is slightly nicer than this:

```
vector::Vector<int> vec = vector::Vector([1,2,3])
vec = vector::push(vec,4)
```

**Therefore**, it would be nice to support syntax for this.  In fact,
[Elm has something like
this](https://stackoverflow.com/questions/30172903/what-does-exposing-mean-in-elm):

```
import Html.Lazy exposing (lazy2)
```

# Technical Details


This RFC proposes the following syntax:

```
import std::vector with Vector
```

This then replaces the two lines above with one.  In addition, we can
use a comma-separated list as follows:

```
import std::ascii with string, char
```

The above is then equivalent to the following:

```
import std::ascii
import string from std::ascii
import char from std::ascii
```

# Terminology

None.

# Drawbacks and Limitations

None.

# Unresolved Issues

None.