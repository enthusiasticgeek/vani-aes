# aes

AES-128 (FIPS-197), pure vāṇी, hardware-agnostic and heap-free.

Extracted from [Dhruva OS](https://github.com/enthusiasticgeek/dhruvaos)'s
Pi 1 kernel, where AES-128 was first written for CCMP (WPA2's own
data-frame encryption: AES-128 in CCM mode, RFC 3610). That original
implementation only ever needed AES in the ENCRYPT direction -- CTR
mode uses AES-encrypt for both encrypting and decrypting the payload,
and CBC-MAC (the other half of CCM) is also encrypt-only. Every
function here takes fixed-size array parameters and returns by value
or writes through a `mut ref` array -- no heap allocator, no
`extern "C"` scratch accessors, no persistent cross-call state. Drop-in
portable to any vāṇी target, with or without a heap.

## What's here

- **AES-128 block cipher, both directions.** Encrypt is the original,
  already FIPS-197 Appendix B-verified implementation, unchanged.
  Decrypt is new -- added because RFC 3394 AES Key Wrap's own *unwrap*
  direction needs it (see below), a real need this package's original
  motivating use case (CCMP) never had. Uses a constant-time,
  table-free S-box construction on both directions (`aes_sbox`/
  `aes_inv_sbox`) rather than a 256-entry lookup table.
- **RFC 3394 AES Key Wrap / Unwrap**, built on the block cipher above.
  Needed to decrypt a WPA2 GTK (Group Temporal Key) out of an
  EAPOL-Key Message 3's own Key Data field, which real access points
  always AES-key-wrap in practice. Bounded to 2..=8 64-bit blocks
  (16..=64 bytes of key material) -- RFC 3394 itself imposes no such
  bound, but this package's own fixed-array design needs one, and
  nothing wrapping/unwrapping a GTK (16 bytes) or IGTK (16/32 bytes)
  ever needs more.

## Correctness

- `aes128_encrypt_block`/`aes128_decrypt_block`: FIPS-197 Appendix B's
  own worked example, both directions, plus a real-vs-tampered-key
  cross-check (a single flipped key bit must NOT produce the same
  ciphertext). Verified fresh against an independent implementation
  (Python's `cryptography.hazmat` AES-ECB) before being hardcoded, not
  recalled from memory.
- `aes_inv_sbox`'s own inverse-affine-then-GF-invert formula was
  confirmed correct by independently computing the TRUE functional
  inverse of this package's own `aes_sbox` (inverting its 256-entry
  output table in a one-off verification script) and checking the
  formula reproduces that exact table for all 256 inputs -- not
  trusted from a remembered formula alone.
- `aes_key_wrap`/`aes_key_unwrap`: RFC 3394's own official 128-bit-KEK
  test vector (Section 4), fetched from the RFC text directly and
  independently cross-checked against Python's
  `cryptography.hazmat.primitives.keywrap` (both agreeing byte for
  byte), plus a negative case (a single flipped ciphertext byte must
  make unwrap fail its own integrity check, not silently return
  garbage plaintext).

All self-tests pass under both `vanic run`'s default backend and
`--backend=llvm` (what Dhruva OS's own kernel build actually uses).

## Layout

- `src/core.vani` -- the primitives alone (no self-tests), for
  consumers (like Dhruva OS's own kernel) that want the functions
  without vendoring test code.
- `src/lib.vani` -- `core.vani`'s own content plus `aes128_self_test`/
  `aes_key_wrap_self_test`, this package's actual `entry` point.
- `test/host_test.vani` / `test/host_core_test.vani` -- host-runnable
  validation (`vanic run test/host_test.vani`).
