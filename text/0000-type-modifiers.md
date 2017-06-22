- Feature Name: type_modifiers
- Start Date: 22/06/17

# Summary

Proposes a syntax and semantic for type modifiers which allow
restrictions to be placed on values of a type.

# Motivation

There is a clear need for the ability to express a range of simple
properties over types.  This document does not propose any such
properties but, instead, proposes a syntax and semantic for expressing
them.  Such properties are referred to as _type modifiers_.

## Examples

We now present a number of examples to illustrate some uses of such
type modifiers.

* The _final_ modifier would prevent variables (or parts of variables)
from being modified.  Any variable declared `final` would require an
_initialiser_.  The following illustrates:

```
final int x = 1
```

* The _atomic_ modifier would indicate a variable which can be shared
amongst one or more concurrently executing threads.  By default, this
would be sequentially consistent (following C11).  The following illustrates:

```
atomic &int x = new 1
```

* The _foreign_ modifier would indicate a type representing foreign
  data and, hence, that forms part of the _foreign function
  interface_.

```
type JsObject is foreign { ... }
```

* The _ghost_ modifier would indicate a value present purely for the
  purposes of verification.  Such values would never be compiled on
  the back-end into actual code.

```
type Purse is {
  ghost &Mint owner,
  int amount
}
```

# Technical Details

* Reserved Keywords.

# Terminology

* **Type Modifier**.

# Drawbacks and Limitations

# Unresolved Issues
