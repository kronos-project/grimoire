# Binary Layout Notation

Binary structure tables use `Offset | Type | Field | Description`.

- Offsets count bytes from the start of the structure named immediately above
  the table. Nested structures start again at `0x00`.
- Fixed offsets use hexadecimal, such as `0x0A`. Expressions such as `0x12 + L`
  locate fields after variable-length data. Each table defines its length
  variables; `?` means the offset or length is unverified.
- `u8`, `u16`, `u32`, and `u64` are unsigned integers of the indicated bit
  width. `i8`, `i16`, and `i32` are signed integers. `f32` and `f64` are
  IEEE-754 floating-point values. Multibyte values are little-endian unless
  explicitly stated otherwise.
- `T[N]` is an array of `N` elements of type `T`. For `u8[N]`, `N` is also the
  byte count. For `u16[N]`, the byte count is `2 * N`.
- Text and opaque bytes both use `u8[N]`; the description supplies the encoding
  and whether a terminator is included. Length prefixes occupy separate rows and
  are not included in the array unless stated.
- A byte-sized flag uses `u8` with its values in the description. A packed
  ObjectProperty boolean occupies one bit; it is not a byte-sized flag.
- Field names label wire values for reference and are not transmitted. They do
  not prescribe identifiers in a client or server implementation.

DML schema type names such as `STR`, `UINT`, and `GID` remain unchanged in type
mappings and XML examples. Binary layouts use the concrete wire types above.
Enum, status, and bit-meaning tables are mappings rather than layouts.
