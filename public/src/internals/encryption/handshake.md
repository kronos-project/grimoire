# Handshake

## Overview

This chapter describes the AES-GCM session handshake and its message layouts.

Wizard101 uses AES-GCM, an authenticated symmetric cipher, for encrypting
traffic. The AES key and IV are generated clientside and exchanged with the
server by encrypting them using public key cryptography, or more specifically
RSA.

## Exchange

Five RSA public keys are embedded in each `WizardGraphicalClient.exe` binary.
These are rotated semi-regularly, usually when a new version is released. When a
handshake is initiated, the server indicates one of the keys to use at random
and the client uses it to wrap an AES key and IV used for subsequent immediate
communication, similar in principle to TLS.

The embedded keys have multiple layers of obfuscation which can make them tricky
to locate. This is described in more detail
[here](./obfuscation.md#pubkey-blob).

After the handshake, a new IV is sent periodically, and the GCM context is
reinitialised (reset.) Every 4096 (0x1000) blocks of clientbound traffic, the
server transmits a 16 byte GCM authentication tag for the preceding traffic, and
16 byte IV for the client to use for decrypting traffic that follows. Similarly,
every 256 (0x100) blocks of serverbound traffic, the client transmits an
authentication tag and new IV. The key remains static for the duration of the
session. Note that an AES block is 16 bytes.

<div class="warning">
This usage of AES-GCM is insecure and is not supported by all cryptography libraries. Data decrypted with AES-GCM should only be used if it can be authenticated. However, Wizard101 ignores this and uses the data anyway, only authenticating it periodically when a tag is received.

Most cryptography libraries mandate that an authentication tag follows the
ciphertext for succcessful decryption and do not support encryption without
immediate tagging. Wizard101 uses Crypto++ which appears to have neither of
these requirements.

</div>

## Handshake

The handshake begins with a SessionOffer packet from the server. The signed
message is followed by 256 bytes of an RSA signature.

Layouts use the book's [binary notation](../../notation.md). The following outer
layout matches [Session Offer](../protocol/control.md#session-offer). Offsets
start at its control payload; let `L` be `signed_message_length`.

| Offset     | Type    | Field                 | Description                                     |
| ---------- | ------- | --------------------- | ----------------------------------------------- |
| `0x00`     | `u16`   | session_id            | Proposed session ID                             |
| `0x02`     | `i32`   | timestamp_high        | High 32 bits of epoch seconds                   |
| `0x06`     | `i32`   | timestamp_low         | Low 32 bits of epoch seconds                    |
| `0x0A`     | `u32`   | timestamp_millis      | Milliseconds into the second                    |
| `0x0E`     | `u32`   | signed_message_length | Signed blob byte count, including its signature |
| `0x12`     | `u8[L]` | signed_message        | Signed message and RSA signature                |
| `0x12 + L` | `u8`    | terminator            | Always `0`; outside the signed blob             |

The signed blob has the following layout, with offsets relative to the start of
the length-prefixed blob:

Let `C` be `challenge_length`, including the FNV fields and challenge-type byte.
The RSA signature is 256 bytes for a 2048-bit key.

| Offset     | Type        | Field            | Description                                       |
| ---------- | ----------- | ---------------- | ------------------------------------------------- |
| `0x00`     | `u8`        | flags            | Session flags                                     |
| `0x01`     | `u8`        | key_slot         | RSA key selected by the server                    |
| `0x02`     | `u8`        | key_mask         | XOR mask for decoding the RSA public key          |
| `0x03`     | `u8`        | challenge_length | Byte count of the region beginning at `0x04`      |
| `0x04`     | `u16`       | fnv_offset       | Offset into the raw keyring blob                  |
| `0x06`     | `u16`       | fnv_length       | Keyring byte count to hash                        |
| `0x08`     | `u8`        | challenge_type   | Challenge algorithm selector                      |
| `0x09`     | `u8[C - 5]` | challenge_data   | Challenge-specific data                           |
| `0x04 + C` | `u32`       | nonce            | Value echoed back to the server                   |
| `0x08 + C` | `u8[256]`   | signature        | RSA signature of all preceding bytes in this blob |

The nonce is at `0x04 + C`. After it come 256 signature bytes. The signature
covers the signed portion from `flags` through `nonce`, excluding the signature
itself, length prefix, and outer session fields. It uses RSA PKCS#1 v1.5 and
SHA-1. The key-slot selector addresses one of five keys, indexed `0` through
`4`.

Challenge type `0xF1` selects the ClientSig integrity challenge. Its response is
a `u32`.

The client will then decode the key slot selected by the server and use the
resulting public key to verify the signature. It will also compute the challenge
and assemble a SessionAccept packet to return to the server:

Offsets start at the control payload. Let `L` be `encrypted_message_length`;
this is the [Session Accept layout](../protocol/control.md#session-accept) with
the encrypted blob's algorithm and ciphertext shown separately.

| Offset     | Type        | Field                    | Description                                   |
| ---------- | ----------- | ------------------------ | --------------------------------------------- |
| `0x00`     | `u16`       | reserved                 | Always `0`                                    |
| `0x02`     | `i32`       | timestamp_high           | High 32 bits of epoch seconds                 |
| `0x06`     | `i32`       | timestamp_low            | Low 32 bits of epoch seconds                  |
| `0x0A`     | `u32`       | timestamp_millis         | Milliseconds into the second                  |
| `0x0E`     | `u16`       | session_id               | Proposed session ID from the offer            |
| `0x10`     | `u32`       | encrypted_message_length | Blob byte count, including the algorithm byte |
| `0x14`     | `u8`        | algorithm                | Asymmetric encryption selector                |
| `0x15`     | `u8[L - 1]` | ciphertext               | RSA-encrypted confirmation                    |
| `0x14 + L` | `u8`        | terminator               | Always `0`; outside the encrypted blob        |

| Algorithm            | Value |
| -------------------- | ----- |
| RSA/OAEP-MGF1(SHA-1) | 0x1   |

RSA encryption uses OAEP with SHA-1, MGF1(SHA-1), and an empty label. The length
includes the algorithm byte and RSA ciphertext. With a 2048-bit key it is `257`,
followed by the separate trailing zero.

### Encrypted confirmation plaintext

The decrypted confirmation plaintext has this layout. All integers are
little-endian.

Offsets start at the decrypted confirmation, not at its RSA ciphertext.

| Offset | Type     | Field              | Description                                  |
| ------ | -------- | ------------------ | -------------------------------------------- |
| `0x00` | `u8`     | flags              | Session flags                                |
| `0x01` | `u32`    | fnv_result         | FNV-1a-32 hash of the offered keyring region |
| `0x05` | `u32`    | challenge_response | ClientSig response for challenge type `0xF1` |
| `0x09` | `u32`    | nonce              | Echo of the offer nonce                      |
| `0x0D` | `u32`    | timestamp_secs     | Client epoch seconds                         |
| `0x11` | `u8[16]` | aes_key            | AES-128 key                                  |
| `0x21` | `u8[16]` | iv                 | Initial GCM IV                               |

The confirmation plaintext is 49 bytes. `fnv_result` is the standard FNV-1a-32
hash of the raw blob region, rather than the decoded ASN.1 key.

`0xF1` is the only challenge type observed in practice, but the system allows
further challenge types to be added in the future.

The `0xF1` challenge contains three little-endian `u32` values after its type
byte. The first is a set of buffer flags: bits 5, 9, 12, and 17 control the
disabled, offsets, modules, and XOR-encoding options. The second selects the
hash operation and its parameters. Its low two bits select one of four per-byte
hash functions, bits 2–5 set the chunk size plus one, bits 6–7 set the repeat
count plus one, and the high bits provide the initial hash value. The third
value is the key used to scramble the selected buffers. The challenge handler
assembles the selected data, applies the requested hash operation, and returns
the resulting `u32` in `challenge_response`.

ClientSig is client-integrity data used by this challenge. It contains code
offsets, hashes of client dependency modules, and code bytes copied from the
client at those offsets. The challenge selects and combines these buffers before
hashing them; the returned value lets the server check the client's integrity
response.

## ClientSig file format

`ClientSig.bin` consists of multiple length-prefixed chunks. Each chunk starts
with a little-endian `u32` ciphertext length, followed by an RSA-OAEP ciphertext
of that length. Every chunk must be decrypted separately with the corresponding
RSA private key, using OAEP with SHA-1 and MGF1-SHA-1. Concatenate the decrypted
chunks in order; the result is the ClientSig data:

| Field         | Type    | Contents                              |
| ------------- | ------- | ------------------------------------- |
| `offsets_len` | `u32`   | Number of offset bytes                |
| `offsets`     | `u8[N]` | Offsets of copied code                |
| `modules_len` | `u32`   | Number of module-hash bytes           |
| `modules`     | `u8[N]` | Module hashes for client dependencies |
| `code_len`    | `u32`   | Number of code bytes                  |
| `code`        | `u8[N]` | Code fragments copied from offsets    |

### Encrypted stream boundaries

The offer and accept are exchanged as plaintext frames, except for the encrypted
confirmation payload inside the accept. After the accept, the client and server
use AES-GCM for the entire subsequent framed byte stream, including frame
headers. There are independent cipher states and byte counts in each direction,
initially using the exchanged key and IV.

The periodic boundaries are `65536` ciphertext bytes towards the client and
`4096` towards the server. At each boundary, a 16-byte tag is followed by a
16-byte replacement IV. These 32 bytes belong to the encryption layer, have no
`0xF00D` header, and do not count towards the next ciphertext interval. A
boundary may interrupt a frame or fall between frames; remove the tag/IV bytes
before passing decrypted bytes to the frame parser.

This use of AES-GCM is flawed: plaintext is released before it can be
authenticated. Each tag covers an entire interval, but the receiver decrypts and
acts on the data as it arrives, and only learns whether it was tampered with
when the tag arrives, up to `65536` or `4096` bytes later. GCM is a stream
cipher mode, so an attacker can flip chosen ciphertext bits and flip the same
plaintext bits, with the change only detected at the next boundary, if at all.
Most libraries refuse to expose unauthenticated plaintext, whereas Crypto++
permits it.

Offsets in the boundary record start immediately after the preceding ciphertext
interval:

| Offset | Type     | Field | Description                                   |
| ------ | -------- | ----- | --------------------------------------------- |
| `0x00` | `u8[16]` | tag   | Authentication tag for the preceding interval |
| `0x10` | `u8[16]` | iv    | IV for the following interval                 |
