# Bolt Protocol Core v1

**Version:** 1.0.0
**Status:** Draft
**Date:** 2026-02-19

---

## 1. Overview

- Protocol scope: identity, pairing, session security, message semantics, transfer state machines, conformance rules
- Out of scope: transport selection, discovery mechanism, rendezvous details, application UI, storage format
- Design principles: no novel crypto, proven primitives, minimal complexity

### 1.1 Wire Model

Bolt Core defines:

- message semantics
- security requirements (Bolt-layer protection)
- state machines and conformance rules

A Profile defines:

- transport bindings (how bytes move between peers)
- framing (datagram vs stream, and how message boundaries are preserved)
- encoding identifiers and serialization formats
- rendezvous mechanism (if any) used to establish a peer channel
- resource limit defaults and platform policies

Core requirements for Profiles:

- Profiles MUST provide either:
  - message boundary preservation, or
  - a deterministic framing layer suitable for unambiguous envelope extraction
- Profiles MUST define a deterministic canonical serialization for each encoding:
  - Two equivalent messages MUST serialize to identical byte sequences before encryption
- Profiles MUST declare whether the underlying transport provides confidentiality and integrity
  - This is defense-in-depth only
  - Bolt does not depend on transport-layer security for correctness or confidentiality

---

## 2. Identity

- Device identity: persistent X25519 public key, pinned on first use (TOFU)
- v1 MUST support persistent device identity (keypair stored across sessions)
- Trust model: key pinning (Trust On First Use)
  - A device is trusted by recognizing its pinned public key from a prior session
  - This is authenticated key agreement by prior trust, not cryptographic identity verification via signatures
- v2 note: Ed25519 identity keys for signed assertions and mutual authentication
- Peer code is NOT identity: it is a rendezvous token used by a Profile to route connection requests
- Peer code: 6 characters from 32-char unambiguous alphabet (`ABCDEFGHJKMNPQRSTUVWXYZ23456789`)
- Device metadata: name (string), type (`phone` | `tablet` | `laptop` | `desktop`)

### TOFU Flow

- **First contact** (unknown key): implementation SHOULD prompt SAS verification. If user accepts, pin the remote identity public key for future sessions.
- **Known key match**: proceed without user interaction.
- **Key mismatch** (pinned key differs from received key):
  - implementation MUST send `ENVELOPE(ERROR(KEY_MISMATCH))` and close the session
  - implementation MUST present a clear warning to the user
  - to reconnect, the user must explicitly approve re-pairing, which MUST:
    - delete the old pinned key
    - store the new key
    - require SAS confirmation

### Pinning Scope

- Pinning is keyed by the remote identity public key (raw 32 bytes)
- Implementations SHOULD store a mapping of friendly device name to pinned key, but the pinned key is authoritative
- On key mismatch: MUST block, MUST require explicit re-pair approval, MUST require SAS confirmation

---

## 3. Session Establishment

- Each connection generates a fresh ephemeral X25519 keypair
- Persistent identity keys are exchanged (inside encrypted HELLO) for TOFU verification
- Ephemeral keys are used for Bolt-layer message protection and provide forward secrecy
- Profiles define peer-channel setup; Core defines key usage and handshake gating

### Key Roles

- **Persistent identity keypair**
  - Used only for TOFU key pinning and SAS computation
  - MUST NOT be used directly for bulk encryption of Bolt messages
- **Ephemeral session keypair**
  - Fresh per connection
  - Used for encryption of all protected Bolt messages (control and data)
  - MUST NOT be rotated mid-session in v1

### Encryption Key Usage

- All protected messages are encrypted using the ephemeral keypair via NaCl box
- There is no separate session key derivation step; NaCl box internally computes the shared secret via X25519 ECDH
- One ephemeral keypair per connection, reused for all protected messages within that session
- Ephemeral keys discarded on disconnection (forward secrecy)

### Directional Encryption Mapping (Normative)

- Each peer has exactly one ephemeral keypair per connection: `(eph_pub, eph_sec)`
- For a protected message sent from A to B, carried as an envelope:
  - A MUST encrypt using `box(plaintext, nonce, B_eph_pub, A_eph_sec)`
  - The envelope MUST include `sender_ephemeral_key = A_eph_pub`
  - B MUST decrypt using `box.open(ciphertext, nonce, A_eph_pub, B_eph_sec)`
