- Feature Name: `imports`
- Start Date: `03-08-2019`
- RFC PR: (leave this empty)

# Summary

Whiley permits individual names to be imported from a module, and also
supports wildcard import syntax.  However, it would be helpful to
support syntax for importing multiple names at the same time.

# Motivation

Currently, there are three general forms of `import` syntax in Whiley.
The most common is to import a single module:

```
import std::ascii
```

This allows one to access names using the module as a prefix.  For
example, the above means allows the name `ascii::string` to resolve
but not the name `string`.  To get the latter, we can import the name
itself as follows:

```
import string from std::ascii
```

This means the name `string` will now resolve.  Finally, we can import
all names from a module using the wildcard syntax as follows:

```
import * from std::ascii
```

However, it is often the case that we want to import several names
from a module, but prefer not to import all names.  This can arise,
for example, if there is a name clash between modules.  In such case,
we must import then individually as follows:

```
import string from std::ascii
import char from std::ascii
```

Whilst this syntax is overly verbose and a little awkward, it does
work.  Various other programming languages, however, support
multi-name imports.  For example, in Rust:

```
use std::option::Option::{Some, None};
```
Likewise, in Python:
```
from math import atan,degree
```
Finally, in Elm:
```
import Html exposing (Html, div, text)
```

# Technical Details

This proposal introduces a single-line multiple import statement as
follows:

```
import string, char from std::ascii
```

Observe, however, that this proposal does **not** suggest supporting
the following:

```
import std::ascii, std::vector
```

# Drawbacks and Limitations

None.

# Unresolved Issues

None.