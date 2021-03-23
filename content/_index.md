+++
title = "binrw"
sort_by = "weight"
+++

# binrw

## Declarative

Skip the boilerplate. Declaring a type is all you need.

```rust
#[derive(BinRead, BinWrite)]
#[brw(little, magic = b"PK")]
struct ZipHeader {
    version: u16,
    flags: u16,
    comp_type: CompressionType,
    // ...
}

#[derive(BinRead, BinWrite)]
#[brw(repr(u16))]
enum CompressionType {
    None = 0,
    Shrunk = 1,
    Deflated = 8,
    Lzma = 14,
}
```

Simple definitions and easy-to-read by design to improve maintainability and approachability.

## Powerful

Directives in attributes handle common binary parsing tasks like matching magic numbers, byte ordering, padding, alignment, data validation, and more.

```rust
#[derive(BinRead)]
struct StringTable {
    entry_count: u32,
    #[br(count = entry_count)]
    entries: Vec<NullString>,
}
```

## Composable

With support for generics, custom parsers, and mapping functions, you can create your own building blocks as reusable types. And to keep you from reinventing the wheel we already include types you'll likely need:

* Null-terminated strings (UTF-8 and UTF-16)
* Types for representing data indirection via file offsets
* Implementations for Vecs, Tuples, Arrays, and other standard Rust types

## Safe, Efficient and Embedded-Compatible

* Zero unsafe code
* Out of the box `no_std` and Web Assembly support
* Takes advantage of Rust's speed and safety
