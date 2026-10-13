# Handshake

## Overview

The detailed additions below are tied to Merlin's reviewed 2023 fixtures. The
current live game's key selection, rotation, and negotiation rules have not been
independently verified; see the
[evidence record](../../verification.md#encryption).

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
the length-prefixed blob. Merlin gives more detail inside the challenge region
than the original notes:

Let `C` be `challenge_length`, including the FNV fields and challenge-type byte.
The signature length shown applies to the reviewed 2048-bit RSA key.

| Offset     | Type        | Field            | Description                                       |
| ---------- | ----------- | ---------------- | ------------------------------------------------- |
| `0x00`     | `u8`        | flags            | Meanings unverified                               |
| `0x01`     | `u8`        | key_slot         | RSA key selected by the server                    |
| `0x02`     | `u8`        | key_mask         | XOR mask for decoding the RSA public key          |
| `0x03`     | `u8`        | challenge_length | Byte count of the region beginning at `0x04`      |
| `0x04`     | `u16`       | fnv_offset       | Offset into the raw keyring blob                  |
| `0x06`     | `u16`       | fnv_length       | Keyring byte count to hash                        |
| `0x08`     | `u8`        | challenge_type   | Challenge algorithm selector                      |
| `0x09`     | `u8[C - 5]` | challenge        | Challenge bytes                                   |
| `0x04 + C` | `u32`       | nonce            | Value echoed back to the server                   |
| `0x08 + C` | `u8[256]`   | signature        | RSA signature of all preceding bytes in this blob |

The nonce is at `0x04 + C`. After it come 256 signature bytes in the reviewed
capture. The signature covers the signed portion from `flags` through `nonce`,
excluding the signature itself, length prefix, and outer session fields. Merlin
verifies it with RSA PKCS#1 v1.5 and SHA-1. The key-slot selector addresses one
of five keys, indexed `0` through `4`.

Merlin's offer fixture has `challenge_length = 17`, `fnv_offset = 1007`,
`fnv_length = 473`, and a nonce at offset `0x15`. It supports the region layout
above. The flag bits and challenge algorithm are still unknown; the reviewed
proxy forwards the client's challenge response.

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

Merlin's reviewed confirmation fixture establishes this layout, before RSA
encryption. All integers are little-endian.

Offsets start at the decrypted confirmation, not at its RSA ciphertext.

| Offset | Type     | Field              | Description                                  |
| ------ | -------- | ------------------ | -------------------------------------------- |
| `0x00` | `u8`     | flags              | Meanings unverified                          |
| `0x01` | `u32`    | fnv_result         | FNV-1a-32 hash of the offered keyring region |
| `0x05` | `u8[4]`  | challenge_response | Response in the reviewed fixture             |
| `0x09` | `u32`    | nonce              | Echo of the offer nonce                      |
| `0x0D` | `u32`    | timestamp_secs     | Client epoch seconds                         |
| `0x11` | `u8[16]` | aes_key            | AES-128 key                                  |
| `0x21` | `u8[16]` | iv                 | Initial GCM IV                               |

The fixture is 49 bytes. Four-byte challenge responses are an assumption in
Merlin's parser, not an established rule for every challenge type. FNV-1a-32
starts at `0x811C9DC5`; for each selected raw keyring byte, XOR the byte and
multiply by `0x01000193`, wrapping to 32 bits. Hash the raw blob region, rather
than the decoded ASN.1 key.

### Encrypted stream boundaries

The offer and accept are exchanged as plaintext frames. Merlin switches to
AES-GCM after the accept, wrapping the entire subsequent framed byte stream,
including frame headers. There are independent cipher states and byte counts in
each direction, initially using the exchanged key and IV.

The periodic boundaries are `65536` ciphertext bytes towards the client and
`4096` towards the server. At each boundary, a 16-byte tag is followed by a
16-byte replacement IV. These 32 bytes belong to the encryption layer, have no
`0xF00D` header, and do not count towards the next ciphertext interval. A
boundary may interrupt a frame or fall between frames; remove the tag/IV bytes
before passing decrypted bytes to the frame parser.

Offsets in the boundary record start immediately after the preceding ciphertext
interval:

| Offset | Type     | Field | Description                                   |
| ------ | -------- | ----- | --------------------------------------------- |
| `0x00` | `u8[16]` | tag   | Authentication tag for the preceding interval |
| `0x10` | `u8[16]` | iv    | IV for the following interval                 |

The reviewed Merlin stream code does not verify received tags and has incomplete
boundary accounting. It supports the intended record layout, but cannot validate
tag correctness or arbitrary stream splits. Those remain
[verification gaps](../../verification.md#encryption).

## Unencrypted mode

The original notes describe the client opting out by sending a single zero
algorithm byte as `EncryptedMessage`, with a length of `1`. This is separate
from the trailing frame zero. Restrictions on which login messages KI accepts in
this mode have not been verified for a current server revision.

The original notes also describe a server opting out by omitting the signed
message. Whether this means a zero-length extended blob or the older layout
without a length prefix is not verified. The reviewed repositories do not
establish whether a current client accepts this downgrade.
