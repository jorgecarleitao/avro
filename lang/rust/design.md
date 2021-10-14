# Design v0.2

This is a proposal to redesign the `avro` crate with performance in mind.

This design is motivated by the design outlined [here](https://github.com/jorgecarleitao/arrow2/blob/main/src/io/README.md).

## Goals

The goal of the `avro` crate is to offer a sound and fast implementation of avro format in Rust.

Non-goals:

* interoperate with other storage formats (e.g. json, parquet, csv)
* interoperate with other in-memory formats (e.g. arrow, numpy, `serde-json`)

## Design

### In-memory schema format

In-memory schema format refers to the specific structs and/or enums that declare a deserialized an avro schema.
Avro (and many other formats) have two distinct meta types:

* Logical types
* Physical types

Logical types have a many to one relationship with physical types, as defined [in the spec](https://avro.apache.org/docs/current/spec.html).

The proposal is to declare the types as

```
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum LogicalType {
    Float,
    Double,
    String,
    UUID,
    ...
    Enum(Box<Enum>), // Enum is a struct containing all required fields; `Box` reduces the size of `LogicalType`
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum PhysicalType {
    Float,
    Double,
    String,
    Bytes,
    ...
}

impl LogicalType {
    // the physical type associated to the logical type
    pub fn physical_type(&self) -> PhysicalType;
}

// support to read and write to json corresponding exactly to the schema definitions in avro, since the spec _is_ json.
impl Serialize for LogicalType {...};
impl Serialize for PhysicalType {...};
```

The background for this is that it allow users to more easily write schemas as well as using `match`.

### In-memory data format

In-memory data format in this context refers to the specific structs and/or enum that
declare a deserialized row. Formally, it is the physical layout of the data.
Because avro is not an in-memory format, the particular format we use is not part of
the spec but merely a tool to be able to use avro from Rust.

There are two main options: an `enum` and a trait object. We use an `enum`s:

```rust
pub enum Value {
    Float(f64),
    Int(i64),
    String(String),
    UUID(String),
    Bytes(Vec<u8>),
    Array(Vec<Value>),
}
```

An alternative implementation is to use 

```rust
pub enum Value {
    Float(f64),
    Int(i64),
    String(String, LogicalType),
    Bytes(Vec<u8>),
}
```

i.e. enumerate physical types and attached logical types to derive logic-specific
semantics to the consumers. This has the advantage of allowing functionality specific
to a physical type to be `matched` using `Value::String(v, _)`.

Note that this is a row-based representation (i.e. `Value` represents a single row).
This choice is driven by the format being row-based.

Note that this declaration is _not_ suitable for consumption by other formats. Specifically,

```
block -> Value -> target in-memory format
```

incurs an allocation per item for every non-copy type (e.g. `String`). This scales
with the number of items times the number of nodes, which is expensive. For this reason,
we _must_ expose, as part of the API, interfaces that allow to deserialize from avro
without commiting to `Value` (see below).

### CPU and IO

Reading a file is usually composed by two types of operations: CPU-bounded and IO-bounded.
The specific system or configuration executing them drives whether the overall operation is
CPU-bounded or IO-bounded. E.g. reading from s3 is usually IO-bounded, while reading from
disk is usually CPU-bounded.

For this reason, we separate IO-bounded APIs from CPU-bounded APIs. Users _may_ mix then in
the same thread for simplicity, but the API must allow for e.g. a single producer (of blocks)
multiple consumer pattern.

### Read, decompress and deserialize

The operation to bring avro into memory is composed by 4 main ops:

1. Read the metadata (`R: Read -> Schema`) (IO-bounded)
2. Read block to memory (`R: Read -> Vec<u8>`) (IO-bounded)
3. decompress block (`Vec<u8> -> Vec<u8>`) (CPU-bounded)
4. deserialize block (`Vec<u8> -> Vec<Value>`) (CPU-bounded)

The `sync` version of 1.-2. looks as follows (the `async` is very similar, via `futures::AsyncRead`):

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

##### Notes

* The iterator must be a `StreamingIterator` (as opposed to an `Iterator`) because we want to
  keep ownership of the internal buffer so that we can re-use on the next read. An iterator relinquishes ownership
  of the item, which in our case would imply that we could not re-use it.
* `FallibleStreamingIterator` because the `advance` may fail, when we can't read the block at all (e.g. networking)
* It is possible to write an equivalent `async` version of this using `AsyncRead`

#### Decompression

Decompression is CPU-bounded and benefits from re-using an internal buffer. We declare an abstraction to handle it:

```rust
pub struct Decompressor<'a, R: Read> {
    blocks: BlockStreamIterator<'a, R>,
    codec: Codec,
    buf: (Vec<u8>, usize),
    was_swapped: bool,
}

// streaming iterator so that we can re-use the buffer for other decompressions without re-allocations
impl<'a, R: Read> FallibleStreamingIterator for Decompressor<'a, R> {
    type Item = (Vec<u8>, usize);  // (block, number of rows)
    type Error = AvroError;
}
```

Like before, we use a streaming iterator so that we keep ownership of the buffer after `advance`. Note how `Decompressor`
is built on top of `BlockStreamIterator`. This API is not useful for multi-threaded programs because it requires mixing IO-bounded from CPU-bounded.
An equivalent API using `I: Iterator<(Vec<u8>, usize)>` is required for multi-threaded programs (that send blocks through a channel).
The advantage of this API is that it allows re-using both buffers at the same time (via `std::mem::swap`). See [here](https://github.com/jorgecarleitao/arrow2/blob/main/src/io/avro/read/mod.rs#L146) for an example implementation that uses `swap` to achieve this behavior.

#### Deserialize

Deserialization is CPU intensive. Here is a proposal to write such an operation:

```rust
/// Deserializes an individual item of the `block` into a `ValueRef`, returning a new where some bytes have been consumed.
/// Invariants:
/// `ValueRef` is consistent with `schema`
/// * `block.len() >=` `.len()` of the second item of the return type
fn deserialize_item<'a>(
    mut block: &[u8],
    schema: &AvroSchema,
) -> Result<(Value, &'a [u8])> {
    todo!("reads from block according to `schema`")
}


pub fn deserialize(
    mut block: &'a [u8],
    avro_schemas: &[AvroSchema],
    rows: &mut [Value<'a>],
) -> Result<()> {
    todo!("use deserialize_item, but over all rows (rows.len()). Re-use `rows` to avoid re-allocs")
}
```

Note that this operation commits to `Value`. Other in-memory formats need to write a
deserializer to their own in-memory format. The `deserialize` would be a reference implementation (for `Value`).

See [here](https://github.com/jorgecarleitao/arrow2/blob/main/src/io/avro/read/deserialize.rs#L1) for an implementation for the arrow format that
bipasses `Value` directly to `arrow`.


