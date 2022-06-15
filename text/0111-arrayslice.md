- Feature Name: `array_slice`
- Start Date: `27-05-22`
- RFC PR: https://github.com/Whiley/RFCs/pull/111

# Summary

The array slice is a useful operation found in many languages, and it
would be good to add this to Whiley.

# Motivation

The array slice would be a useful addition to Whiley.  For example,
consider the current implementation of `sum()`:

```Whiley
property sum(int[] items, int i) -> (int r):
   if i < 0 || i >= |items|:
      return 0
   else:   
      return items[i] + sum(items,i+1)
```

This is a bit cumbersome because of the need to use an index `i`.
Note that, whilst the `std::array::slice()` exists, this is not a
`property` and hence would be uninterpreted in this situation meaning
some things would not verify.  A neat solution can be found using the
array slice operator as follows:

```Whiley
property sum(int[] items) -> (int r):
   if items == []:
      return 0
   else:
      return items[0] + sum(items[1..])
```

This is a simpler and more naturally recursive implementation.

# Technical Details

The mechanics are pretty straightforward and would operate in a
similar fashion to ranges.  As for ranges, the left-hand side is
inclusive and the right-hand side exclusive (this follows Rust and
Dafny for example).  Should the syntax for ranges change to allow
different combinations of exclusivity, then we would update this
operator accordingly.

The operator would support three forms:

   * `arr[s..e]` which returns the subarray from `s` upto (but not
     including) `e`.

   * `arr[s..]` is syntactic sugar for `arr[s..|arr|]`

   * `arr[..e]` is syntactic sugar for `arr[0..e]`.

# Terminology

None.

# Drawbacks and Limitations

N/A

# Unresolved Issues

N/A
