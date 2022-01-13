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