- The same ephemeral keypairs are used for both directions; directionality comes from which pub and sec are applied

### Bootstrap Rule

- The first protected message after peer-channel establishment MUST be an encrypted HELLO inside an envelope
- Each envelope carries `sender_ephemeral_key` in cleartext, allowing decryption without any transport-specific key exchange
- Each peer MUST send exactly one HELLO per connection attempt

### Handshake Completion Rule (Normative)

Handshake is complete only after:

- both peers successfully exchange and decrypt HELLO messages
- identity verification and TOFU checks succeed

Before handshake completion, a peer MUST accept only:

- `PING` and `PONG` (plaintext)
- encrypted envelopes containing `HELLO`
- encrypted envelopes containing `ERROR`

All other messages MUST be rejected with `ENVELOPE(ERROR(INVALID_STATE))`.

### SAS Verification

- Short Authentication String for user confirmation of key exchange
- SAS binds identity and ephemeral keys
- `sort32(a,b)`: lexicographically sort two 32-byte values and concatenate

SAS inputs in v1:

- `identity_A`, `identity_B`: raw identity public keys from decrypted HELLO messages
- `ephemeral_A`, `ephemeral_B`: raw ephemeral public keys from the envelope headers that carried those HELLO messages

Computation:

- `SAS_input = SHA-256( sort32(identity_A, identity_B) || sort32(ephemeral_A, ephemeral_B) )`
- Display first 6 hex chars uppercase
- SAS MUST be computed over raw 32-byte keys, not encoded representations
- 6 hex chars = 24 bits of entropy
- SAS is OPTIONAL but RECOMMENDED for first-time pairing
- v2 note: consider 10-char base32 for stronger verification

### Trust Model

- Trust is established by key pinning plus optional SAS verification
- Forward secrecy provided by ephemeral session keys
- v2 hardening: transcript hash binding and explicit KDF step

---

## 4. Version and Capability Negotiation

- HELLO message is sent as the first encrypted envelope after peer-channel open
- Highest common version selected
- Unknown fields within known message types MUST be ignored
- Unknown message types:
  - MUST be ignored after handshake completion (only possible inside an envelope)
  - During handshake: if unknown messages prevent HELLO completion, respond with `ENVELOPE(ERROR(INVALID_STATE))`
- Encoding MUST be negotiated before transfer begins
- Capabilities are strings. Reserved namespace: `bolt.*`. Unknown capabilities ignored.

### HELLO Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `bolt_version` | uint32 | required | Protocol version (1 for this spec) |
| `capabilities` | string[] | required | List of supported capabilities |
| `encoding` | string | required | Encoding identifier |
| `identity_key` | bytes32 | required | Persistent X25519 public key |
| `limits` | object | optional | Receiver resource limits |
| `limits.max_file_size` | uint64 | optional | Maximum file size in bytes |
| `limits.max_total_chunks` | uint32 | optional | Maximum chunks per transfer |
| `limits.max_concurrent_transfers` | uint32 | optional | Maximum concurrent transfers |

If limits are present, the sender MUST respect the receiver's limits. If absent, Profile defaults apply.

---

## 5. Connection Approval

Connection approval happens via rendezvous (Profile-defined) before the encrypted peer channel is established.

### CONNECTION_REQUEST

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | required | Initiator peer code |
| `device_name` | string | required | Human-readable device name |
| `device_type` | string | required | `phone` \| `tablet` \| `laptop` \| `desktop` |

### CONNECTION_ACCEPTED

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | required | Responder peer code |

### CONNECTION_DECLINED

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string | required | Responder peer code |
| `reason` | string | required | `user_declined` \| `busy` \| `cancelled` |

**Sequence:** `CONNECTION_REQUEST` -> `CONNECTION_ACCEPTED` -> peer-channel setup -> HELLO exchange.

---

## 6. Bolt Message Protection (Normative)

Bolt v1 defines an encrypted message envelope providing confidentiality and integrity independent of transport security.

### Protected Messages

The following message types MUST be transmitted inside an encrypted envelope:

- `HELLO`, `FILE_OFFER`, `FILE_ACCEPT`, `FILE_CHUNK`, `FILE_FINISH`
- `PAUSE`, `RESUME`, `CANCEL`, `ERROR`

### Unprotected Messages

The following message types are plaintext:

- `PING`, `PONG`

Unprotected messages MUST NOT contain sensitive data.

### Security Goal

Bolt-layer protection provides:

