# Obfuscation

## Pubkey blob

The reviewed key-blob pattern supports the cipher IDs and key/IV sizes
below. Cipher modes, padding, exact length semantics, and the runtime
transformation remain [verification gaps](../../verification.md#encryption).
This page does not yet specify a complete public-key decoder.

Each public key inside of the binary is encrypted with a symmetric algorithm. There are five different algorithms that can be used for each key:

| Algorithm | N |
|-----------|---|
| SPECK64   | 1 |
| ARIA      | 2 |
| CAMELLIA  | 3 |
| SEED      | 4 |
| LEA       | 5 |

A byte indicating the algorithm, as well as the decryption key and IV immediately precede each key's ciphertext. Note that `SPECK64` uses a 12 byte key and 8 byte IV, while the other ciphers use a 16 byte key and IV.

After decryption, the full plaintext must be XOR'd with a single byte. This byte is sent by the server upon initiating the handshake, but it is also trivial to derive. Each public key is ASN.1 encoded meaning that they always start with `0x30`. By XOR'ing the first byte with `0x30`, you get the byte that would be sent by the server and can use it to decode the rest of the key.

At runtime, an additional layer of obfuscation is applied to the key blob.

Each key's datastructure can be expressed as:

| Field         | Type   | Description                          |
|---------------|--------| ------------------------------------ |
| Length        | uint16 | Expresses length of this structure   |
| Cipher        | byte   | As described in table above          |
| DecryptionKey | byte[] | 12 or 16 bytes depending on cipher   |
| DecryptionIV  | byte[] | 8 or 16 bytes depending on cipher    |
| Ciphertext    | byte[] |                                      |
