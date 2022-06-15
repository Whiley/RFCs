- Feature Name: `binary-heap`
- Start Date: `29-08-2017`
- RFC PR: https://github.com/Whiley/RFCs/pull/14

# Summary

This clarifies the generic binary file format used for representing
_syntactic heaps_.  This can, for example, be used to store Abstract
Syntax Trees and is the underlying encoding used in binary WyIL files.

# Motivation

This is the first step in documenting the binary representation of
WyIL files which are essentially a binary encoding of Whiley source
files, along with additional information as necessary.  This document
does not dictate the exact constraints of a WyIL file.  Rather, it
describes the general encoding format upon which they are built.
Subsequent RFCs will document the exact details of WyIL files.

# Technical Details

The format presented is a binary file format, and the purpose of this
document is to clarify the exact representation.  Various primitives
are used for encoding a syntactic heap, and these are discussed first.

## Bit Streams

A _bit stream_ is a sequence of one or more bits of which the first
bit is referred to as the _Least Significant Bit (LSB)_ and the last
as the _Most Significant Bit (MSB)_.  The intuition is that the LSB is
written first to the stream and the MSB is written last (though in
practice this depends on how bytes are stored in the underlying
medium).

For the purposes of this document, we write bit streams with a
right-to-left layout where the right-most bit is the LSB.  For
example:

```
010111100110 >>
```

The `>>` denotes the start of the stream and, thus, the bit
immediately to its left is the LSB.

Bit streams are packaged into _byte streams_ and, where padding is
required, one or more zero bits are added after the MSB to ensure the
overall stream is a multiple of eight bits.  After padding, our stream
above looks as follows:

```
00000101 11100110 >>
```

Thus, the byte stream is `05 E6` in hexadecimal where `E6` is the
first byte written to the stream.

## Fixed-Width Integer Encoding

Fixed width integers are denoted as `un` or `in` where `n` is a given
number of bits.  For example, `u8` represents an unsigned byte.  Fixed
width integers are encoded as is (i.e. without meta-information) in
a _little endian format_.  For example, consider the following
definition:

```
type Header is { u8 opcode, u16 operands }
```

An instance of `Header` is stored in a binary stream as follows (which
reads from right-to-left for convenience):

```
+----------------+--------+
|    operands    | opcode |
+----------------+--------+
 23             8 7      0
```

The first 8 bits represent the `opcode` field, and the remaining 16
bits represent the operand array.  Bit `0` is the _least significant
bit_ and would be written to the stream first.  Thus, literal
`{opcode=3,operands=16}` is encoded as the following bit stream:

```
00000000 00010000 00000011 >>
```

In hexadecimal, this corresponds with `0x00 0x10 0x03` (remembering
that the _rightmost_ byte here is written first).

## Variable-Width Integer Encoding

The encoding of variable-width integer values shares some similarity
with the [LEB128](https://en.wikipedia.org/wiki/LEB128) format.  Like
LEB128, chunks following a little-endian layout.  However, unlike,
LEB128 each chunk is only four bits in length.

Unsigned integers are denoted by `uv` and are zero extended to the
nearest multiple of three bits.  Likewise, signed integers are denoted
by `iv` and are sign-extended to the nearest multiple of three bits.
Each chunk is then extended to a four bit nibble with a _carry flag_
as the most significant bit to indicate whether (or not) more nibbles
follow.  This looks roughly as follows:

```
+----+     +----+----+
|0???| ... |1???|1???|
+----+     +----+----+
            7  4 3  0
```

Payload bits are indicated by `?`, whilst the carry bit is `1' for the
non-terminating nibbles and '0' for the terminating nibble.

As an example, we consider the following alternative definition of
`Header`:

```Whiley
type Header is { u8 opcode, uv operands }
```

The literal `{opcode=5, operands=10}` the corresponds to the following
bitstream:

```
0001 1010 00000101 >>
```

The bits representing the `opcode` field are first and form a single
byte.  The raw bit representation for `operands` is `1010` which
splits into two chunks `001 010` that are extended with carry bits to
give `0001 1010`.

## Array Encoding

Arrays are encoded using an explicit length variable.  For example,
consider the following:

```Whiley
type packet is { u8 len, u8[len] data }
```

The number of elements in the `data` array is explicitly determined by
the `len` field.  No packing is used and, hence, the layout goes like
this:

```
+---+     +---+---+---+
| n | ... | 1 | 0 |len|
+---+     +---+---+---+
            16   8   0
```

Here, elements are indexed from zero and the nth element has index
`len-1`.  In general, explicit length variables are used as their
exact type cannot otherwise be easily inferred.  Finally, a constant
array length can be used (e.g. `u8[8]`) and, in such cases, no length
field is encoded.

## Binary Layout

Syntactic items are the fundamental abstraction used in the binary
heap.  In essence, a binary heap consists of a `header` followed by
zero or more instances of `Item`.  The following clarifies the
high-level format:

```Whiley
type BinaryHeap is {
  u8[?] header,
  uv nItems,
  Item[nItems] items
}
```

The `header` field is simply an arbitrary sequence of bytes.  Details
of the header's contents and length are determined by the concrete
format being used (e.g. WyIL).  Typically, the header will include a
magic number and supplementary meta-data.

The `Item` type provides the main building block of the format.  Each
item may refer to zero or more _operands_ which are themselves
`Items`.  In addition, an `Item` may contain a payload of zero or more
bytes.  The payload is typically used for encoding integer and string
constants, etc.  The generic format of an `Item` is defined as follows:

```Whiley
type Item is {
  u8 opcode,
  u8[?] rest
}
```

Here, the number of bytes are determined by the overall `schema` of
the concrete format.  We define a number of concrete examples:

```Whiley
type NaryItem is {
  u8 opcode,
  uv size,
  uv[size] operands
}
```

This defines an `Item` which has an arbitrary number of
operands, but carries no payload.  Likewise, we can define instances
for opcodes which have a fixed number of operands as follows: 

```Whiley
type UnaryItem is {
  u8 opcode,
  uv[1] operands
}
type BinaryItem is {
  u8 opcode,
  uv[2] operands
}
type TernaryItem is {
  u8 opcode,
  uv[3] operands
} 
```

As another example, we can define an `Item` which has no operands but,
instead, carries an arbitrary-sized payload:

```Whiley
type ArbitraryPayloadItem is {
  u8 opcode,
  uv len,
  u8[len] payload
}
```

This is a common type for holding arbitrary integer or string
constants.  At this stage, we have completed our overview of the file
format.  Many details remain to be fleshed out by concrete
instantiations.  Nevertheless, they will still follow this high-level
organisation.

# Terminology

# Drawbacks and Limitations

- The use of four bit nibbles remains a potential limitation.  Further
  research is necessary to determine whether larger chunks are more
  space efficient in practice.

- The use of a single byte opcode is potentially limiting.  However,
  as in other bytecode formats, one or more bytecode values could be
  used to indicate a multi-byte opcode.

# Unresolved Issues

- The addressing mode for `operands` is unspecified.  This could be
  either _absolute_ or _relative_.  A relative scheme offers the
  benefits of smaller nibbles in the case of spatially located
  operands.  Alternatively, we could simply leave the decision of
  which addressing mode to use upto the concrete implementation.
