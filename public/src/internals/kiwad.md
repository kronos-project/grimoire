# KIWAD Archives

The "KingsIsle Where's All the Data?" format (name guessed) is a custom file
format used by the game to bundle all kinds of game assets in archives. The
format presumably draws inspiration from
[the original Doom WAD format](https://doomwiki.org/wiki/WAD#Header) and
similarly uses `.wad` as file extension.

Archives consist of a [header](#header), followed by the
[file table](#file-table), and lastly the encoded file data as described in the
table. All data is assumed to be encoded in little-endian byteorder unless
stated otherwise.

Layouts use the book's [binary notation](../notation.md). Header offsets start
at the archive; file-entry offsets start at the individual table entry.

## Header

The header may consume 13 or 14 bytes in size and outlines the amount of files
to expect in the archive:

| Offset | Type    | Field      | Description                         |
| ------ | ------- | ---------- | ----------------------------------- |
| `0x00` | `u8[5]` | magic      | ASCII `KIWAD`, without a terminator |
| `0x05` | `u32`   | version    | Archive format version              |
| `0x09` | `u32`   | file_count | Number of file entries              |
| `0x0D` | `u8`    | flags      | Present only when `version >= 2`    |

### Flags

On file format versions >= 2, the header encodes a mask of configuration bits
which should be treated as implementation-defined for the most parts; KIWAD
implementations can safely ignore these bit flags and always write `0`.

This is how the official client uses them:

| Bit | Name                       |
| --- | -------------------------- |
| 0   | Memory-maps the file view  |
| 1   | Prefetches the file handle |

## File table

The reviewed reader consumes the flags byte but does not verify the official
client's bit meanings; those above remain original research notes.

This table consists of `n` repetitions of the [file](#file) structure, where `n`
is the number of archived files as encoded in the [header](#header).

## File

This structure denotes the metadata of an encoded file, needed for extraction
and validation:

Let `P` be `path_length`, including the terminating zero byte.

| Offset | Type    | Field             | Description                                             |
| ------ | ------- | ----------------- | ------------------------------------------------------- |
| `0x00` | `u32`   | data_offset       | Absolute file-data offset from the start of the archive |
| `0x04` | `u32`   | uncompressed_size | Uncompressed byte count                                 |
| `0x08` | `i32`   | compressed_size   | Compressed byte count; see the sentinel caveat below    |
| `0x0C` | `u8`    | is_compressed     | `0` for raw data; nonzero for zlib-compressed data      |
| `0x0D` | `u32`   | checksum          | CRC-32 of stored file data                              |
| `0x11` | `u32`   | path_length       | Path byte count, including its terminating zero         |
| `0x15` | `u8[P]` | path              | Null-terminated archive path                            |

### Compression

If a file is compressed, the corresponding file table entry encodes its
uncompressed size and the compressed size and the data will run through
[zlib](https://zlib.net).

The original notes record `-1` (or `0xFFFF_FFFF`) as the compressed-size
sentinel for an uncompressed file. Kronos ignores this field when the
compression flag is clear, so it does not verify the sentinel universally.

Use the compression flag to select the stored size and decoding method. File
extensions do not determine compression. The path length includes the
terminating zero byte, and the file offset is absolute from the start of the
archive.

### Checksums

Kronos verifies the checksum over the stored bytes, before decompression. It
uses CRC-32/ISO-HDLC: polynomial `0x04C11DB7` (reflected form `0xEDB88320`),
initial register `0xFFFFFFFF`, reflected input and output, and final XOR
`0xFFFFFFFF`. Python's standard library gives the same result:

```python
import zlib

assert zlib.crc32(b"123456789") == 0xCBF43926
```

The [verification record](../verification.md#archives-and-identifiers) links the
archive reader and checksum evidence.
