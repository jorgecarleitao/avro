# Design v0.2

This is a proposal to redesign the `avro` crate with performance in mind.

## Goals

The goal of the `avro` crate is to offer a sound implementation of avro format.

Non-goals:

* interoperate with other storage formats (e.g. json, parquet, csv)
* interoperate with other in-memory formats (e.g. arrow, numpy)

## Design

### In-memory format

For the crate to be useful, we need to commit to an in-memory format, so that users can e.g. read an avro file.
There are two main options: an `enum` and a trait object.

In Rust, there are two main "types" to store a value: owning and non-owning (a ref with a lifetime). We support both.
We use two different `enum`s: 

```
pub enum Value {
    ...
}

pub enum ValueRef<'a> {
    ...
}

impl ValueRef {
    /// converts itself to the owned version of [`ValueRef`]
    pub fn to_owned(&self) -> Value;
}
```

the first one owns the data (e.g. `Value::String(String)`), the second one owns a non-mutable reference of the data `Value::String(&str)`.

### CPU and IO

We divide the operations that are CPU-bounded from operations that are IO-bounded.
They can be distinguished by whether there is `R: Read` as a generic parameter, which denotes IO-bounded ops.

### Read, decompress and deserialize

Read blocks is IO-bounded and benefits from reusing the same buffer across blocks. The `sync` version looks as follows (the `async` is very similar):

```rust
/// Reads the avro metadata from `reader` into a [`Schema`], [`Codec`] and magic marker.
pub fn read_metadata<R: std::io::Read>(reader: &mut R) -> Result<(Vec<AvroSchema>, Schema, Codec, [u8; 16])>


/// Reads a block from the file into `buf`.
/// # Panic
/// Panics iff the block marker does not equal to the file's marker
/// used by [`BlockStreamIterator`] below.
fn read_block<R: Read>(reader: &mut R, buf: &mut Vec<u8>, file_marker: [u8; 16]) -> Result<usize>;


/// [`StreamingIterator`] of blocks of avro data
pub struct BlockStreamIterator<'a, R: Read> {
    buf: (Vec<u8>, usize),
    reader: &'a mut R,
    file_marker: [u8; 16],
}

// streaming iterator so that we can re-use the buffer across blocks without re-allocations
impl<'a, R: Read> FallibleStreamingIterator for BlockStreamIterator<'a, R> {
    type Item = (Vec<u8>, usize);  // (block, number of rows)
    type Error = AvroError;
}
```

Decompression is CPU intensive and benefits from re-using its buffer. We declare a zero-cost abstraction to handle both cases:

```rust
pub struct Decompressor<'a, R: Read>;

// streaming iterator so that we can re-use the buffer for other decompressions without re-allocations
impl<'a, R: Read> FallibleStreamingIterator for Decompressor<'a, R> {
    type Item = (Vec<u8>, usize);  // (block, number of rows)
    type Error = AvroError;
}
```

Deserialization is CPU intensive. Here is a proposal to write such an operation:

```rust
/// Deserializes an individual item of the `block` into a `ValueRef`, returning a new where some bytes have been consumed.
/// Invariants:
/// `ValueRef` is consistent with `schema`
/// * `block.len() >=` `.len()` of the second item of the return type
pub fn deserialize_item<'a>(
    mut block: &'a [u8],
    schema: &AvroSchema,
) -> Result<(ValueRef<'a>, &'a [u8])> {
    todo!("reads from block according to `schema`")
}


pub fn deserialize(
    mut block: &'a [u8],
    avro_schemas: &[AvroSchema],
    rows: &mut [ValueRef<'a>],
) -> Result<()> {
    todo!("same as deserialize_item, but over all rows (rows.len()). Re-use `rows` to avoid re-allocs")
}
```







