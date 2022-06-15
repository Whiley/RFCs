- Feature Name: `imports`
- Start Date: `03-08-2019`
- RFC PR: https://github.com/Whiley/RFCs/pull/53
- Tracking Issue: https://github.com/Whiley/WhileyCompiler/issues/967

# Summary

Whiley permits individual names to be imported from a module, and also
supports wildcard import syntax.  However, it would be helpful to
support syntax for importing multiple names at the same time.

# Motivation

Currently, there are three general forms of `import` syntax in Whiley.
The most common is to import a single module:

```Whiley
import std::ascii
```

This allows one to access names using the module as a prefix.  For
example, the above means allows the name `ascii::string` to resolve
but not the name `string`.  To get the latter, we can import the name
itself as follows:

```Whiley
import string from std::ascii
```

This means the name `string` will now resolve.  Finally, we can import
all names from a module using the wildcard syntax as follows:

```Whiley
import * from std::ascii
```

However, it is often the case that we want to import several names
from a module, but prefer not to import all names.  This can arise,
for example, if there is a name clash between modules.  In such case,
we must import them individually as follows:

```Whiley
import string from std::ascii
import char from std::ascii
```

Whilst this syntax is overly verbose and a little awkward, it does
work.  Various other programming languages, however, support
multi-name imports.  For example, in Rust:

```Rust
use std::option::Option::{Some, None};
```
Likewise, in Python:
```Python
from math import atan,degree
```
Finally, in Elm:
```Elm
import Html exposing (Html, div, text)
```

# Technical Details

This proposal introduces a single-line multiple import statement as
follows:

```Whiley
import string, char from std::ascii
```

Observe, however, that this proposal does **not** suggest supporting
the following:

```Whiley
import std::ascii, std::vector
```

# Drawbacks and Limitations

None.

# Unresolved Issues

None.
