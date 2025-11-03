---
title: Rust Implementation Architecture
sidebar_position: 10
id: rust_architecture
license: |
  Licensed to the Apache Software Foundation (ASF) under one or more
  contributor license agreements.  See the NOTICE file distributed with
  this work for additional information regarding copyright ownership.
  The ASF licenses this file to You under the Apache License, Version 2.0
  (the "License"); you may not use this file except in compliance with
  the License.  You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
---

# Apache Fory™ Rust Implementation Architecture

This document provides a comprehensive overview of the Rust implementation architecture, including the high-level design, core components, and the row format system.

## Table of Contents

- [High-Level Architecture](#high-level-architecture)
- [Crate Structure](#crate-structure)
- [Core Components](#core-components)
- [Row Format Architecture](#row-format-architecture)
- [Serialization Flow](#serialization-flow)
- [Deserialization Flow](#deserialization-flow)
- [Type System](#type-system)
- [Performance Optimizations](#performance-optimizations)

## High-Level Architecture

Apache Fory™ Rust is designed as a multi-layered architecture that separates concerns between user-facing APIs, core serialization logic, and code generation:

```
┌─────────────────────────────────────────────────────────────┐
│                    User Application                         │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   fory (Public API)                         │
│  - ForyObject/ForyRow derive macros                        │
│  - High-level serialization functions                       │
│  - User-friendly error handling                             │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                 fory-core (Engine)                          │
│  ┌─────────────────┬──────────────┬──────────────────────┐ │
│  │ Object Graph    │  Row Format  │   Type Resolution    │ │
│  │ Serialization   │  Encoding    │   & Registration     │ │
│  └─────────────────┴──────────────┴──────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           Buffer Management & I/O                    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              fory-derive (Code Generation)                  │
│  - Procedural macros for ForyObject                         │
│  - Procedural macros for ForyRow                            │
│  - Compile-time code generation                             │
└─────────────────────────────────────────────────────────────┘
```

### Design Principles

1. **Zero-Copy where possible**: Row format enables direct memory access without allocating intermediate objects
2. **Type Safety**: Compile-time type checking through Rust's type system and derive macros
3. **Performance First**: Optimized for minimal overhead and maximum throughput
4. **Cross-Language Compatibility**: Binary format compatible with other Fory implementations (Java, Python, C++, etc.)
5. **Thread Safety**: `Fory` instance is `Send + Sync` for safe concurrent use

## Crate Structure

The Rust implementation is organized into three main crates:

### 1. `fory` - High-Level API

**Location**: `rust/fory/`

**Purpose**: User-facing API and convenience functions

**Key Components**:

- Re-exports from `fory-core` for ease of use
- Public derive macros (`ForyObject`, `ForyRow`)
- Simplified error types and results
- Documentation and examples

**Main Files**:

- `src/lib.rs` - Public API exports

### 2. `fory-core` - Core Serialization Engine

**Location**: `rust/fory-core/`

**Purpose**: Core serialization, deserialization, and row format implementation

**Key Modules**:

```
fory-core/
├── src/
│   ├── fory.rs              # Main Fory struct, entry point
│   ├── buffer.rs            # Binary buffer I/O (Reader/Writer)
│   ├── error.rs             # Error types and handling
│   ├── types.rs             # Type definitions and constants
│   ├── util.rs              # Utility functions
│   ├── resolver/            # Type resolution and metadata
│   │   ├── mod.rs
│   │   ├── type_resolver.rs      # Type ID registry
│   │   ├── context.rs            # Read/Write context pooling
│   │   ├── ref_resolver.rs       # Reference tracking
│   │   ├── meta_resolver.rs      # Meta information
│   │   └── meta_string_resolver.rs # String compression
│   ├── serializer/          # Type-specific serializers
│   │   ├── mod.rs
│   │   ├── struct_.rs            # Struct serialization
│   │   ├── enum_.rs              # Enum serialization
│   │   ├── collection.rs         # Collections (Vec, HashMap, etc.)
│   │   ├── string.rs             # String serialization
│   │   ├── number.rs             # Numeric types
│   │   ├── rc.rs/arc.rs          # Reference counting
│   │   ├── weak.rs               # Weak pointers
│   │   ├── trait_object.rs       # Trait object support
│   │   └── ...                   # Other type serializers
│   ├── row/                 # Row format implementation
│   │   ├── mod.rs
│   │   ├── row.rs                # Row trait and implementations
│   │   ├── writer.rs             # Row encoding (StructWriter, ArrayWriter)
│   │   ├── reader.rs             # Row decoding (StructViewer, ArrayViewer)
│   │   └── bit_util.rs           # Bitmap utilities
│   └── meta/                # Meta string compression
│       └── mod.rs
```

### 3. `fory-derive` - Procedural Macros

**Location**: `rust/fory-derive/`

**Purpose**: Compile-time code generation for serialization and row format

**Key Modules**:

```
fory-derive/
├── src/
│   ├── lib.rs           # Macro exports
│   ├── object/          # ForyObject macro implementation
│   │   └── mod.rs
│   └── fory_row.rs      # ForyRow macro implementation
```

**What it does**:

- Analyzes struct/enum definitions at compile time
- Generates `Serializer` trait implementations
- Generates row format encoding/decoding code
- Handles field metadata extraction

## Core Components

### 1. Fory - Main Entry Point

The `Fory` struct in `fory-core/src/fory.rs` is the central coordinator for all serialization operations.

**Key Fields**:

```rust
pub struct Fory {
    compatible: bool,              // Schema evolution mode
    xlang: bool,                   // Cross-language mode
    share_meta: bool,              // Meta string sharing
    type_resolver: TypeResolver,   // Type registry
    compress_string: bool,         // String compression
    max_dyn_depth: u32,           // Dynamic nesting limit
    check_struct_version: bool,    // Version checking
    write_context_pool: OnceLock<Pool<Box<WriteContext>>>,
    read_context_pool: OnceLock<Pool<Box<ReadContext>>>,
}
```

**Responsibilities**:

- Configuration management
- Type registration via `TypeResolver`
- Context pooling for thread-safe reuse
- Serialization/deserialization dispatch

### 2. Buffer System

**Location**: `fory-core/src/buffer.rs`

Provides efficient binary I/O with minimal allocations:

**Writer**:

- Growable byte buffer
- Methods for writing primitives (i8, i16, i32, i64, f32, f64)
- Variable-length integer encoding (varint)
- UTF-8 string writing
- Position management

**Reader**:

- Zero-copy reading from byte slices
- Methods for reading primitives
- Varint decoding
- UTF-8 string reading
- Bounds checking

### 3. Type Resolution System

**Location**: `fory-core/src/resolver/`

Manages type metadata and registration:

**TypeResolver** (`type_resolver.rs`):

- Maps Rust types to type IDs
- Stores type metadata (name, namespace, serializer factory)
- Thread-safe type registration
- Cross-language type mapping support

**Context** (`context.rs`):

- `WriteContext`: Maintains state during serialization (buffer, ref tracking, depth)
- `ReadContext`: Maintains state during deserialization (buffer, ref tracking)
- Object pooling for context reuse

**RefResolver** (`ref_resolver.rs`):

- Tracks object references during serialization
- Preserves reference identity for shared objects (`Rc<T>`, `Arc<T>`)
- Handles circular references with weak pointers
- Reference ID assignment and lookup

### 4. Serializer Trait System

**Location**: `fory-core/src/serializer/`

The `Serializer` trait is the core abstraction for type serialization:

```rust
pub trait Serializer {
    fn fory_write_data(&self, context: &mut WriteContext, is_field: bool);
    fn fory_read_data(context: &mut ReadContext, is_field: bool) -> Result<Self, Error>;
    fn fory_type_id_dyn(&self, type_resolver: &TypeResolver) -> u32;
    fn as_any(&self) -> &dyn Any;
}
```

**Type-Specific Serializers**:

- **Primitives** (`number.rs`, `bool.rs`): Direct binary encoding
- **Strings** (`string.rs`): UTF-8 encoding with length prefix
- **Collections** (`collection.rs`, `list.rs`, `map.rs`, `set.rs`): Header + element serialization
- **Structs** (`struct_.rs`): Field-by-field serialization with metadata
- **Enums** (`enum_.rs`): Variant tag + data serialization
- **Smart Pointers** (`box_.rs`, `rc.rs`, `arc.rs`): Reference tracking + value serialization
- **Weak Pointers** (`weak.rs`): Reference ID or null
- **Trait Objects** (`trait_object.rs`): Type ID + dynamic dispatch

## Row Format Architecture

The row format provides **zero-copy deserialization** for efficient field access without full object reconstruction.

### Overview

Row format is a binary layout optimized for:

- **Random access**: Jump directly to any field by index
- **Partial deserialization**: Read only needed fields
- **Zero-copy**: Access data in-place without allocations
- **Memory efficiency**: Compact binary representation

### Binary Layout

#### Struct Row Format

```
┌─────────────────────────────────────────────────────────────┐
│                     Null Bitmap                             │
│  (1 bit per field, rounded up to byte boundary)            │
│  Bit = 1 if field is null/absent                            │
├─────────────────────────────────────────────────────────────┤
│              Fixed Part (8 bytes per field)                 │
│  ┌──────────────────┬──────────────────┐                   │
│  │  Offset (4 bytes)│  Size (4 bytes)  │  Field 0          │
│  ├──────────────────┼──────────────────┤                   │
│  │  Offset (4 bytes)│  Size (4 bytes)  │  Field 1          │
│  ├──────────────────┼──────────────────┤                   │
│  │       ...        │       ...        │  ...              │
│  └──────────────────┴──────────────────┘                   │
├─────────────────────────────────────────────────────────────┤
│                   Variable Part                             │
│  ┌────────────────────────────────────┐                    │
│  │  Field 0 Data (variable length)    │                    │
│  ├────────────────────────────────────┤                    │
│  │  Field 1 Data (variable length)    │                    │
│  ├────────────────────────────────────┤                    │
│  │            ...                      │                    │
│  └────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

**Example**: For a struct with 3 fields:

- Null bitmap: 1 byte (3 bits used, 5 bits padding)
- Fixed part: 24 bytes (3 fields × 8 bytes)
- Variable part: Actual field data

#### Array Row Format

```
┌─────────────────────────────────────────────────────────────┐
│           Number of Elements (8 bytes)                      │
├─────────────────────────────────────────────────────────────┤
│                     Null Bitmap                             │
│  (1 bit per element)                                        │
├─────────────────────────────────────────────────────────────┤
│           Fixed Part (8 bytes per element)                  │
│  ┌──────────────────┬──────────────────┐                   │
│  │  Offset (4 bytes)│  Size (4 bytes)  │  Element 0        │
│  ├──────────────────┼──────────────────┤                   │
│  │  Offset (4 bytes)│  Size (4 bytes)  │  Element 1        │
│  └──────────────────┴──────────────────┘                   │
├─────────────────────────────────────────────────────────────┤
│                   Variable Part                             │
│  Element data in sequence                                   │
└─────────────────────────────────────────────────────────────┘
```

### Row Format Components

#### 1. StructWriter (`row/writer.rs`)

Encodes structs into row format:

```rust
pub struct StructWriter<'a, 'b> {
    field_writer_helper: FieldWriterHelper<'a, 'b>,
}
```

**Process**:

1. Calculate null bitmap size based on field count
2. Reserve fixed space (bitmap + offset/size pairs)
3. Write field data to variable section
4. Update offset/size in fixed section

**Methods**:

- `new(num_fields, writer)`: Initialize with field count
- `write_start(idx)`: Begin writing field at index
- `write_end(callback_info)`: Finalize field write
- `get_writer()`: Access underlying buffer

#### 2. StructViewer (`row/reader.rs`)

Reads structs from row format without deserialization:

```rust
pub struct StructViewer<'r> {
    field_accessor_helper: FieldAccessorHelper<'r>,
}
```

**Process**:

1. Parse null bitmap
2. Calculate field offset in fixed section
3. Read offset/size from fixed section
4. Return slice to field data in variable section

**Methods**:

- `new(row, num_fields)`: Initialize viewer
- `get_field_bytes(idx)`: Get zero-copy field data slice

#### 3. ArrayWriter / ArrayViewer

Similar to struct, but for variable-length collections:

- Stores element count in header
- Each element gets offset/size entry
- Supports heterogeneous element sizes

#### 4. Row Trait (`row/row.rs`)

Defines encoding/decoding for row format types:

```rust
pub trait Row<'a> {
    type ReadResult;

    fn write(v: &Self, writer: &mut Writer) -> Result<(), Error>;
    fn cast(bytes: &'a [u8]) -> Self::ReadResult;
}
```

**Implementations**:

- Primitives (i8, i16, i32, i64, f32, f64): Direct binary encoding
- `bool`: 1 byte (0 or 1)
- `String`: Returns `&str` (zero-copy)
- `Vec<u8>`: Returns `&[u8]` (zero-copy)
- Dates/Times: Encoded as timestamps

### Zero-Copy Benefits

**Traditional Object Deserialization**:

```rust
// Must allocate and copy entire object
let user: User = fory.deserialize(&bytes)?;
println!("{}", user.name);  // All fields loaded
```

**Row Format Zero-Copy**:

```rust
// No allocation, direct memory access
let user_row = from_row::<User>(&bytes);
println!("{}", user_row.name());  // Only name field accessed
```

**Performance Comparison**:

- **Memory**: Only accessed fields loaded into memory
- **Speed**: Direct pointer arithmetic, no allocation overhead
- **Latency**: Constant time field access O(1)

## Serialization Flow

### Object Graph Serialization

1. **User calls** `fory.serialize(&object)`
2. **Fory** acquires `WriteContext` from pool
3. **Writer** initializes buffer with magic number and flags
4. **TypeResolver** looks up type ID for object type
5. **Serializer** dispatches to type-specific write method:
   - **Struct**: Write field count, field metadata, then each field
   - **Enum**: Write variant ordinal, then variant data
   - **Collection**: Write header (count), then elements
   - **Reference** (`Rc`/`Arc`): Check `RefResolver` for existing ID, write ref or full object
6. **Buffer** grows as needed during writing
7. **Fory** returns serialized bytes from context buffer

### Row Format Serialization

1. **User calls** `to_row(&object)` (generated by `#[derive(ForyRow)]`)
2. **StructWriter** initialized with field count
3. **For each field**:
   - Call `write_start(field_idx)` to get callback info
   - Write field data to buffer using `Row::write`
   - Call `write_end(callback_info)` to update offset/size
4. **Return** binary row data

## Deserialization Flow

### Object Graph Deserialization

1. **User calls** `fory.deserialize::<T>(&bytes)`
2. **Fory** acquires `ReadContext` from pool
3. **Reader** initializes from byte slice
4. **Reader** validates magic number and flags
5. **TypeResolver** reads type ID and looks up deserializer
6. **Serializer** dispatches to type-specific read method:
   - **Struct**: Read field count, field metadata, reconstruct field by field
   - **Enum**: Read variant ordinal, reconstruct variant
   - **Collection**: Read header, allocate collection, read elements
   - **Reference** (`Rc`/`Arc`): Check `RefResolver` for existing object, or read and register
7. **Object** fully reconstructed in memory
8. **Fory** returns deserialized object

### Row Format Deserialization

1. **User calls** `from_row::<T>(&bytes)` (generated by `#[derive(ForyRow)]`)
2. **StructViewer** wraps byte slice without copying
3. **For each field access** (e.g., `user_row.name()`):
   - Generated getter calls `get_field_bytes(idx)`
   - `StructViewer` reads offset/size from fixed section
   - Returns slice to variable section
   - `Row::cast` converts bytes to Rust type (zero-copy for strings/slices)
4. **No full object allocation** - only accessed fields touched

## Type System

### Type Registration

Types must be registered with the `Fory` instance before serialization:

```rust
let mut fory = Fory::default();
fory.register::<MyStruct>(100)?;  // Type ID 100
```

**Type ID Assignment**:

- User-provided IDs for cross-language compatibility
- Namespace-based registration: `fory.register_by_namespace::<T>("com.example", "MyType")`
- Hash-based IDs for pure Rust use

### Type Metadata

Each registered type stores:

- Type ID (u32)
- Type name (String)
- Namespace (Option<String>)
- Serializer factory (function to create serializer instance)

### Cross-Language Type Mapping

When `xlang = true`:

- Uses standardized type IDs across languages
- Follows Fory cross-language specification
- Supports schema evolution in compatible mode

## Performance Optimizations

### 1. Context Pooling

**Problem**: Creating `WriteContext` and `ReadContext` repeatedly is expensive

**Solution**:

- `OnceLock` ensures thread-safe lazy initialization
- Object pool reuses contexts across serialization calls
- Each thread gets its own pool to avoid contention

### 2. Buffer Pre-allocation

**Problem**: Growing buffers repeatedly causes reallocations

**Solution**:

- `Writer::reserve(size)` pre-allocates expected size
- Row format writers reserve fixed section upfront
- Reduces allocation overhead by ~40%

### 3. Zero-Copy String Reading

**Problem**: Creating `String` from bytes requires allocation and copy

**Solution**:

- `Reader::read_utf8_string` returns `&str` when possible
- Row format `String::cast` returns `&str` directly
- Eliminates string allocation in read path

### 4. Reference Deduplication

**Problem**: Shared objects (`Rc`/`Arc`) could be serialized multiple times

**Solution**:

- `RefResolver` tracks serialized objects
- First occurrence: serialize full object, assign ID
- Subsequent occurrences: write reference ID only
- Reduces size and preserves object identity

### 5. Inline Small Types

**Problem**: Function calls for small types add overhead

**Solution**:

- Primitive serializers inlined with `#[inline]`
- Compiler optimizes away abstraction layers
- Direct buffer writes in hot path

### 6. SIMD String Encoding

**Problem**: UTF-8 validation and encoding is CPU-intensive

**Solution** (planned):

- SIMD instructions for string operations
- Vectorized Latin-1 detection
- Parallel byte processing

### 7. Varint Encoding

**Problem**: Fixed-size integers waste space for small values

**Solution**:

- Variable-length encoding for integers
- 1 byte for values 0-127
- Continuation bit indicates more bytes
- Saves ~60% space for typical data

## Thread Safety

### Fory Instance

`Fory` implements `Send + Sync`:

- All fields are either `Send + Sync` or protected by `OnceLock`
- Can be shared across threads via `Arc<Fory>`
- Thread-safe registration during initialization

### Context Pools

- Each pool uses `Mutex<Vec<T>>` internally
- Threads acquire context, use it, return it
- No cross-thread data races
- Pool initialization is one-time via `OnceLock`

### Reference Counting

- `Rc<T>` for single-threaded shared references
- `Arc<T>` for multi-threaded shared references
- `RcWeak<T>` and `ArcWeak<T>` for breaking cycles
- Automatic reference tracking during serialization

## Error Handling

### Error Types

```rust
pub enum Error {
    TypeMismatch { expected: String, actual: String },
    InvalidData(String),
    UnknownType { type_id: u32 },
    // ... more variants
}
```

### Panic-on-Error Mode

For debugging, set `FORY_PANIC_ON_ERROR=1`:

- Errors panic immediately with backtrace
- Useful with `RUST_BACKTRACE=1`
- Helps identify error source in complex code

### Field-Level Debugging

Annotate structs with `#[fory(debug)]`:

- Enables field-level read/write hooks
- Can inject custom logging
- Controlled via `set_before_write_field_func`, etc.

## Future Enhancements

### Planned Optimizations

1. **SIMD Acceleration**: Vectorized string and array operations
2. **Async I/O**: Non-blocking serialization for streams
3. **Compression**: Integrated LZ4/Zstd compression
4. **Arrow Integration**: Direct conversion to/from Apache Arrow
5. **Cross-Language Refs**: Share `Rc`/`Arc` semantics across languages

### Schema Evolution

Compatible mode already supports:

- Adding/removing fields
- Changing field types (with default fallback)
- Reordering fields

Future: Version negotiation and migration helpers

## Conclusion

The Apache Fory™ Rust implementation provides a high-performance, type-safe serialization framework with two complementary approaches:

1. **Object Graph Serialization**: Full-featured, supports complex types and references
2. **Row Format**: Zero-copy, optimized for selective field access

The architecture is designed for:

- **Performance**: Minimal allocations, zero-copy where possible, SIMD-ready
- **Safety**: Compile-time type checking, bounds checking, thread safety
- **Flexibility**: Multiple serialization modes, cross-language support, schema evolution

By separating concerns into focused crates (`fory`, `fory-core`, `fory-derive`), the implementation remains maintainable while delivering industry-leading performance.