- confidentiality from untrusted rendezvous infrastructure
- confidentiality from transport intermediaries
- integrity and authenticity of control and transfer metadata
- transport-independent security guarantees

Transport encryption is defense-in-depth only.

### 6.1 Encrypted Envelope (Normative)

All protected messages are carried inside an encrypted envelope.

#### Envelope Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sender_ephemeral_key` | bytes32 | required | Sender's ephemeral X25519 public key |
| `nonce` | bytes24 | required | Cryptographically random nonce |
| `ciphertext` | bytes | required | NaCl box output |

```
ciphertext = NaCl box(
  plaintext_message_bytes,
  nonce,
  receiver_ephemeral_public,
  sender_ephemeral_secret
)
```

#### Plaintext Message

- MUST be exactly one canonical Bolt message serialization
- MUST NOT batch multiple messages

#### Encryption Rules

Implementations MUST:

- generate a fresh 24-byte random nonce per envelope
- use a cryptographically secure random number generator for nonce generation
- NEVER reuse a nonce with the same ephemeral keypair
- generate a fresh ephemeral keypair per connection
- discard ephemeral keys after session termination

#### Decryption Rules

- Receiver uses `sender_ephemeral_key` from the envelope and its own ephemeral secret key
- Receiver MUST verify MAC before processing message contents

#### Decrypt Failure Handling

- If decrypt fails, receiver SHOULD send `ENVELOPE(ERROR(ENCRYPTION_FAILED))` and MAY terminate the session
- If receiver cannot safely respond, it MAY terminate the session without response

#### Rationale

`sender_ephemeral_key` in cleartext enables transport independence and stateless decryption. Exposure does not reveal plaintext.

---

## 7. Message Model (transport-independent)

All transfer messages include a `transfer_id` to avoid filename collisions.

- `transfer_id`: bytes16, random via CSPRNG (encoding profile-defined)
- `identity_key`: bytes32, X25519 public key (encoding profile-defined)

### Canonical Message Schemas (plaintext inside envelope)

#### HELLO

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `bolt_version` | uint32 | required | Protocol version |
| `capabilities` | string[] | required | Supported capabilities |
| `encoding` | string | required | Encoding identifier |
| `identity_key` | bytes32 | required | Persistent identity key |
| `limits` | object | optional | Receiver resource limits |

#### FILE_OFFER

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Random unique transfer identifier |
| `filename` | string | required | Original filename |
| `size` | uint64 | required | Total file size in bytes |
| `total_chunks` | uint32 | required | Number of chunks |
| `chunk_size` | uint32 | required | Plaintext chunk size in bytes |
| `file_hash` | bytes32 | conditional | SHA-256 of complete plaintext; required if `bolt.file-hash` negotiated |

#### FILE_ACCEPT

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Echoes `transfer_id` from `FILE_OFFER` |

#### FILE_CHUNK

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Identifies the transfer |
| `chunk_index` | uint32 | required | 0-based chunk index |
| `total_chunks` | uint32 | required | Total chunks in file |
| `payload` | bytes | required | Plaintext chunk bytes (envelope provides encryption) |

#### FILE_FINISH

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Identifies the transfer |
| `file_hash` | bytes32 | optional | Echo of `FILE_OFFER` hash (only if `bolt.file-hash` negotiated) |

The `FILE_OFFER` `file_hash` is the authoritative expected value. The receiver computes SHA-256 of the reassembled plaintext and verifies against the `FILE_OFFER` hash. `FILE_FINISH` MAY echo the hash for debugging, but the receiver MUST use the `FILE_OFFER` hash as the verification target.

#### PAUSE

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Identifies the transfer |

#### RESUME

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Identifies the transfer |

#### CANCEL

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `transfer_id` | bytes16 | required | Identifies the transfer |
| `cancelled_by` | string | required | `initiator` \| `responder` |

#### ERROR

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | required | Error code (see section 10) |
| `message` | string | required | Human-readable diagnostic |
| `transfer_id` | bytes16 | optional | If error relates to a specific transfer |
| `details` | any | optional | Implementation-specific context |

#### PING / PONG

No fields. Plaintext (not inside envelope).

---

## 8. File Transfer

### Chunking Rules

