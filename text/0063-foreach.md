- Feature Name: `foreach`
- Start Date: `19-11-2019`
- RFC PR: https://github.com/Whiley/RFCs/pull/63
- Tracking Issue: https://github.com/Whiley/WhileyCompiler/issues/982

# Summary

Whiley currently lacks a `for` loop statement which needs to be
addressed.

# Motivation

Iterating over arrays in Whiley is a fundamental and common objective
which, currently, must be done using a `while` loop.  This is rather
inadequate.

# Technical Details

This proposal reinstates Whiley's `for` each statement whose syntax is
illustrated by the following:

```Whiley
function sum(int[] items) -> (int r):
   int x = 0
   for i in 0..|xs|:
      x = x + items[i]
   return x
```

Here, the existing _array range operator_ is used as the source for
the declared variable `i`.  This proposal only supports the use of
array range syntax at this time.  The loop variable `i` must also not
have been already declared.  As usual, `for` loops do support both
`break` and `continue`, as well as `where`.  The following illustrates
a more complex example:

```Whiley
property contains(int[] items, int item, int n)
where where all { k in 0..n | items[k] != item }

function contains(int[] items, int item) -> (bool r)
ensures r ==> items[r] == item:
ensures !r ==> all { k in 0..|items| | items[k] != item }:
   //
   for i in 0..|items| where !contains(items,item,i)   
      if i == item[i]:
         return true
   return false
```

An important an unanswered question is whether or not the `where`
clause is actually needed with a `for` loop, or whether it can be
ommitted in many or all cases.  This proposal does not address this
point and leaves it for future determination.

# Extensions

Several extensions are also suggested (though not required) by this
proposal:

* **Explicit Type Declarations**.  The following syntax allows the type
    of the declared variable be given:

  ```Whiley
  for(int i in 0..|items|):
     if items[i] == item:
        return true
  ```

  This syntax requires the use of braces for disambiguation.

* **Arbitrary Sources**.  The restriction requiring the loop source be
    an array range expression could be lifted.  For example, we could
    write `contains()` as follows:
  ```Whiley
  function contains(int[] items, int item) -> (bool r)
  ensures r ==> items[r] == item:
  ensures !r ==> all { k in 0..|items| | items[k] != item }:
    //
    for i in items:
      if i == item:
         return true
    return false
  ```
  The problem here is that it's unclear whether this code could be
  verified.  Specifically, we cannot write an inductive loop invariant
  without knowledge of the current index with `items`.  Should a
  solution to this become available, then it would make sense to be
  adopted.
  
* **Pair Syntax**.  Another option might be to have a "pair" syntax,
  such as follows: 
  ```Whiley
  function contains<T>(T[] items, T item) -> (bool r)
  ensures r ==> items[r] == item:
  ensures !r ==> all { k in 0..|items| | items[k] != item }:
    //
    for i;t in items:
      if t == item:
         return true
    return false
  ```
  This could potentially work, but care is needed to avoid potential
  future problems with destructuring of tuples (hence, `;` not `,`).
  
# Terminology

* **Loop Variable**.  The _loop variable_ is the variable declared by
    the `for` loop.  This variable must not have already been
    declared.

* **Loop Source**.  The loop source is the array of items over which
    the loop variable iterates.

# Drawbacks and Limitations

None.

# Unresolved Issues

As above.
