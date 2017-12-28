- Feature Name: `readwrite_types`
- Start Date: `28-12-17`
- RFC PR:

# Summary

Currently, Whiley supports the concepts of _readable_ and _writable_
types which are known collectively as _effective types_.  This
proposal simply removes these types from the language in order that
they can be properly reintroduced at a later stage.

# Motivation

A union of two compatible types may be treated as effective type in
certain situations.  The following illustrates an effective record
being used in a read position:

```
type Point2D is {int x, int y}
type Point3D is {int x, int y, int z}
type Point is Point2D | Point3D

function getX(Point p) -> (int r):
	return p.x
```

Here, the _readable type_ of parameter `p` is `{int x, int y, ...}`.
The following illustrates a similar situation with arrays:

```
type arr_t is (int[])|(bool[])

function read(arr_t arr, int idx) -> (int|bool r):
	return arr[idx]
```

Again, the _readable type_ of parameter `arr` is `(int|bool)[]` and,
thus, reading from this returns an element of type `int|bool`.

When effective types are used for assignment, they must exhibit
appropriate _writeable types_.  The following illustrates for arrays:

```
type nint is null|int
type nbool is null|bool
type arr_t is (nint[])|(nbool[])

function write(arr_t arr, int idx) -> (arr_t r):
    arr[idx] = null
	return arr
```

Here, the writable type for `arr` is `((null|int)&(null|bool))[]`
which simplifies to just `null[]`.  That is, since only `null` values
are common to both arrays, we can only assign them safely.

Effective types are a very useful shorthand which shares similarity
with the concept of a
[Common Initial Sequence in C](http://www.iso-9899.info/wiki/Common_Initial_Sequence).
However, effective types are not strictly required as they are simply
a form of syntactic sugar.  For example, the above can be rewritten as follows:

```
type nint is null|int
type nbool is null|bool
type arr_t is (nint[])|(nbool[])

function write(arr_t arr, int idx) -> (arr_t r):
    if arr is nint[]:
       arr[idx] = null
	else:
	   arr[idx] = null
	return arr
```

This does not make use of effective types since parameter `arr` has
an explicit array type in both branches.

# Technical Details

The change is relatively straightforward.  The flow type checker needs
to be updated to remove uses of the readable and writeable type extractors.

# Terminology

* **Effective Type**.  This is a union of two or more compatible types
  which give the appearance of a concrete type (e.g. an array, a
  record, etc).

* **Readable Type**.  This is the effective type for an expression
  when used in a read position.

* **Writeable Type**.  This is the effective type for an expression
  when used in an assignment position.

# Drawbacks and Limitations

Existing code will need to be migrated.

# Unresolved Issues