- Default chunk size: 16384 bytes (16KB), negotiable via capabilities
- Plaintext split into sequential chunks at offset `i * chunk_size`
- Last chunk MAY be shorter
- `total_chunks = ceil(file_size / chunk_size)`
- Each chunk is sent as a `FILE_CHUNK` message inside its own encrypted envelope
- `FILE_CHUNK.payload` is plaintext; the envelope encrypts it

### Backpressure

- Sender MUST respect transport flow control
- Sender SHOULD NOT queue more data than transport can handle

### File Integrity Verification

When `bolt.file-hash` is negotiated:

- `FILE_OFFER` MUST include SHA-256 of complete plaintext file
- Receiver MUST compute SHA-256 after reassembly and compare
- Mismatch: receiver MUST discard file and send `ERROR(INTEGRITY_FAILED)`

When `bolt.file-hash` is not negotiated:

- Receiver relies on per-message integrity (Poly1305 MAC on each envelope)

### Replay Protection

- Replay detection is scoped per `(transfer_id, chunk_index)`
- Receiver MUST reject duplicate `chunk_index` for the same `transfer_id`
- Receiver MUST reject `chunk_index >= total_chunks`

### Resource Limits

- Implementations MUST enforce limits to prevent DoS
- MUST bound:
  - maximum file size per transfer
  - maximum `total_chunks` per transfer
  - maximum concurrent transfers per session
- `FILE_OFFER` exceeding limits MUST be rejected with `ERROR(LIMIT_EXCEEDED)`
- Limits MAY be advertised in HELLO; sender MUST respect receiver limits when present

### Resume

No retry/resume in v1 (future via `bolt.resume`).

---

## 9. State Machines

### Signaling State (Profile-defined rendezvous)

```
DISCONNECTED -> CONNECTING -> CONNECTED <-> RECONNECTING
```

### Peer Connection State

```
IDLE -> REQUEST_SENT -> APPROVED -> TRANSPORT_CONNECTING -> HANDSHAKING -> CONNECTED -> DISCONNECTED
```

Transitions:

- Enter `TRANSPORT_CONNECTING` when approval granted
- Enter `HANDSHAKING` when peer channel open
- Exit `HANDSHAKING` only when mutual HELLO and TOFU verification succeed
- Enter `CONNECTED` after successful handshake completion

### Transfer State

```
IDLE -> OFFERED -> ACCEPTED -> TRANSFERRING <-> PAUSED -> COMPLETED
                                    |                        |
                                  ERROR <------------- CANCELLED
```

---

## 10. Error Taxonomy

Protocol error codes (sent inside encrypted envelopes):

| Code | Description |
|------|-------------|
| `VERSION_MISMATCH` | No common protocol version |
| `ENCRYPTION_FAILED` | Decryption or MAC verification failed |
| `INTEGRITY_FAILED` | File hash mismatch after reassembly |
| `REPLAY_DETECTED` | Duplicate chunk index received |
| `TRANSFER_FAILED` | Chunk reassembly, I/O, or storage error |
| `LIMIT_EXCEEDED` | File size, chunk count, or concurrent transfer limit exceeded |
| `CONNECTION_LOST` | Transport disconnected during transfer |
| `PEER_NOT_FOUND` | Rendezvous target does not exist |
| `ALREADY_CONNECTED` | Peer is busy with another connection |
| `INVALID_STATE` | Message received in wrong state (e.g. before handshake) |
| `KEY_MISMATCH` | Remote identity key does not match pinned key |

`KEY_MISMATCH` details SHOULD include:

- `pinned_fingerprint`: hex fingerprint of the previously pinned key
- `observed_fingerprint`: hex fingerprint of the key just received

Error code separation:

- `LIMIT_EXCEEDED`: configured resource limits only
- `TRANSFER_FAILED`: I/O, reassembly, storage errors
- `INTEGRITY_FAILED`: file hash mismatch only (when `bolt.file-hash` used)
- `ENCRYPTION_FAILED`: decrypt or MAC failure only

Implementations MAY wrap these in language-specific error types.

---

## 11. Security Properties

- Bolt-layer encryption: all protected messages encrypted via envelope
- Per-message authentication: Poly1305 MAC on every envelope
- File integrity: SHA-256 after reassembly when `bolt.file-hash` negotiated
- TOFU key pinning: persistent identity across sessions
- Nonce freshness: random nonce per envelope, no reuse
- Replay protection: duplicate chunk indices rejected
- Forward secrecy: ephemeral keys discarded after disconnect
- SAS verification: optional user confirmation (24-bit)
- Transport independence: security does not depend on transport encryption

