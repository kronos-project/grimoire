# KIWAD Archives

The "KingsIsle Where's All the Data?" format (name guessed) is a custom file format used by the
game to bundle all kinds of game assets in archives. The format presumably draws inspiration from
[the original Doom WAD format](https://doomwiki.org/wiki/WAD#Header) and similarly uses `.wad` as
file extension.

Archives consist of a [header](#header), followed by the [file table](#file-table), and lastly
the encoded file data as described in the table. All data is assumed to be encoded in little-endian
byteorder unless stated otherwise.

## Header

The header may consume 13 or 14 bytes in size and outlines the amount of files to expect in
the archive:

| Offset | Type     | Description                                           |
|--------|----------|-------------------------------------------------------|
| 0x0    | string   | `"KIWAD"` as an ASCII string                          |
| 0x5    | uint32   | The KIWAD version that is used                        |
| 0x9    | uint32   | An integer holding the amount of files in the archive |
| 0xD    | uint8    | optional; [WAD Flags](#flags) if version >= `2`       |

### Flags

On file format versions >= 2, the header encodes a mask of configuration bits which should be
treated as implementation-defined for the most parts; KIWAD implementations can safely ignore
these bit flags and always write `0`.

This is how the official client uses them:

| Bit | Name                       |
|-----|----------------------------|
| 0   | Memory-maps the file view  |
| 1   | Prefetches the file handle |

## File table

This table consists of `n` repetitions of the [file](#file) structure, where `n` is the number
of archived files as encoded in the [header](#header).

## File

This structure denotes the metadata of an encoded file, needed for extraction and validation:

| Offset | Type     | Description                                                |
|--------|----------|------------------------------------------------------------|
| 0x0    | uint32   | The start offset of the file data in the archive           |
| 0x4    | uint32   | The size of the [uncompressed](#compression) file in bytes |
| 0x8    | int32    | The size of the [compressed](#compression) file in bytes   |
| 0xC    | bool     | Whether the encoded file data is compressed                |
| 0xD    | uint32   | The [checksum](#checksums) of the file data                |
| 0x11   | uint32   | The length of the file's path in the archive in bytes      |
| 0x15   | char[]   | The archive file name as a null-terminated string          |

### Compression

If a file is compressed, the corresponding file table entry encodes its uncompressed size
and the compressed size and the data will run through [zlib](https://zlib.net).

If a file is stored uncompressed, `-1` (or `0xFFFF_FFFF`) will be written for its
compressed size.

Notably, only `.mp3` and `.ogg` files are shipped uncompressed in official archives
by KingsIsle.

### Checksums

The checksum of a file is calculated by CRC32 using the polynomial `0x4C11DB7` and
reversed input and output values:

```python
import crcengine

crc32 = crcengine.create(0x04C11DB7, 32, 0, xor_out=0)

assert crc32(b"KIWAD") == 4265429514
```
