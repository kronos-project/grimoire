# DML Tables

This is a binary format which is used most notably for `LatestFileList.bin`.

All integers are in little-endian byte order unless otherwise specified.

## Type Tag

When binary-serialized, the subsequent values of field types are identified
by a tag value dynamically. The following maps tags to their respective
DML types:

| Tag | Type  |
|-----|-------|
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

DML tables at present only use message types defined in the `ExtendedBaseMessages`
protocol.

| Index | Type             |
|-------|------------------|
| 1     | MSG_CUSTOMDICT   |
| 2     | MSG_CUSTOMRECORD |

## Structure

The following documents the outline of a serialized table blob.

### Table

This structure and everything it encapsulates should be read as many times as needed until EOF is reached.

| Name         | Type    | Description                |
|--------------|---------|----------------------------|
| ValuesLength | UINT    | Length of the Values array |
| Values       | Value[] | Array of Value structures  |

### Value

| Name       | Type  | Description                                                                         |
|------------|-------|-------------------------------------------------------------------------------------|
| ProtocolID | UBYT  | ID of `ExtendedBaseMessages` protocol (always 2)                                    |
| -          | UBYT  | Type of structure that follows according to the [Message Order](#message-orders)    |
| Size       | USHRT | Size including this four-byte header and its payload                                |

These entries use a DML-style header without the network frame wrapper or
trailing zero. A parser can delimit the payload as `Size - 4` bytes. The
table's leading `UINT` counts entries, not bytes. The reviewed writer
confirms this header; its reader does not fully implement dictionary and
record decoding, so the template details below remain partly unverified.

### RecordTemplate

| Name         | Type          | Description                     |
|--------------|---------------|---------------------------------|
| RecordFields | RecordField[] | Array of RecordField structures |

### RecordField

| Name   | Type  | Description                                                                                                            |
|--------|-------|------------------------------------------------------------------------------------------------------------------------|
| Length | USHRT | Length prefix of Name                                                                                                  |
| Name   | uint8[Length] | Raw name bytes; no second STR prefix. `_TargetTable` denotes the target-table field. |
| Type   | UBYT  | Type of the field according to the [Type Tags](#type-tag) table                                                        |
| ?      | UBYT  | DML flags; wire meanings not yet verified                                                                               |

### TargetTable

The original notes assume this is always part of a RecordTemplate. Its
placement, value encoding, and interaction with the field type/flags need
verification; the reviewed table implementation does not establish them.

| Name   | Type  | Description                                          |
|--------|-------|------------------------------------------------------|
| Length | USHRT | Length prefix of Name                                |
| Name   | uint8[Length] | Raw target-table name bytes; no second prefix |

### Record

| Name | Type | Description                                                                         |
|------|------|-------------------------------------------------------------------------------------|
| -    | -    | Values should be read according to the RecordFields in the preceding RecordTemplate |