---

## 12. Threat Model

- Rendezvous infrastructure is UNTRUSTED for confidentiality and integrity
- Rendezvous infrastructure MAY observe: peer codes, IP addresses, timing, connection patterns
- Rendezvous infrastructure MUST NOT observe: file contents, filenames, encryption keys, or transfer metadata (encrypted at Bolt layer)
- MITM at rendezvous is probabilistically detectable via SAS
- Network observers may still infer activity via traffic timing and sizes
- Bolt does NOT attempt traffic analysis resistance
- Device compromise: exfiltration of identity secret key enables impersonation in future sessions until re-pair. Past sessions remain safe due to ephemeral session keys. Implementations MUST allow unpair and re-pair.

---

## 13. Conformance

### Requirements

- MUST: support persistent device identity (TOFU)
- MUST: send `ERROR(KEY_MISMATCH)` and close session on key mismatch
- MUST: require SAS confirmation when re-pairing after key mismatch
- MUST: encrypt all protected messages using the envelope
- MUST: send `ERROR` only inside an encrypted envelope
- MUST: reject protected messages received outside an envelope
- MUST: use fresh ephemeral keys per connection
- MUST: NOT rotate ephemeral keys mid-session in v1
- MUST: verify envelope MAC before processing contents
- MUST: generate fresh random nonce per envelope using a CSPRNG and never reuse it
- MUST: complete handshake before sending transfer messages
- MUST: send exactly one HELLO per connection attempt
- MUST: use `transfer_id` to identify transfers
- MUST: reject duplicate chunk indices (scoped per `(transfer_id, chunk_index)`)
- MUST: reject `chunk_index >= total_chunks`
- MUST: verify remote identity matches pinned key when previously seen
- MUST: ignore unknown fields within known message types
- MUST: ignore unknown message types after handshake completion
- MUST: enforce resource limits
- MUST: verify `file_hash` after reassembly when `bolt.file-hash` negotiated
- SHOULD: implement SAS verification on first pairing
- SHOULD: advertise limits in HELLO
- SHOULD: implement backpressure handling
- MAY: support pause/resume

---

## 14. Constants

| Constant | Value |
|----------|-------|
| `NONCE_LENGTH` | 24 bytes |
| `PUBLIC_KEY_LENGTH` | 32 bytes |
| `SECRET_KEY_LENGTH` | 32 bytes |
| `DEFAULT_CHUNK_SIZE` | 16384 bytes |
| `TRANSFER_ID_LENGTH` | 16 bytes |
| `PEER_CODE_LENGTH` | 6 characters |
| `PEER_CODE_ALPHABET` | `ABCDEFGHJKMNPQRSTUVWXYZ23456789` (32 chars) |
| `SAS_LENGTH` | 6 hex characters (uppercase) |
| `SAS_ENTROPY` | 24 bits |
| `FILE_HASH_ALGORITHM` | SHA-256 |
| `FILE_HASH_LENGTH` | 32 bytes |
| `BOLT_VERSION` | 1 |
| `CAPABILITY_NAMESPACE` | `bolt.*` (reserved) |

---

## Appendix A: Profile System

- Bolt Core is transport-agnostic
- Profiles define rendezvous, transport, encoding, framing, limits, and policies
- Known profiles: LocalBolt Profile v1, ByteBolt Profile (future)
- A ByteBolt implementation SHOULD support the LocalBolt Profile for LAN interoperability

## Appendix B: Key Rotation (out of scope for v1)

Key rotation is not defined in Bolt Core v1. Implementations re-key by unpairing and re-pairing, replacing pinned keys. v2 will define explicit `KEY_ROTATE` messages with SAS confirmation.

## Appendix C: Conformance Tests

1. Handshake gating: reject `FILE_OFFER` before HELLO completion -> `ERROR(INVALID_STATE)`
2. Key mismatch: pinned key differs -> `ERROR(KEY_MISMATCH)` then close
3. Replay: duplicate `(transfer_id, chunk_index)` -> `ERROR(REPLAY_DETECTED)`
4. Limit: oversize offer -> `ERROR(LIMIT_EXCEEDED)`
5. SAS vectors: raw identity keys + envelope-header ephemeral keys -> expected 6 hex chars
6. Envelope: protected message outside envelope -> reject
7. Plaintext leak: `PING`/`PONG` contain no sensitive data
