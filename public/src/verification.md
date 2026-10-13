# Verification and Gaps

This record cross-references the book with the Kronos organisation's source
repositories as reviewed on 13 October 2026. Its purpose is to support an
independent client or server: packet layouts, data encodings, file formats, and
externally observable exchanges are in scope. Database design, actor systems,
code generation APIs, proxy architecture, and tool capture formats are outside
the game specification.

## Evidence and revisions

The priority sources are pinned below. These are repository revisions, not game
client revisions. Agreement establishes what the reference code encodes or
decodes; a versioned client capture is stronger evidence of actual game
behaviour. Some links require access to private repositories.

| Repository                     | Revision       | Relevant coverage                                        |
| ------------------------------ | -------------- | -------------------------------------------------------- |
| [kronos][kronos]               | `daa767fa78aa` | DML, ObjectProperty, CoreObject, KIWAD, hashes           |
| [kronos-old][old]              | `10f7db402a93` | Plaintext transport, login schemas, CK1/REC1, characters |
| [moonlight][moonlight]         | `f14e3024678d` | Capture decoding, control blobs, DML metadata            |
| [merlin][merlin]               | `43222053581c` | Encrypted handshake fixtures and transport               |
| [login][login]                 | `059761c860ab` | CK1 and REC1 test vectors                                |
| [imhex-patterns][patterns]     | `1b40bacada23` | Session and key-blob layouts                             |
| [reverse-engineering][reverse] | `722efc2ba996` | Runtime key extraction; limited format evidence          |
| [netpack][netpack]             | `ecdfe34b35ac` | Reviewed to distinguish tool records from game packets   |

The organisation repository listing was also checked. `traffic-analysis`
contains revision-labelled capture archives; their contents have not been
decoded in this review. `netpack101` was inspected by file listing only. Other
repositories were not used as protocol evidence. The book is not a fully
verified specification of the current live game.

## Transport

