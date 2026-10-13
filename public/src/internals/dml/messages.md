# Messages

Messages are the central data structures exposed by a [protocol](./protocols.md). They
define a data layout and are used for runtime serialization and deserialization of data.

A message consists of an XML element with its sole child element being a [record](#records)
holding the data [fields](#fields) along with their [metadata](#metadata).

A DML message may be defined like this:

```xml
<MSG_PING>
  <RECORD>
    <!-- Message metadata -->
    <_MsgName TYPE="STR" NOXFER="TRUE">MSG_PING</_MsgName>
    <_MsgOrder TYPE="UBYT" NOXFER="TRUE">1</_MsgOrder>
    <_MsgDescription TYPE="STR" NOXFER="TRUE">PING request.</_MsgDescription>
    <_MsgHandler TYPE="STR" NOXFER="TRUE">MSG_Ping</_MsgHandler>
    <_MsgAccessLvl TYPE="UBYT" NOXFER="TRUE">0</_MsgAccessLvl>

  </RECORD>
</MSG_PING>
```

## Records

Records only ever occur as XML elements named `RECORD` inside
[protocol information elements](./protocols.md#protocol-information) and messages.

They group various [fields](#fields) together.

## Fields

Fields are used to represent the data layout of a DML message structure or important
metadata for both, [protocols](./protocols.md#protocol-information) and
[messages](#metadata) alike. They only ever occur inside [record](#records) elements.

When used to describe metadata, fields always have a fixed value assigned to them. In
most cases however, their values are undefined and will be filled with runtime object
values.

### Field attributes

Fields may carry specific properties indicated by key-value pairs of XML attributes.
The following presents known attributes, their supported values, and how they influence
the behavior of a field.

#### Types

The most common attribute is `TYPE`. It specifies the data type of [fields](#fields)
using a string acronym:

| Name  | Type     | Description                                                           |
|-------|----------|-----------------------------------------------------------------------|
| BYT   | int8     | Signed 8-bit integer                                                  |
| UBYT  | uint8    | Unsigned 8-bit integer                                                |
| SHRT  | int16    | Signed 16-bit integer in little-endian byte order                     |
| USHRT | uint16   | Unsigned 16-bit integer in little-endian byteorder                    |
| INT   | int32    | Signed 32-bit integer in little-endian byteorder                      |
| UINT  | uint32   | Unsigned 32-bit integer in little-endian byteorder                    |
| STR   | uint8[]  | A length-prefixed string of arbitrary bytes (may be UTF-8)            |
| WSTR  | uint16[] | A length-prefixed string of UTF-16 code units in little-endian order  |
| FLT   | float    | IEEE-754 32-bit floating point number in little-endian byteorder      |
| DBL   | double   | IEEE-754 64-bit floating point number in little-endian byteorder      |
| GID   | uint64   | Unsigned 64-bit integer in little-endian byteorder                    |

#### Visibility

Every field may individually control whether it is visible as a serializable data field
or not. This is accomplished by setting the `NOXFER` attribute to either `"TRUE"` or
`"FALSE"`.

NOXFER stands for "No Transfer" and is mostly used for [metadata fields](#metadata) which
carry fixed values. As per convention, such fields should always start with an underscore
(`_`) in their name to highlight the hidden status.

### Serialization

Serialization and deserialization of a message works by consecutively iterating through the
fields in the order they were listed in the message specification, and encoding the value as
defined by the type. Fields with `NOXFER="TRUE"` are skipped. An absent
attribute or `NOXFER="FALSE"` leaves the field transferable.

`STR` has a little-endian `uint16` byte count followed by that many bytes,
without a null terminator. It may contain binary data, including embedded
zero bytes. `WSTR` has a little-endian `uint16` count of UTF-16 code units,
followed by twice that many bytes in UTF-16LE, without a BOM or terminator.
Supplementary Unicode characters occupy two code units.

The reviewed sources disagree about large string lengths: Kronos uses a
signed 16-bit limit, while kronos-old uses an unsigned prefix. The prefix
width and ordinary positive lengths agree; the upper range needs a client
capture. See [verification gaps](../../verification.md#dml-and-tables).

#### Example

The message

```xml
<MSG_PERSON>
  <RECORD>
    <Name TYPE="STR">Edgar Allan Poe</Name>
    <Age TYPE="UBYT">40</Age>
  </RECORD>
</MSG_PERSON>
```

serializes to

```python
[
    # String length prefix (15)
    0x0f, 0x00,
    # String bytes without null terminator (Edgar Allan Poe)
    0x45, 0x64, 0x67, 0x61, 0x72, 0x20, 0x41, 0x6c, 0x6c, 0x61, 0x6e, 0x20, 0x50, 0x6f, 0x65,
    # Age byte (40)
    0x28
]
```

## Metadata

The records of messages may have a set of hidden [fields](#fields) assigned to them which
solely describe metadata required for runtime handling and querying of individual messages.

Unlike the [protocol information](./protocols.md#protocol-information) however, these fields
must all be marked as [`NOXFER`](#visibility).

Below is a list of fields and their meaning. Some of them are discussed in greater detail
in subsections of this paragraph.

| Name            | Type | Optional | Description                                                |
|-----------------|------|----------|------------------------------------------------------------|
| _MsgName        | STR  | true     | The name of the message; Overrides the XML message tag     |
| _MsgOrder       | UBYT | true     | The [order value](#order-number)                           |
| _MsgDescription | STR  | true     | Short description of the message and its purpose           |
| _MsgHandler     | STR  | varies   | Local handler metadata; not transmitted on the wire       |
| _MsgAccessLvl   | UBYT | true     | The [access level](#access-level) for the message          |

### Order number

`_MsgHandler` names in reference XML may refer to customised server
callbacks. Independent clients and servers do not need those symbols or
the same dispatch mechanism; the transferable field layout and wire
service/order pair determine interoperability.

Every message has its own index assigned to it that is unique within a protocol - the order
value. Order values usually start at `1` and are used in combination with the unique service
ID of the protocol to address a specific message when several protocols are available at once.

As the value is represented as an unsigned 8-bit integer, the amount of messages per protocol
is limited to 255 as a result of that.

Order values may either be explicitly specified using the `_MsgOrder` field or left away for
every message within the protocol. In case of the latter, the implementation automatically
assigns order values in ascending order by sorting message names (name from xml tag; NOT _MsgName) alphabetically. Mixing both
approaches is rejected by the reviewed Kronos parsers. Use either explicit
orders for every message or implicit orders for every message.

The wire identity is the service ID and order number. `_MsgName` is metadata
and can differ from the XML tag; changing it does not change the order of
implicitly numbered messages. Use definitions from the target client
revision, rather than assuming names and orders remain stable.

### Access level

The access level defines the minimum value that must be held by a session in order to be
allowed to process the message.

For the scope of the system, this trait is fairly unimportant, but it plays a great role
in handling [network sessions](../protocol/sessions.md#access-level) and is further explained
there.
