- Feature Name: `binary-heap`
- Start Date: `29-08-2017`
- RFC PR:

# Summary

This clarifies the generic binary file format used for representing
_syntactic heaps_.  This can, for example, be used to store Abstract
Syntax Trees and is as the encoding for binary WyIL files.

# Motivation

This provides the first step in documenting the binary representation
of WyIL files.  These are essentially a binary encoding of Whiley
source files, and can contain additional information as necessary.
This document does not dictate the exact constraints of a WyIL file.
Rather, it describes the general encoding format upon which they are
built.

# Technical Details

The format presented is a binary file format, and the purpose of this
document is to clarify the exact representation used.

## Preliminaries

Various primitives are used for representing the overall encoding of a
syntactic heap.

### Fixed-Width Integer Encoding

Fixed width integers are denoted as `un` or `in` where `n` is a given
number of bits.  For exampled, `u8` represents an unsigned byte.
Fixed with integers are encoded as is (i.e. without any
meta-information) in a _big endian format_.  For example, consider
the following definition:

```
type Header is { u8 opcode, u16 operands }
```

An instance of `Header` is stored in a binary stream as follows:

```
+--------+----------------+
| opcode |    operands    |
+--------+----------------+
 0      7 8             23
```

Here, the first 8 bits represent the `opcode` field, and the remaining
16 bits represent the operand array.  Here, bit `0` represents the
_least significant bit_ of the `opcode`.  

**FIXME:** how to describe this in terms of bytes?x

**FIXME:** should be a little endian encoding?

### Variable-Width Integer Encoding

LEB128

### Array Encoding

### Padding

## Overview

## Item Encoding

## Data Encoding

# Terminology

# Drawbacks and Limitations

# Unresolved Issues
