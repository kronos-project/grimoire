# Serialization

This aims to give insight into the serialization of Property Classes.

The following uses Python pseudocode for illustration which makes use of format strings
defined by the [struct module](https://docs.python.org/3/library/struct.html).

These examples illustrate the format with abstract byte/bit buffers.
Conflicting size, alignment, and filtering rules are recorded in
[verification gaps](../../verification.md#objectproperty-and-coreobject).

## Binary

The binary serialization mode is commonly encountered for both local game files and also
for `STR` types in many DML messages transferred over the network.

### Buffering

For writing the binary data, a sink with bit-oriented instead of byte-oriented buffering
is preferred due to some types being serialized in units of single bits only.

In such cases, the buffer should progress sequentially from the LSB of a byte to its MSB
before advancing to the next one.

When the binary size of a type is in units of whole bytes, the buffer will be aligned to the
start of a full byte with bit position 0, if not already there, before writing said type.

### Flags

Binary serializers and deserializers may have a set of flags attached to them to customize
their behavior:

| Bit | Purpose                                                                           |
|-----|-----------------------------------------------------------------------------------|
| 0   | Indicates that these flags should be serialized and re-used by the deserializer   |
| 1   | Tries to pack length prefixes into smaller quantities for compact serialization   |
| 2   | Causes enum variants to be serialized as human-readable strings instead of values |
| 3   | Enables [zlib](https://zlib.net) compression of serialized object state           |
| 4   | Properties with flag bit `8` set must always be dirty when serialized             |

### Header

A serialized stream starts with the necessary header data followed by the compressed or
uncompressed object bytes:

```py
output = bytearray()

# Serialize our flags value if `STATEFUL_FLAGS` (bit 0) is set.
if serializer_flags & STATEFUL_FLAGS != 0:
    output.extend(serializer_flags.to_bytes(4, "little"))

# Handle compression if `WITH_COMPRESSION` (bit 3) is set.
if serializer_flags & WITH_COMPRESSION != 0:
    compressed_object_data = zlib.compress(object_data)

    if len(compressed_object_data) < len(object_data):
        uncompressed_size = len(object_data)
        object_data = compressed_object_data

        # Indicate that the data is compressed.
        output.append(1)
        # Write the size of the uncompressed object for the deserializer to validate.
        output.extend(uncompressed_size.to_bytes(4, "little"))
    else:
        # Indicate that the data is uncompressed.
        output.append(0)

# Write either the compressed or uncompressed data.
output.extend(object_data)
```

### Objects and Properties

The serialization system deals with whole `PropertyClass`es at any time. No loose values
anywhere.

A serializer accepts the following inputs to customize its behavior:

- a [mask of serializer flags](#flags) for configuration

- a boolean denoting whether the output will be *shallow* or *deep*

- a property mask selecting properties that contain every bit in the mask:
  `(property.flags & mask) == mask`

#### Data model

The *ObjectProperty* data model defines what types are supported and how they are serialized.
This may be freely extended with custom types that are implementation-defined and not
`PropertyClass`es themselves.

The following examples of serialization modes will use an imaginary `serialize_value` function
that should be thought of as a mapping arbitrary values into this data model and serializing
them to the `buffer` argument.

Be sure to consider the buffering remarks at the start when implementing this.

- booleans will be written as a single bit; `1` for `true` and `0` for `false`

- primitive integer types (signed and unsigned) will be written as bytes in little-endian order

- floating-point numbers according to IEEE-754 are bit-copied into `uint32_t`/`uint64_t` and
serialized as such

- strings are serialized as UTF-8 bytes with their **length prefixed**

- wide strings are serialized as UTF-16 code points in little-endian order without BOM and with
their **length prefixed** in UTF-16 code units, not bytes or Unicode scalar values

- collections, such as lists or vectors, are serialized as a sequence of element values with
their **length prefixed**

- tuples or arrays with a known length are serialized as just the sequence of elements

- when a property is **optional** (i.e. has bit 8 set in its flags set), its value may be
skipped (unless the serializer has bit 4 set in its flags); a single bit of `0` denotes that no
value is given, otherwise a value of `1` followed by the property's value is written

- enum variants are either serialized as their integral value or, when serializer flag bit 2 is
set, as a string representation of the variant name
  - an empty string for bit enums is equivalent to a value of `0`
  - the bit enum variant string is a list of flag names: `A|B|C`

- **length prefixes** are `uint16_t` for (w)strings and `uint32_t` for collections unless serializer
bit 1 is set, which enables a common compression algorithm applied to both types - when the length
is smaller than `0x80`, write one zero discriminator bit followed by seven
length bits; otherwise write one discriminator bit followed by 31 length
bits. Equivalently, when byte-aligned, the encoded integer is `length << 1`
for the short form and `(length << 1) | 1` for the long form. For example,
length `3` encodes as `06`, `127` as `FE`, and `128` as `01 01 00 00`.
The maximum compact length is `0x7FFFFFFF`. The payload following the prefix
is byte-aligned. Alignment before compact prefixes differs between the
reviewed implementations and needs client verification.

#### Type Tag

Every serialized `PropertyClass` state has a type tag associated with it to uniquely identify it
during deserialization.

The type tag is a [**string ID**](../string-id.md) of the type's name.

A zero type tag represents a null object reference and has no property
payload. Class names and property order must come from the target client's
type definitions; shallow mode cannot reconstruct missing type metadata.

#### Property Tag

Property tags uniquely identify a property within an object in *deep* serialization mode.

The tag is a sum of the property type's [**string ID**](../string-id.md) and a slightly modified
[djb2 hash](https://theartincode.stanis.me/008-djb2/) of the property's name with the MSB value discarded.

Practically speaking:

```py
type_tag = string_id(property.type_name)  # NOT object.type_name
name_hash = djb2(property.name) & 0x7FFF_FFFF

property_tag = (type_tag + name_hash) & 0xFFFF_FFFF
```

#### Shallow mode

In shallow mode, the 32-bit [object type tag](#type-tag) is written followed by a sequence of masked
property values in their correct order:

```py
buffer = BinaryBuffer()

buffer.write("<I", object.type_hash)
for property in filter(lambda p: p.flags & mask == mask, object.properties):
    serialize_value(buffer, property.value)
```

The original notes require deprecated properties to retain their slots in
shallow mode, but Kronos's generated shallow serializer skips them. The
client's behaviour is unresolved. Do not assume either rule is universally
correct without a matching type dump and capture.

#### Deep mode

In deep mode, the concept is a bit different. Here, the 32-bit [object type tag](#type-tag)
is serialized, followed by a mapping of [property tags](#property-tag) to their values.
Additionally, size information in bits is written for integrity validation.

In practice, this looks like this:

```py
buffer = BitBuffer()

buffer.write("<I", object.type_hash)

# Reserve a placeholder for the object size.
object_size_position = len(buffer)
buffer.write("<I", 0)

# Here we don't only skip unmasked properties, but also deprecated ones.
for property in filter(
    lambda p: p.flags & mask == mask and p.flags & FLAG_DEPRECATED == 0, object.properties
):
    # Reserve a placeholder for the property size.
    property_size_position = len(buffer)
    buffer.write("<I", 0)  # Will be replaced by a real size later.

    # Write the mapping of property hash to value.
    buffer.write("<I", property.hash)
    serialize_value(buffer, property.value)

    # Patch back the real property size.
    property_end = len(buffer)
    buffer.seek_bit(property_size_position)
    buffer.write("<I", property_end - property_size_position)
    buffer.seek_bit(property_end)

# Patch back the real object size.
object_end = len(buffer)
buffer.seek_bit(object_size_position)
buffer.write("<I", object_end - object_size_position)
buffer.seek_bit(object_end)
```

Here buffer lengths and positions are in bits. Property sizes include their
32-bit size field, 32-bit tag, value, and any alignment padding counted from
the size field's start. The illustrated object size includes its own
32-bit size field but excludes the preceding type tag. Kronos's reader
uses that convention; kronos-old's reader disagrees with its own writer
about those 32 bits. This is a tracked interoperability gap, not proof of
two supported wire variants.

The order of property entries, while usually maintained, is not as important as it is for
**shallow** serialization.

## Files

When serializing to files, a common convention is to use an `.xml` suffix. This orignates
from different ways of representing the serialized data inside them.

For debugging purposes, a human-readable format is often desired. It is very straightforward
and can be fully explained in a short example:

```xml
<Objects>
  <Class Name="class Example">
    <!-- We place a tag for every property and its value as the tag's content. -->
    <m_someString>Test</m_someString>
    <m_someInt>1337</m_someInt>
    <m_someObject>
      <Class Name="class SomeObject">
        <m_test>Properties holding objects will hold a nested Class element</m_test>
      </Class>
    </m_someObject>
    <m_someTuple>1,0,0,1</m_someTuple>

    <!-- This is how we serialize properties holding container values. -->
    <m_listOfStrings>A</m_listOfStrings> <!-- Index 0 -->
    <m_listOfStrings>B</m_listOfStrings> <!-- Index 1 -->
    <m_listOfStrings>C</m_listOfStrings> <!-- Index 2 -->
  </Class>
</Objects>
```

When distributing game data, specifically data that is not meant to be edited afterwards,
a more compact format is often preferred. This is
[exhaustive binary serialization](#objects-and-properties) with a special file magic:

```py
FILE_MAGIC = 0x644E4942  # b"BINd" in little-endian byteorder

buffer.write("<I", FILE_MAGIC)
buffer.extend(serialized_object_state)
```