Corroborated: `0xF00D` framing, four bytes of body metadata, service/order
addressing, and the DML length including its four-byte header but excluding its
trailing zero. [kronos-old's encoder][encoder] establishes the inner layout;
[Merlin's frame reader/writer][frames] establishes body lengths and
four/eight-byte outer headers. The book now gives a complete empty ping frame
and distinguishes TCP reads from frame boundaries.

Conflicting: kronos-old chooses the large header from DML payload size; Merlin
chooses it from body size. A capture at body lengths `32767` and `32768` would
settle the boundary. The latter avoids an ambiguous small length whose high bit
selects the extended header.

Conflicting: the old plaintext session codec omits blob length prefixes;
[Moonlight][control] and [Merlin's session fixtures][session-tests] use
length-prefixed signed/encrypted blobs. The book distinguishes these layouts
instead of treating an absent blob as an empty extended blob.

Missing, blocking broad interoperability:

- Client revisions selecting old or extended session layouts, including the
  exact no-encryption offer/accept bytes.
- UDP framing, reliability, and session lifecycle.
- Meaning and payloads of control opcodes `0x1` and `0x2`.
- Session invalidation, retries, ID uniqueness scope, and error paths.
- Capture-confirmed heartbeat time units, scheduling, and disconnect thresholds.
  The [old frame helpers][old-frames] specify 10/60-second intervals and
  response echoes; this does not prove current KI policy.
- A mechanism, if any, for DML payloads beyond the inner 16-bit size.

## DML and tables

Corroborated: field order follows XML; only `NOXFER="TRUE"` fields are excluded;
explicit `_MsgOrder` values or alphabetical XML-tag order identify messages.
[Kronos's field parser][fields] and [order assignment][orders] agree with the
old parser. `_MsgName` may differ from the XML tag and does not change implicit
ordering.

The book fixes `_MsgType` to `_MsgOrder`, includes the `SHRT` supported by
Moonlight and the old scalar codec, and specifies byte/code-unit string counts.
[Old DML I/O][dml-io] reads `WSTR` code units; its size calculation omits the
factor of two. Moonlight's WSTR reader consumes a byte count instead. These are
source disagreements, not reasons to invent another string type. [Kronos's DML
I/O][dml-new] also uses a signed string limit.

Missing:

- A WSTR capture containing a supplementary Unicode character, and STR captures
  spanning the signed/unsigned 16-bit boundary.
- Client semantics of `_MsgHandler`, metadata defaults, and protocol version
  negotiation. Handler names in the supplied login XML are customised server
  callbacks, not required game symbols.
- Complete `LatestFileList.bin` decoding: dictionary termination/counts, flags,
  `_TargetTable` placement, template association, and all tag mappings. The [old
  table writer][tables] confirms entry headers and counts but does not establish
  a complete dictionary reader. The book's tag table and detailed templates
  remain original research notes.

## ObjectProperty and CoreObject

Corroborated: LSB-first bits, little-endian scalar values, type/property hashes,
all-bits property mask selection, and compact length encoding. [Kronos's bit
helpers][lengths] and the [old serializer][serializer] support a discriminator
followed by 7 or 31 length bits. The compression example now preserves the
uncompressed size before replacing the buffer. The deep example restores the
write position after patching lengths.

Conflicting or incomplete:

- The original shallow-mode notes retain deprecated properties; [Kronos's
  generated serializer][reflect] skips them. Resolve using a client type dump
  and capture containing such a property.
- The deep object-size convention includes its own size field in the writers and
  Kronos reader, but the [old reader][deserializer] compares it against only the
  following property bits. Property alignment at unaligned size fields also
  needs a capture with bit-sized values.
- Compact prefix alignment differs between the old and new helpers.
- [Kronos's compression code][compression] writes a different size than its
  decompressor expects, and its header handling is incomplete. The old
  serializer/deserializer agree on an uncompressed-size prefix. Do not use the
  newer implementation's discrepancy as a wire variant.
- Deep-mode property filtering, inherited-property grouping, optional presence
  bits, and stateful flag handling need shared client fixtures.

Missing, blocking game-world support:

- Versioned type dumps and the mask, flags, shallow/deep mode, and any outer
  compression for each binary DML field. DML XML alone cannot supply these
  settings; Moonlight requires additional metadata.
- A complete CoreObject chapter. Both sources have a distinct identity envelope:
  a type byte, marker byte, and 32-bit template ID; ordinary PropertyClass
  identities have a zero prefix before their type hash in that context.
  [Kronos's CoreObject serializer][core] and the old one are evidence, but
  marker reconstruction and template validation need client confirmation.
- Template/manifest and behaviour payload formats, custom primitive types,
  unknown-property handling, and null references in deep mode.

## Authentication and characters

Corroborated: CK1 hashes the base64 password hash and decimal session salt; REC1
uses the documented key/IV byte placement and Twofish OFB. [The old
authentication helper][auth-helper] agrees with the independent [login CK1
vector][ck1]. Although the older [REC1 code][rec1] calls its mode CTR, it
restarts each block with the previous keystream as its nonce; this produces the
OFB recurrence rather than an incrementing CTR stream. The book now
distinguishes string inputs, bytes, and standard padded base64, and fixes the Go
record helper to encrypt a byte slice.

[LoginMessages.xml][login-schema] establishes field order, `INT` PayingUser,
character message aliases/orders, and AFK fields. The book now lists those
aliases, the shallow `0x18` character mask, and the `TooManyChars` and
`DeletionFailed` error strings used by the handlers. These error mappings are
implementation evidence, pending client captures.

Missing, blocking a complete login-to-game flow:

- CK2 response plaintext and encoding, generation, lifetime, and renewal. [The
  old handler][auth-handler] emits a placeholder CK2 and leaves PK3 checking
  unfinished. PK3's helper hashes CK2 text, but no completed validation exchange
  was established.
- Meaning of authentication/validation `Flags` and `TimeStamp`; complete
  `Error`/`Reason` combinations; required client-version and identity checks.
- Busy-queue and character `PrepPhase` captures. Their extra values are original
  book notes; the old handler only confirms admission status `1`.
- `MSG_SELECTCHARACTER` / `MSG_CHARACTERSELECTED` definitions and orders,
  connection closure timing, game-server admission/selection-key check, and the
  initial tutorial zone/position. The reviewed login schema does not include
  these messages.
- Name-index component order, revision-specific character defaults/slot limits,
  AFK warning values, and exact UI activity triggers. The original
  character-creation error `1745079928` is still unmapped; it is not the string
  ID of `TooManyChars` (`255337254`).
- Historical v1/v2 authentication support. Ignoring them is a reference server
  policy, not proof that every KI server ignores them.

## Encryption

[Merlin's session parser and tests][session-tests] add challenge-region fields
and an AES-128 confirmation fixture. [Its keyring operations][rsa] establish
PKCS#1 v1.5/SHA-1 signatures, OAEP/SHA-1 with an empty label, and FNV-1a over a
raw keyring region. [The proxy stream settings][streams] agree with the existing
4096/65536-byte directional boundaries. The book now separates these encryption
records from framed messages.

Missing, blocking independent encrypted handshakes:

- Challenge algorithms, challenge-type/flag meanings, variable response lengths,
  and complete known-answer fixtures for every supported type.
- Independent verification of GCM tags and arbitrary split/boundary handling.
  [Merlin's cipher stream][crypt] skips received tag validation; its GCM
  encryption hashes plaintext while decryption hashes ciphertext. Its output is
  insufficient evidence for a correct tag implementation.
- Current key rotation, encryption opt-out support, downgrade behaviour, and
  encryption reset when a session is refreshed.
- Symmetric modes, padding, length-field meaning, and runtime obfuscation of
  public-key blobs. [The key-blob pattern][keyblob] supports algorithm IDs and
  key/IV sizes, but not a complete decoding algorithm. The book's
  runtime-obfuscation statement still has no documented transformation.

## Archives and identifiers

[Kronos's archive loader/checksum verifier][wad] and [file entries][wad-file]
agree with the book's header and file table. The corrected checksum uses
CRC-32/ISO-HDLC on stored bytes. Path lengths include the terminator;
compression is selected by the entry flag, not the filename extension.

Missing: archive fixtures for each version, flag-bit behaviour in the official
client, and confirmation of the `-1` compressed-size sentinel for uncompressed
files. The loader ignores that size when uncompressed, which does not prove all
writers produce the sentinel.

[Kronos's hashes][hash-new] and [old hashes][hash-old] agree for ASCII string
IDs and property-name hashing. UTF-16 versus UTF-8 input iteration differs for
non-ASCII names. Record client vectors before declaring either representation
universal. GIDs remain unsigned 64-bit values; kronos-old's epoch/worker-based
generator is server-specific and was not added as a game ID requirement.

The graphical client argument list has no complete parser or help-output fixture
in the reviewed sources. Argument spellings, `-U` tokenisation, and
required/optional arguments need validation against a named client revision.
Existing spellings are retained as original notes.

## Closing a gap

For each unresolved item, retain the game client revision, XML/type dump
revision, relevant raw bytes, direction, session state, and expected decode or
encode result. A source conflict stays unresolved until that evidence
distinguishes client behaviour from a reference implementation bug or local
policy. Update the affected chapter and this record together.

[kronos]:
  https://github.com/kronos-project/kronos/tree/daa767fa78aaf6c85f9f7cb3f39db09308c738e3
[old]:
  https://github.com/kronos-project/kronos-old/tree/10f7db402a93c0bf0fa05822ed6b245dafcbb22c
[moonlight]:
  https://github.com/kronos-project/moonlight/tree/f14e3024678d8bef72a2595af10de966f8c20177
[merlin]:
  https://github.com/kronos-project/merlin/tree/43222053581c66384e1f478849416a2935b2e255
[login]:
  https://github.com/kronos-project/login/tree/059761c860ab4247ec2d265c64dd1380913998e8
[patterns]:
  https://github.com/kronos-project/imhex-patterns/tree/1b40bacada23564c4baf54588a72b0e21e5704df
[reverse]:
  https://github.com/kronos-project/reverse-engineering/tree/722efc2ba996323123005c1d711610cbdde4c278
[netpack]:
  https://github.com/kronos-project/netpack/tree/ecdfe34b35acd0dedbf3249a11e2e2783445faa9
[encoder]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/net/src/codec/encoder.rs
[frames]:
  https://github.com/kronos-project/merlin/blob/43222053581c66384e1f478849416a2935b2e255/kiproto/frame.go
[control]:
  https://github.com/kronos-project/moonlight/blob/f14e3024678d8bef72a2595af10de966f8c20177/moonlight/net/control.py
[session-tests]:
  https://github.com/kronos-project/merlin/blob/43222053581c66384e1f478849416a2935b2e255/kiproto/messages_test.go
[old-frames]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/net/src/frame.rs
[fields]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.Dml.Generator/Specification/Field.cs
[orders]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.Dml.Generator/Specification/Protocol.cs
[dml-io]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/dml/src/io.rs
[dml-new]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.Dml/DmlEncodingExt.cs
[tables]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/dml/src/table.rs
[lengths]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.ObjectProperty/Serialization/BitWriterExt.cs
[serializer]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/serialize/src/binary/ser.rs
[reflect]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.ObjectProperty.Generator/CodeGen/ReflectEmitter.cs
[deserializer]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/serialize/src/binary/de.rs
[compression]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.ObjectProperty/Serialization/AbstractBinarySerializer.cs
[core]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.CoreObject/Serialization/CoreObjectSerializer.cs
[auth-helper]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/src/core/login/auth.rs
[ck1]:
  https://github.com/kronos-project/login/blob/059761c860ab4247ec2d265c64dd1380913998e8/src/client_key.rs
[rec1]:
  https://github.com/kronos-project/login/blob/059761c860ab4247ec2d265c64dd1380913998e8/src/rec1.rs
[login-schema]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/messages/login/LoginMessages.xml
[auth-handler]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/src/login/auth.rs
[rsa]:
  https://github.com/kronos-project/merlin/blob/43222053581c66384e1f478849416a2935b2e255/kicrypto/keyring.go
[streams]:
  https://github.com/kronos-project/merlin/blob/43222053581c66384e1f478849416a2935b2e255/merlin/merlin.go
[crypt]:
  https://github.com/kronos-project/merlin/blob/43222053581c66384e1f478849416a2935b2e255/kicrypto/crypt.go
[keyblob]:
  https://github.com/kronos-project/imhex-patterns/blob/1b40bacada23564c4baf54588a72b0e21e5704df/patterns/keyblob.hexpat
[wad]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.Wad/WadArchive.cs
[wad-file]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.Wad/ArchiveFile.cs
[hash-new]:
  https://github.com/kronos-project/kronos/blob/daa767fa78aaf6c85f9f7cb3f39db09308c738e3/src/Kronos.Utils/HashExt.cs
[hash-old]:
  https://github.com/kronos-project/kronos-old/blob/10f7db402a93c0bf0fa05822ed6b245dafcbb22c/crates/utils/src/hash.rs
