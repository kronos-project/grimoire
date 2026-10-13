# DML Tables

This is a binary format which is used most notably for `LatestFileList.bin`.

All integers are in little-endian byte order unless otherwise specified.

Layouts use the book's [binary notation](../../notation.md). Offsets start at
each named structure. DML type tags in the mapping below retain their XML names;
structure tables use concrete wire types.

## Type Tag

When binary-serialized, the subsequent values of field types are identified by a
tag value dynamically. The following maps tags to their respective DML types:

| Tag | Type  |
| --- | ----- |
| 0   | GID   |
| 1   | INT   |
| 2   | UINT  |
| 3   | FLT   |
| 4   | BYT   |
| 5   | UBYT  |
| 6   | USHRT |
| 7   | DBL   |
| 8   | STR   |
| 9   | WSTR  |

## Message Orders

DML tables at present only use message types defined in the
`ExtendedBaseMessages` protocol.

| Index | Type             |
| ----- | ---------------- |
| 1     | MSG_CUSTOMDICT   |
| 2     | MSG_CUSTOMRECORD |

## Structure

The following documents the outline of a serialized table blob.

### Table

This structure and everything it encapsulates should be read as many times as
needed until EOF is reached.

Let `N` be `value_count`. `Value[N]` counts entries, not bytes; each entry has
its own size.

| Offset | Type       | Field       | Description                                  |
| ------ | ---------- | ----------- | -------------------------------------------- |
| `0x00` | `u32`      | value_count | Number of entries in the values array        |
| `0x04` | `Value[N]` | values      | Sequence of variable-length Value structures |

### Value

Let `S` be `size`, including the four-byte Value header.

| Offset | Type        | Field         | Description                                       |
| ------ | ----------- | ------------- | ------------------------------------------------- |
| `0x00` | `u8`        | protocol_id   | ExtendedBaseMessages service ID, recorded as `2`  |
| `0x01` | `u8`        | message_order | Structure selector from the message-order mapping |
| `0x02` | `u16`       | size          | Byte count including this header and its payload  |
| `0x04` | `u8[S - 4]` | payload       | Dictionary or record data                         |

These entries use a DML-style header without the network frame wrapper or
trailing zero. A parser can delimit the payload as `Size - 4` bytes. The table's
leading `UINT` counts entries, not bytes. The reviewed writer confirms this
header; its reader does not fully implement dictionary and record decoding, so
the template details below remain partly unverified.

### RecordTemplate

| Offset | Type             | Field         | Description                                             |
| ------ | ---------------- | ------------- | ------------------------------------------------------- |
| `0x00` | `RecordField[?]` | record_fields | Field definitions; count and termination are unverified |

### RecordField

Let `N` be `name_length`, a byte count.

| Offset     | Type    | Field       | Description                                                   |
| ---------- | ------- | ----------- | ------------------------------------------------------------- |
| `0x00`     | `u16`   | name_length | Name byte count                                               |
| `0x02`     | `u8[N]` | name        | Raw name bytes; `_TargetTable` denotes the target-table field |
| `0x02 + N` | `u8`    | type_tag    | Selector from the DML type-tag mapping                        |
| `0x03 + N` | `u8`    | flags       | Wire meanings unverified                                      |

### TargetTable

The original notes assume this is always part of a RecordTemplate. Its
placement, value encoding, and interaction with the field type/flags need
verification; the reviewed table implementation does not establish them.

Let `N` be this structure's `name_length`, a byte count.

| Offset | Type    | Field       | Description                                          |
| ------ | ------- | ----------- | ---------------------------------------------------- |
| `0x00` | `u16`   | name_length | Target-table name byte count                         |
| `0x02` | `u8[N]` | name        | Raw target-table name bytes, without a second prefix |

### Record

| Offset | Type    | Field  | Description                                             |
| ------ | ------- | ------ | ------------------------------------------------------- |
| `0x00` | `u8[?]` | values | Decode fields according to the preceding RecordTemplate |
