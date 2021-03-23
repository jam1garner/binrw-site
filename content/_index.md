+++
title = "binrw"
sort_by = "weight"
+++

# binrw

## Features

* Generates efficient data parsers for structs and enums using `#[derive]`
* Reads data from any source using standard `io::Read + io::Seek` streams
* [Directives in attributes](https://docs.rs/binrw/latest/binrw/attribute)
  handle common binary parsing tasks like matching magic numbers, byte ordering,
  padding & alignment, data validation, and more
* Includes reusable types for common data structures like
  [null-terminated strings](https://docs.rs/binrw/latest/binrw/struct.NullString.html) and
  [data indirection using offsets](https://docs.rs/binrw/latest/binrw/struct.FilePtr.html)
* Parses types from third-party crates using
  [free functions](https://docs.rs/binrw/latest/binrw/attribute#custom-parsers)
  or [value maps](https://docs.rs/binrw/latest/binrw/attribute#map)
* Uses efficient in-memory representations (does not require `#[repr(C)]` or
  `#[repr(packed)]`)
* Code in attributes is written as code, not as strings
* Supports no_std
