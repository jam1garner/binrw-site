+++
title = "Tutorial"
weight = 0
+++

# Tutorial

## Introduction

Before actually introducing how to use binrw, I'd like to introduce a little terminology as well as provide some direction regarding how to find things in the binrw documentation.

First off, binrw uses heavy use of Rust attributes, which look something like:

```rust
#[br(magic = b"PK")] // <--- attribute
struct ZipFile {
    // ...
}
```

binrw makes heavy use of attributes in order to describe modifications to parsing behavior.
Each item inside the `#[br(...)]` attribute is referred to as a "directive". Occasionally these
may also be referred to as "attributes" just due to the fact that, ignoring the `br` (binread) 
specifier, a lone directive is effectively an attribute.

```rust
#[br(magic = b"PK")]
//   ^^^^^^^^^^^^^
//     directive
```

Since attributes are such an important part of binrw as a language, they are organized into their
own section, separated into [directives used while reading](https://docs.rs/binrw/latest/binrw/attribute/read/index.html) and [directives used while writing](https://docs.rs/binrw/latest/binrw/attribute/write/index.html).

Since those pages are so commonly referenced, links to them have been included at the beginning of the top-level
documentation of the crate:

![A table labelled "quick links" including `#[br]`, `#[bw]`, and a few other options is shown](https://user-images.githubusercontent.com/8260240/149064578-240e583f-b894-4adc-986e-6f95fb846087.png)

The `#[br]` link takes you to the reading docs, and the `#[bw]` link takes you to the writing docs.

## The Simplest Parser

To start us off we have the basics. The simplest case in parsing is to take a couple fixed-width types and parse them one after another. Even in complicated formats you'll typically see plenty of this, so binrw tries to keep this as simple as possible:

```rust
use binrw::binrw;

#[binrw]
struct Vec3 {
    x: f32,
    y: f32,
    z: f32,
}
```

And we've now finished writing our first parser and writer! Let's try actually using it:

```rust
use binrw::{BinReaderExt, BinWriterExt}; // extension traits for use with readers and writers
use binrw::io::{Cursor, Seek, SeekFrom}; // A no_std reimplementation of std::io

let position = Vec3 { x: 3.0, y: 1.0, z: 0.0 };
let mut writer = Cursor::new(Vec::new());

// Write our position to the Vec with little endian (le) byteorder
writer.write_le(&position).unwrap();

// Read our position back out of our Vec
let mut reader = writer;
reader.seek(SeekFrom::Start(0)).unwrap();
let pos: Vec3 = reader.read_le().unwrap();

println!("Position: {}, {}, {}", pos.x, pos.y, pos.z);
```

binrw provides extension traits in order to enable taking existing types implementing `std::io::Read` or
`std::io::Write` and use them with binrw. See [`BinReaderExt`](https://docs.rs/binrw/latest/binrw/trait.BinReaderExt.html) and [`BinWriterExt`](https://docs.rs/binrw/latest/binrw/trait.BinWriterExt.html) for more info.

For `#![no_std]` projects, `binrw::io` is a drop-in replacement for `std::io`. In fact, it's encouraged to import from these locations *regardless* in binrw projects as, if the `std` feature is enabled, the traits are just re-exported from the standard library.

## Lists

The thing is, just parsing fixed-sized data is rarely enough for most formats. So the next most common technique in file formats is items in a list with a count of how many items are in the list.

The way this is handled in binrw is using the `count` directive:

```rust
#[binread]
struct Mesh {
    vertex_count: u32,

    #[br(count = vertex_count + 1)]
    vertices: Vec<Vec3>,
}
```

(Note: `#[binread]` and `#[binwrite]` are also provided for when you only need one, while `#[binrw]` does both)

The count directive can reference any fields which come before it, and arbitrary expressions are allowed on the
right hand side. For user convenience, the expression may evaluate to any type which can be cast to `usize`.

However the above only support reading. In order to make it support writing, we'll need to add some additional annotations to tell binrw how to take the length of the vertices field in order to use it as the value for `vertex_count` when writing:

```rust
#[binrw]
struct Mesh {
    #[bw(calc = (vertices.len() - 1) as u32)]
    vertex_count: u32,

    #[br(count = vertex_count + 1)]
    vertices: Vec<Vec3>,
}
```

Here we mix a `#[br]` attribute to describe the reading behavior with a `#[bw]` attribute to describe writing behavior. On occasion you might want to apply the same attribute to both (such as when specifying endianess). For that purpose there is a `#[brw(...)]` attribute.

||
|:-:|
| [*Gist of the example code up to this point*](https://gist.github.com/jam1garner/966a9a0f3ba765fc92d8d58c340aa5da) |

## Enums

binrw features enums in two forms: as sum types (enums with data) or as a limited set of values (C-style enums).

By default, enum variants are parsed in-order, moving from one to the next, parsing each until an error occurs. Typically, this is done via assertions. binrw has two main forms of assertions included: the [`assert` directive](https://docs.rs/binrw/latest/binrw/attribute/read/index.html#assert), which allows providing a condition that must hold true in order, otherwise an error is created. The other form is the [`magic` directive](https://docs.rs/binrw/latest/binrw/attribute/read/index.html#magic), which is a constant value that is checked to match a value in the reader being parsed from. This is commonly used for reading "file magics", or a fixed set of bytes included at the start of the file to confirm that the data is that of the expected file type.

For example, the following can be used to ensure the first 4 bytes of a file are "MODL":

```rust
#[binrw]
#[brw(magic = b"MODL")]
struct ModelFile {
    // ...
}
```

When using magic with `#[brw(...)]`, the constant will also be written in the BinWrite implementation, making the operation capable of round-tripping without additional effort. Magics can also be integer literals (with type specified, such as `0x1234_u32`).

```rust
#[binread]
enum Value {
    #[br(magic = 0u8)] U16(u16),
    #[br(magic = 1u8)] U32(u32),
    #[br(magic = 2u8)] String {
        len: u32,

        #[br(count = len)]
        bytes: Vec<u8>,
    },
}
```

For example, if we apply this parser (big endian) to the following data:

```
01 00 00 00 05
```

The steps it'd take is:

1. Read a single byte from the reader
2. Check if that byte is 0. Since it isn't, move on.
3. Check if that byte is 1. Since it is, continue parsing the Value::U32 variant.
4. Take 4 bytes from the stream, convert to a u32, construct a Value::U32 from it.

## C-like Enums

The other form of enums is C-like enums. They have no associated data, but are less verbose to write. While one *could* write such an enum like this:

```rust
#[binrw]
enum Compression {
    #[brw(magic = 0u8)] None,
    #[brw(magic = 1u8)] Zlib,
    #[brw(magic = 2u8)] Gzip,
    #[brw(magic = 3u8)] Lzma,
}
```

But it's a bit repetitive, especially if you already are specifying the values for FFI (as then you'd be specifying the value twice!). So instead, binrw has a shorthand for this, with the added benefit that it guarantees an efficient parser generation:

```rust
#[binrw]
#[brw(repr(u8))]
enum Compression {
    None = 0,
    Zlib = 1,
    Gzip = 2,
    Lzma = 3,
}
```

And, like typical [C-like enums](https://doc.rust-lang.org/rust-by-example/custom_types/enum/c_like.html) you don't actually have to specify each number. The rules for their relationship to numeric values is identical, with the first value defaulting to zero unless otherwise specified, each variant increasing by one unless manually specified.
