> **Canonical Location:** `bolt-protocol/PROTOCOL.md`
> This file is the single authoritative specification.
> SDKs and daemons implement this specification.

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

### Peer Code Security Model

#### Role of Peer Code

The peer code is a **routing and discovery hint**, not an authentication secret.

It exists solely to:

- Allow two peers to locate each other via rendezvous infrastructure.
- Reduce accidental collisions between concurrent sessions.
- Provide a human-typable pairing token.

The peer code does not provide authentication or confidentiality guarantees.

#### Security Chain (Authoritative)

The security model for Bolt transport is:

1. **Peer Code** → rendezvous routing only
2. **Encrypted HELLO** → ephemeral key exchange
3. **TOFU identity pinning** → continuity across sessions
4. **SAS verification** → active MITM detection

Authentication and integrity derive from HELLO + TOFU + SAS.
They do not derive from peer code entropy.

Even if a peer code is guessed:

- The attacker must still complete the encrypted HELLO exchange.
- Identity mismatch triggers TOFU failure.
- Active MITM is detectable via SAS mismatch.

Peer code compromise alone does not compromise session security.

#### Default Length Policy

Alphabet: `ABCDEFGHJKMNPQRSTUVWXYZ23456789` (31 characters)

| Mode | Format | Characters | Entropy | Purpose |
|------|--------|------------|---------|---------|
| Local (LAN) | 6 characters | 6 | ~29.7 bits | Low-friction human entry (default) |
| Remote / Internet | XXXX-XXXX | 8 | ~39.6 bits | Reduced collision probability for wide-area pairing |

12+ characters are unnecessary because peer code is not an authentication primitive.

Profiles define which mode applies in their scope. See LocalBolt Profile v1 for local-mode defaults.

#### Abuse Mitigation

Brute-force resistance is enforced by:

- Rendezvous rate limiting
- Connection throttling
- Server-side room caps
- Optional IP-level controls

Security does not depend on peer code length alone.

#### Non-Goals

Peer code MUST NOT be treated as:

- A password
- A bearer token
- A long-term shared secret
- A substitute for SAS verification

Any design that relies on peer code entropy for authentication violates the Bolt threat model.

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

### Registered Capabilities

| Capability String | Specification | Description |
|-------------------|---------------|-------------|
| `bolt.file-hash` | §8 File Integrity Verification | SHA-256 file hash verification after reassembly |
| `bolt.profile-envelope-v1` | LocalBolt Profile §14 | Profile-level envelope wrapping |
| `bolt.transfer-ratchet-v1` | §16 Bolt Transfer Ratchet (BTR) | Per-transfer key isolation and per-chunk forward secrecy |

### `bolt.transfer-ratchet-v1` Capability Negotiation (Normative)

The `bolt.transfer-ratchet-v1` capability enables the Bolt Transfer Ratchet (BTR)
for per-chunk forward secrecy and per-transfer key isolation. Negotiation follows
the standard HELLO capability intersection model.

#### Negotiation Matrix

| Local Support | Remote Support | Result | Security Level | Behavior |
|---------------|----------------|--------|----------------|----------|
| YES | YES | **Full BTR** — per-transfer DH ratchet + per-chunk symmetric chain | Per-chunk FS, transfer isolation, self-healing | Normal operation |
| YES | NO | **Downgrade** — static ephemeral (v1 behavior) | Session-level FS only | Log `[BTR_DOWNGRADE]`, warn user |
| NO | YES | **Downgrade** — static ephemeral (v1 behavior) | Session-level FS only | Log `[BTR_DOWNGRADE]`, warn user |
| NO | NO | **Static ephemeral** — current v1 behavior | Session-level FS only | Normal operation |
| YES | MALFORMED | **Reject** — peer advertises `bolt.transfer-ratchet-v1` but sends invalid BTR metadata | N/A | `RATCHET_DOWNGRADE_REJECTED` + disconnect |
| MALFORMED | YES | **Reject** — peer advertises `bolt.transfer-ratchet-v1` but sends invalid BTR metadata | N/A | `RATCHET_DOWNGRADE_REJECTED` + disconnect |

**Downgrade-with-warning** is the default compatibility mode for one-sided support.
Implementations MUST:

- Log `[BTR_DOWNGRADE]` when downgrade occurs (with local and remote capability lists).
- Surface a user-visible warning indicating reduced security when feasible.
- Continue the session using static ephemeral encryption (v1 behavior).
- MUST NOT refuse the connection solely due to missing BTR support.

**Malformed BTR metadata** (rows 5–6) applies when a peer advertises
`bolt.transfer-ratchet-v1` in HELLO capabilities but subsequently:

- Sends envelopes missing required BTR fields (§16.2) during a BTR-negotiated transfer.
- Sends BTR fields with invalid types, sizes, or values.
- Claims BTR capability but uses static ephemeral keys for transfer messages.

This is a protocol violation (misadvertised capability), not a normal capability
mismatch. Implementations MUST send `RATCHET_DOWNGRADE_REJECTED` and disconnect.

#### SAS Computation — Unchanged

BTR does NOT alter SAS computation. SAS inputs remain:

- `identity_A`, `identity_B` — raw identity public keys from HELLO
- `ephemeral_A`, `ephemeral_B` — raw ephemeral public keys from HELLO envelope headers

Ratchet-derived keys are NOT SAS inputs. SAS verifies the initial handshake;
BTR extends key material derivation post-handshake.

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
| `ciphertext` | bytes | required | NaCl box output (keyed by ephemeral shared secret or BTR message key — see §16.2) |
| `ratchet_public_key` | bytes32 | conditional | Current DH ratchet public key. Present IFF `bolt.transfer-ratchet-v1` negotiated AND a DH ratchet step occurs (transfer boundary). See §16.2. |
| `ratchet_generation` | uint32 | conditional | DH ratchet epoch counter. Present IFF `ratchet_public_key` is present. Monotonically increasing per session. See §16.2. |
| `chain_index` | uint32 | conditional | Symmetric chain position (= chunk index within current transfer). Present on every envelope in a BTR-negotiated transfer. See §16.2. |

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

#### BTR Envelope Field Rules

When `bolt.transfer-ratchet-v1` is negotiated:

- `ratchet_public_key` and `ratchet_generation` MUST be present on the first
  envelope of each transfer (FILE_OFFER) and whenever a DH ratchet step occurs.
- `chain_index` MUST be present on every FILE_CHUNK envelope.
- `chain_index` MUST equal the chunk_index for that transfer (0-based).
- Receiving a BTR field in a non-BTR session (capability not negotiated) is a
  protocol violation: `PROTOCOL_VIOLATION` + disconnect.
- Receiving a transfer envelope without required BTR fields in a BTR session
  (capability negotiated) is a protocol violation: `RATCHET_STATE_ERROR` + disconnect.
- Unknown BTR fields MUST be ignored (forward compatibility with future ratchet versions).

**Overhead estimate:** +36 bytes on DH-step envelopes (32B public key + 4B generation),
+4 bytes on every FILE_CHUNK envelope (chain_index). Negligible relative to typical
16KB chunk payloads.

#### Decryption Rules

- Receiver uses `sender_ephemeral_key` from the envelope and its own ephemeral secret key (non-BTR), or the BTR-derived message key (BTR sessions — see §16)
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
- When `bolt.transfer-ratchet-v1` is negotiated, replay detection is further
  strengthened by `ratchet_generation`: envelopes with a stale generation MUST
  be rejected. See §11 REPLAY-BTR and §16.

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

This section is the **single canonical wire error code registry** for
Bolt v1. Every error frame sent on the wire MUST use a code from this
registry. Implementations MUST reject inbound error frames carrying
codes not listed here (→ `PROTOCOL_VIOLATION` + disconnect).

> **Registry authority (PROTO-HARDEN-1R1):** This table unifies the
> original §10 protocol-level codes with the enforcement codes formerly
> maintained in `PROTOCOL_ENFORCEMENT.md` Appendix A. Appendix A is now
> non-normative — this table is the sole authority.

Codes are classified into three tiers:

- **PROTOCOL** — defined in this specification. Apply to all transports.
- **ENFORCEMENT** — defined for handshake and envelope enforcement.
  Apply to transport implementations.
- **BTR** — defined for Bolt Transfer Ratchet (§16). Apply only when
  `bolt.transfer-ratchet-v1` is negotiated.

Three codes (`KEY_MISMATCH`, `INVALID_STATE`, `LIMIT_EXCEEDED`) appeared
in both the original §10 and Appendix A. They are classified PROTOCOL
(higher authority).

| Code | Class | When | Framing | Semantics |
|------|-------|------|---------|-----------|
| `VERSION_MISMATCH` | PROTOCOL | during-hello | plaintext | No common protocol version between peers. |
| `ENCRYPTION_FAILED` | PROTOCOL | post-handshake | envelope | Decryption or MAC verification failed on received payload. |
| `INTEGRITY_FAILED` | PROTOCOL | post-handshake | envelope | File hash mismatch after reassembly (requires `bolt.file-hash`). |
| `REPLAY_DETECTED` | PROTOCOL | post-handshake | envelope | Duplicate chunk index received for an active transfer. |
| `TRANSFER_FAILED` | PROTOCOL | post-handshake | envelope | Chunk reassembly, I/O, or storage error during transfer. |
| `LIMIT_EXCEEDED` | PROTOCOL | post-handshake | envelope | File size, chunk count, rate, or concurrency limit exceeded. |
| `CONNECTION_LOST` | PROTOCOL | any | n/a (local) | Transport disconnected during transfer. Locally observed, not sent on wire. |
| `PEER_NOT_FOUND` | PROTOCOL | pre-hello | plaintext | Rendezvous target does not exist. |
| `ALREADY_CONNECTED` | PROTOCOL | pre-hello | plaintext | Peer is busy with another connection. |
| `INVALID_STATE` | PROTOCOL | any | context-dependent | Message received in wrong session state. Plaintext before keys; envelope after. |
| `KEY_MISMATCH` | PROTOCOL | during-hello | plaintext | Remote identity key does not match pinned key (TOFU violation). |
| `DUPLICATE_HELLO` | ENFORCEMENT | during-hello | plaintext | Second HELLO received after key exchange complete. |
| `HELLO_PARSE_ERROR` | ENFORCEMENT | during-hello | plaintext | HELLO outer frame is unparseable. |
| `HELLO_DECRYPT_FAIL` | ENFORCEMENT | during-hello | plaintext | HELLO sealed payload fails decryption. |
| `HELLO_SCHEMA_ERROR` | ENFORCEMENT | during-hello | plaintext | HELLO inner payload missing required fields or has wrong types. |
| `ENVELOPE_REQUIRED` | ENFORCEMENT | post-handshake | plaintext (terminal) | Plaintext frame received in an envelope-required session. Sent as plaintext terminal error + disconnect. |
| `ENVELOPE_UNNEGOTIATED` | ENFORCEMENT | post-handshake | plaintext (terminal) | Envelope-wrapped frame received when `bolt.profile-envelope-v1` was not negotiated. |
| `ENVELOPE_DECRYPT_FAIL` | ENFORCEMENT | post-handshake | plaintext (terminal) | Sealed envelope payload fails decryption. |
| `ENVELOPE_INVALID` | ENFORCEMENT | post-handshake | plaintext (terminal) | Decrypted envelope payload fails parse or schema validation. |
| `INVALID_MESSAGE` | ENFORCEMENT | post-handshake | envelope | Inner message (post-envelope decrypt) fails parse. |
| `UNKNOWN_MESSAGE_TYPE` | ENFORCEMENT | post-handshake | envelope | Inner message parses but contains an unrecognized type field. |
| `PROTOCOL_VIOLATION` | ENFORCEMENT | any | context-dependent | Catch-all for violations not covered by a specific code. Plaintext before keys; envelope after. |
| `RATCHET_STATE_ERROR` | BTR | post-handshake | envelope | BTR state desynchronization: ratchet generation mismatch, unexpected DH ratchet key, or missing required BTR fields in a BTR-negotiated session. Terminal: disconnect. |
| `RATCHET_CHAIN_ERROR` | BTR | post-handshake | envelope | Symmetric chain index mismatch or gap within a BTR-negotiated transfer. Terminal: cancel transfer. |
| `RATCHET_DECRYPT_FAIL` | BTR | post-handshake | envelope | Decryption using BTR-derived message key failed. Distinct from `ENCRYPTION_FAILED` (static ephemeral). Terminal: cancel transfer. |
| `RATCHET_DOWNGRADE_REJECTED` | BTR | post-handshake | envelope | Peer advertised `bolt.transfer-ratchet-v1` but sent non-BTR or malformed BTR envelopes (misadvertised capability). Terminal: disconnect. |

`KEY_MISMATCH` details SHOULD include:

- `pinned_fingerprint`: hex fingerprint of the previously pinned key
- `observed_fingerprint`: hex fingerprint of the key just received

Error code separation:

- `LIMIT_EXCEEDED`: configured resource limits only
- `TRANSFER_FAILED`: I/O, reassembly, storage errors
- `INTEGRITY_FAILED`: file hash mismatch only (when `bolt.file-hash` used)
- `ENCRYPTION_FAILED`: decrypt or MAC failure using static ephemeral keys only
- `RATCHET_DECRYPT_FAIL`: decrypt or MAC failure using BTR-derived message keys
- `RATCHET_STATE_ERROR`: BTR state desynchronization (generation, DH key, missing fields)
- `RATCHET_CHAIN_ERROR`: symmetric chain index mismatch or gap
- `RATCHET_DOWNGRADE_REJECTED`: misadvertised BTR capability (protocol violation)

Framing rules:

- **plaintext**: sent before shared keys are established (pre-hello and during-hello phases).
- **envelope**: MUST be sent inside an encrypted envelope after handshake completion.
- **plaintext (terminal)**: sent as a plaintext terminal error immediately before disconnect, even in envelope-required mode (best-effort delivery per §15.4).
- **context-dependent**: plaintext before keys exist; envelope after handshake.
- **n/a (local)**: locally observed condition, not transmitted on the wire.

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

### BTR Security Properties (when `bolt.transfer-ratchet-v1` negotiated)

The following properties apply ONLY when BTR is active (both peers negotiated
`bolt.transfer-ratchet-v1`). Without BTR, the static ephemeral properties above
apply.

#### REPLAY-BTR — Anti-Replay Across Reconnect/Resume

- Replay detection is extended to `(transfer_id, ratchet_generation, chain_index)`.
- `ratchet_generation` is monotonically increasing per session. A replayed envelope
  from a prior generation MUST be rejected.
- Reconnection creates a new session with a fresh ephemeral handshake. BTR state
  from a prior session MUST NOT carry over (memory-only policy).
- There is no BTR session resume in v1 (BTR-NG1).

#### ISOLATION-BTR — Transfer Isolation

- Each transfer derives an independent root key via HKDF bound to `transfer_id`.
- No key derivation path exists between transfer A's keys and transfer B's keys.
- Compromise of one transfer's key material MUST NOT enable decryption of any
  other transfer's chunks.

#### ORDER-BTR — Ordered-Chunk Assumption

- Receiver MUST reject `chain_index` != `expected_next_index` (no gap tolerance).
- There is no skipped-message-key buffer. Out-of-order delivery is a protocol
  error: `RATCHET_CHAIN_ERROR` + cancel transfer.
- This invariant is justified by Bolt's ordered-reliable transport requirement
  (§1.1 Wire Model: Profiles MUST provide message boundary preservation).

#### EPOCH-BTR — Bounded Compromise Window

- A DH ratchet step occurs at each transfer boundary (FILE_OFFER / FILE_ACCEPT).
- Compromise of the current ratchet state reveals at most the current transfer's
  remaining chunks (from the compromised chain_index forward).
- After the next DH ratchet step (next transfer), forward secrecy is restored
  (self-healing property).
- The compromise window is bounded by: `remaining_chunks × chunk_size` bytes.

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
- MAY: implement `bolt.transfer-ratchet-v1` (BTR)
- If BTR implemented, MUST: derive keys per §16.3 key schedule
- If BTR implemented, MUST: enforce BTR invariants BTR-INV-01 through BTR-INV-11
- If BTR implemented, MUST: support downgrade-with-warning for non-BTR peers
- If BTR implemented, MUST: zeroize all BTR state on disconnect
- If BTR implemented, MUST: reject chain index gaps (no skipped-key buffer)
- If BTR implemented, MUST: pass all BTR conformance vectors (Appendix C)

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
| `PEER_CODE_ALPHABET` | `ABCDEFGHJKMNPQRSTUVWXYZ23456789` (31 chars, unambiguous subset: no 0/O, 1/I/L) |
| `SAS_LENGTH` | 6 hex characters (uppercase) |
| `SAS_ENTROPY` | 24 bits |
| `FILE_HASH_ALGORITHM` | SHA-256 |
| `FILE_HASH_LENGTH` | 32 bytes |
| `BOLT_VERSION` | 1 |
| `CAPABILITY_NAMESPACE` | `bolt.*` (reserved) |
| `BTR_SESSION_ROOT_INFO` | `"bolt-btr-session-root-v1"` |
| `BTR_TRANSFER_ROOT_INFO` | `"bolt-btr-transfer-root-v1"` |
| `BTR_MESSAGE_KEY_INFO` | `"bolt-btr-message-key-v1"` |
| `BTR_CHAIN_ADVANCE_INFO` | `"bolt-btr-chain-advance-v1"` |
| `BTR_DH_RATCHET_INFO` | `"bolt-btr-dh-ratchet-v1"` |
| `BTR_HKDF_HASH` | SHA-256 |
| `BTR_KEY_LENGTH` | 32 bytes |

---

## 15. Handshake Invariants — Authoritative

> **Phase:** PROTO-HARDEN-1
> **Status:** Normative
> **Date:** 2026-02-26

This section formalizes handshake security properties that were previously
implicit across §3, §6, and §10. These invariants are authoritative and
MUST be enforced by all implementations.

### 15.1 Canonical Keying Model: Ephemeral-First

Bolt v1 uses an **ephemeral-first** keying model:

1. Each peer generates a fresh ephemeral X25519 keypair before sending any
   protected message.
2. The first protected message is an encrypted HELLO envelope. The envelope
   header carries `sender_ephemeral_key` in cleartext.
3. The HELLO payload (encrypted under ephemeral keys) carries the persistent
   `identity_key`.
4. The receiver decrypts the HELLO using the sender's ephemeral public key
   (from the envelope header) and its own ephemeral secret key.
5. The receiver extracts `identity_key` from the decrypted HELLO and performs
   TOFU verification.

This means:

- Ephemeral keys are established first (envelope header, cleartext).
- Identity keys are delivered second (HELLO payload, encrypted).
- No protected message may be sent or accepted before the ephemeral keypair
  exists.
- Identity keys are NEVER transmitted in cleartext.

Implementations MUST NOT use an identity-first model where the identity key
is exchanged before the ephemeral key is available.

### 15.2 Cryptographic Binding: Identity to Ephemeral

The identity key and ephemeral key MUST be cryptographically bound within
each session. Bolt v1 achieves this binding through two mechanisms:

**Mechanism 1 — Envelope authentication:**

The `identity_key` is transmitted inside a HELLO message encrypted with
the ephemeral keypair via NaCl box. The Poly1305 MAC on the envelope
authenticates the `identity_key` value under the ephemeral shared secret.
An attacker cannot substitute a different `identity_key` without breaking
the MAC.

**Mechanism 2 — SAS computation:**

SAS binds both key types:

```
SAS_input = SHA-256(
  sort32(identity_A, identity_B) ||
  sort32(ephemeral_A, ephemeral_B)
)
```

If an active MITM substitutes ephemeral keys, the SAS will differ between
peers (assuming honest identity keys). If identity keys are substituted,
the SAS will also differ (assuming honest ephemeral keys). SAS verification
detects any substitution of either key type.

**Invariant (PROTO-HARDEN-01):** An implementation MUST NOT accept an
`identity_key` that was not delivered inside an envelope authenticated
by the session's ephemeral keypair.

**Invariant (PROTO-HARDEN-02):** An implementation MUST compute SAS over
the exact `identity_key` values received in the decrypted HELLO messages
and the exact `sender_ephemeral_key` values from the envelope headers that
carried those HELLO messages. No substitution, caching from prior sessions,
or re-derivation is permitted.

### 15.3 Error Registry Invariants

Bolt v1 defines a single canonical wire error code registry in §10 of
this document. As of BTR-0, §10 contains 26 codes (11 PROTOCOL-class,
11 ENFORCEMENT-class, 4 BTR-class), extending the unified registry
from PROTO-HARDEN-1R1. Appendix A of `PROTOCOL_ENFORCEMENT.md` remains
non-normative.

**Invariant (PROTO-HARDEN-03):** All implementations MUST emit error codes
from the §10 registry. Implementations MUST NOT invent new error codes
without a spec amendment.

**Invariant (PROTO-HARDEN-04):** `PROTOCOL_ENFORCEMENT.md` MUST NOT
define error codes independently. Any error code reference in that
document MUST defer to §10 as the sole authority.

**Invariant (PROTO-HARDEN-05):** Rust and TypeScript implementations MUST
emit the same error code string for the same violation condition. Error
code parity is a conformance requirement.

### 15.4 Post-Handshake Envelope Requirement

After handshake completion (mutual HELLO exchange + TOFU verification),
the envelope requirement is determined by capability negotiation:

- If `bolt.profile-envelope-v1` was negotiated: all post-handshake messages
  MUST be enveloped. No plaintext messages are permitted except PING/PONG.
- If envelope capability was NOT negotiated: legacy plaintext mode applies
  (see PROTOCOL_ENFORCEMENT.md §5).

**Invariant (PROTO-HARDEN-06):** In envelope-required mode, ERROR messages
MUST be transmitted inside an encrypted envelope. The sole exception is
error frames sent as the final message immediately before transport
disconnect, which MAY be plaintext (best-effort delivery per
PROTOCOL_ENFORCEMENT.md §6).

**Invariant (PROTO-HARDEN-07):** An implementation MUST NOT send plaintext
ERROR messages during normal operation in envelope-required mode. The
plaintext error exception applies ONLY to terminal disconnect frames.

### 15.5 HELLO State Machine Guarantees

The HELLO exchange follows a strict state machine with no reentrancy:

```
AWAITING_HELLO -> HELLO_SENT -> HELLO_RECEIVED -> HANDSHAKE_COMPLETE
```

**Invariant (PROTO-HARDEN-08):** The transition from `AWAITING_HELLO` to
`HELLO_SENT` MUST be atomic. Once a HELLO is sent, the implementation MUST
NOT send another HELLO on the same session, regardless of whether a response
has been received.

**Invariant (PROTO-HARDEN-09):** The transition to `HANDSHAKE_COMPLETE` MUST
occur exactly once per session. No state machine path may revisit
`AWAITING_HELLO` or `HELLO_SENT` after reaching `HANDSHAKE_COMPLETE`.

**Invariant (PROTO-HARDEN-10):** Receiving a HELLO when the local state is
already `HANDSHAKE_COMPLETE` MUST trigger `DUPLICATE_HELLO` error and
immediate disconnect. There is no "re-handshake" path.

**Invariant (PROTO-HARDEN-11):** The HELLO state machine MUST NOT have a
reentrancy window. Specifically: if a peer is processing an inbound HELLO
(decrypting, parsing, verifying TOFU), it MUST NOT accept or process a
second inbound HELLO concurrently. Implementations MUST serialize HELLO
processing.

**Invariant (PROTO-HARDEN-12):** Capability negotiation results (including
envelope-required mode) MUST be immutable after `HANDSHAKE_COMPLETE`. No
subsequent message may alter the negotiated capability set.

---

## 16. Bolt Transfer Ratchet (BTR) — Normative

> **Phase:** BTR-0 (Spec Lock)
> **Status:** Normative
> **Date:** 2026-03-09
> **Capability:** `bolt.transfer-ratchet-v1`
> **Prerequisite:** §15 Handshake Invariants (PROTO-HARDEN)

This section specifies the Bolt Transfer Ratchet, a transfer-scoped key agreement
mechanism providing per-chunk forward secrecy and per-transfer key isolation.
BTR is OPTIONAL — it activates only when both peers negotiate
`bolt.transfer-ratchet-v1` via HELLO capability intersection (§4).

### 16.1 Architecture Overview

BTR layers on top of the v1 ephemeral handshake without replacing it:

```
Session Setup (unchanged — §3):
  Fresh X25519 ephemeral DH → shared_secret → NaCl box for HELLO
                                    │
BTR Session Root (new):             │
  HKDF-SHA256(shared_secret, "bolt-btr-session-root-v1") → session_root_key
                                    │
Transfer Key Derivation (new):      │
  Per transfer: HKDF-SHA256(session_root_key, transfer_id) → transfer_root_key
                                    │
Chunk Symmetric Chain (new):        │
  Per chunk: KDF(chain_key) → (next_chain_key, message_key)
  Encrypt chunk plaintext with message_key via NaCl box
                                    │
Inter-Transfer DH Ratchet (new):    │
  At transfer boundary: new X25519 DH keypair → new session_root_key
  Self-healing: forward secrecy restored across transfer boundaries
```

### 16.2 Envelope Fields

When `bolt.transfer-ratchet-v1` is negotiated, the envelope schema (§6.1) is
extended with conditional fields:

| Field | Type | When Present | Description |
|-------|------|-------------|-------------|
| `ratchet_public_key` | bytes32 | DH ratchet step (transfer boundary) | New DH ratchet public key |
| `ratchet_generation` | uint32 | DH ratchet step (transfer boundary) | Monotonically increasing epoch counter |
| `chain_index` | uint32 | Every FILE_CHUNK in BTR session | Symmetric chain position (= chunk_index) |

**Presence rules:**

- FILE_OFFER envelope: MUST include `ratchet_public_key` and `ratchet_generation`.
  The sender generates a fresh X25519 keypair and performs a DH ratchet step.
- FILE_ACCEPT envelope: MUST include `ratchet_public_key` and `ratchet_generation`.
  The receiver generates a fresh X25519 keypair and completes the DH step.
- FILE_CHUNK envelope: MUST include `chain_index`. MUST NOT include
  `ratchet_public_key` or `ratchet_generation` (no mid-transfer DH steps).
- FILE_FINISH, PAUSE, RESUME, CANCEL: MUST include `chain_index` equal to the
  last chunk's chain_index. No DH fields.
- Non-transfer messages (ERROR, PING, PONG): No BTR fields.

### 16.3 Key Schedule

#### Session Root Derivation

After HELLO handshake completes (§3), both peers compute the shared secret from
the ephemeral X25519 DH. BTR derives a session root key:

```
ephemeral_shared_secret = X25519(local_eph_sec, remote_eph_pub)
session_root_key = HKDF-SHA256(
  salt  = empty (zero-length),
  ikm   = ephemeral_shared_secret,
  info  = "bolt-btr-session-root-v1",
  len   = 32
)
```

#### Transfer Root Derivation

For each transfer, a transfer-scoped root key is derived:

```
transfer_root_key = HKDF-SHA256(
  salt  = transfer_id (16 bytes),
  ikm   = current_session_root_key,
  info  = "bolt-btr-transfer-root-v1",
  len   = 32
)
```

The initial `chain_key` for the transfer equals `transfer_root_key`.

#### Symmetric Chain Advancement (Per-Chunk)

For each chunk in a transfer, the chain advances:

```
message_key  = HKDF-SHA256(
  salt  = empty,
  ikm   = chain_key,
  info  = "bolt-btr-message-key-v1",
  len   = 32
)
next_chain_key = HKDF-SHA256(
  salt  = empty,
  ikm   = chain_key,
  info  = "bolt-btr-chain-advance-v1",
  len   = 32
)
```

- `message_key` is used to encrypt/decrypt the chunk payload via NaCl box.
- `chain_key` is replaced by `next_chain_key`. The old `chain_key` MUST be
  zeroized immediately.
- `message_key` MUST be zeroized after single use (encrypt or decrypt).

#### Inter-Transfer DH Ratchet Step

At each transfer boundary (FILE_OFFER sent/received), the initiating peer:

1. Generates a fresh X25519 keypair: `(new_ratchet_pub, new_ratchet_sec)`.
2. Computes new DH shared secret: `dh_output = X25519(new_ratchet_sec, remote_ratchet_pub)`.
3. Derives new session root key:
   ```
   new_session_root_key = HKDF-SHA256(
     salt  = current_session_root_key,
     ikm   = dh_output,
     info  = "bolt-btr-dh-ratchet-v1",
     len   = 32
   )
   ```
4. Increments `ratchet_generation` by 1.
5. Includes `ratchet_public_key` and `ratchet_generation` in the envelope.
6. Zeroizes `new_ratchet_sec` and old `session_root_key`.

The responding peer (FILE_ACCEPT) performs the same steps with its own fresh keypair,
completing the mutual DH ratchet.

### 16.4 Encryption with BTR Message Keys

When BTR is active, chunk encryption uses the BTR-derived `message_key` instead
of the static ephemeral shared secret:

```
ciphertext = NaCl box(
  plaintext_chunk_bytes,
  nonce,               // fresh 24-byte CSPRNG per envelope (unchanged)
  message_key          // BTR-derived, NOT the ephemeral shared secret
)
```

Implementation note: NaCl box normally takes (plaintext, nonce, receiver_pub,
sender_sec). With BTR, both peers derive the same `message_key` deterministically.
Implementations MUST use a symmetric NaCl secretbox (XSalsa20-Poly1305) keyed by
`message_key` for BTR-encrypted envelopes.

The `sender_ephemeral_key` field in the envelope header remains present for
backward compatibility and session identification, but is NOT used for BTR
chunk decryption.

### 16.5 Key Material Lifecycle

#### State Objects

| Object | Scope | Size | Lifetime | Cleanup Trigger |
|--------|-------|------|----------|-----------------|
| `session_root_key` | Per session | 32 B | Session start → disconnect | Disconnect |
| `ratchet_keypair` | Per DH step | 64 B | Transfer boundary → next boundary | Next DH step or disconnect |
| `transfer_root_key` | Per transfer | 32 B | FILE_OFFER → transfer end | Transfer complete, cancel, or disconnect |
| `chain_key` | Per chain step | 32 B | Chunk N → chunk N+1 | Next chain advance (immediate) |
| `message_key` | Single use | 32 B | Encrypt or decrypt one chunk | Immediate after use |
| `ratchet_generation` | Per session | 4 B | Session start → disconnect | Disconnect |

**Total per-session state:** ~164 bytes. No skipped-key buffer required.

#### Memory-Only Policy (Normative)

All BTR key material MUST be stored in memory only. Implementations:

- MUST NOT persist any BTR state to disk, database, or non-volatile storage.
- MUST NOT log key material (session root, transfer root, chain key, message key,
  ratchet secret key) at any log level.
- MUST zeroize all BTR state on disconnect (SEC-04, SEC-05 preserved).
- MUST generate fresh BTR state on reconnection (no session resume in v1).

This policy is locked per PM-BTR-03. Session resumption (persistent ratchet
state across disconnects) is an explicit non-goal for BTR v1 (BTR-NG1).

#### Cleanup Points

| Event | Action |
|-------|--------|
| Transfer complete (FILE_FINISH) | Zeroize: `transfer_root_key`, `chain_key`, all derived `message_key`s. Retain: `session_root_key`, `ratchet_keypair`. |
| Transfer cancel (CANCEL) | Zeroize: all transfer-scoped state immediately. Retain: session state. |
| Disconnect | Zeroize: ALL BTR state (`session_root_key`, `ratchet_keypair`, any in-flight transfer state). |
| Reconnect | Fresh ephemeral handshake → fresh `session_root_key`. No state carryover. |
| DH ratchet step | Zeroize: old `ratchet_secret_key`, old `session_root_key`. Retain: new values. |
| Chain advance | Zeroize: old `chain_key` immediately after deriving `next_chain_key` + `message_key`. |

#### Prohibited Operations

- Persisting BTR state to disk (violates SEC-04).
- Reusing `message_key` for multiple chunks (violates single-use property).
- Carrying BTR state across sessions (violates SEC-05, memory-only policy).
- Skipping chain indices (violates ORDER-BTR — no gap tolerance).

### 16.6 BTR Security Invariants

| ID | Invariant | Normative |
|----|-----------|-----------|
| BTR-INV-01 | Session root key MUST be derived from ephemeral shared secret via HKDF, not used directly | REQUIRED |
| BTR-INV-02 | Transfer root key MUST bind to `transfer_id` via HKDF salt | REQUIRED |
| BTR-INV-03 | Chain key MUST advance per chunk; old chain key zeroized immediately | REQUIRED |
| BTR-INV-04 | Message key MUST be single-use; zeroized after encrypt or decrypt | REQUIRED |
| BTR-INV-05 | DH ratchet step MUST use a fresh X25519 keypair per transfer boundary | REQUIRED |
| BTR-INV-06 | Ratchet generation MUST be monotonically increasing per session | REQUIRED |
| BTR-INV-07 | Chain index gap MUST be rejected (no skipped-key buffer) | REQUIRED |
| BTR-INV-08 | All BTR key material MUST be memory-only (no disk persistence) | REQUIRED |
| BTR-INV-09 | All BTR state MUST be zeroized on disconnect | REQUIRED |
| BTR-INV-10 | BTR MUST NOT alter SAS computation inputs | REQUIRED |
| BTR-INV-11 | BTR-encrypted envelopes MUST use NaCl secretbox keyed by message_key | REQUIRED |

### 16.7 BTR Error Behavior

| Error Code | Trigger | Required Action |
|------------|---------|-----------------|
| `RATCHET_STATE_ERROR` | Ratchet generation mismatch, unexpected DH key, missing required BTR fields | Send error inside envelope, disconnect immediately |
| `RATCHET_CHAIN_ERROR` | `chain_index` != expected next, chain index gap | Send error inside envelope, cancel transfer |
| `RATCHET_DECRYPT_FAIL` | NaCl secretbox open fails with BTR message key | Send error inside envelope, cancel transfer |
| `RATCHET_DOWNGRADE_REJECTED` | Peer advertised BTR but sends non-BTR or invalid BTR envelopes | Send error inside envelope, disconnect immediately |

All BTR errors are sent inside encrypted envelopes (post-handshake). The
`transfer_id` field SHOULD be included when the error relates to a specific transfer.

### 16.8 BTR-1 Entry Criteria

BTR-0 (this spec lock) gates all subsequent implementation phases. BTR-1
(Rust reference implementation) may begin when ALL of the following are satisfied:

1. This section (§16) is tagged and published in `bolt-protocol`.
2. §4 capability negotiation matrix is locked (6 cells defined).
3. §10 error registry includes all 4 BTR error codes.
4. §11 BTR security invariants are locked (BTR-INV-01 through BTR-INV-11).
5. §16.3 key schedule (HKDF info strings, derivation chain) is locked.
6. §16.5 lifecycle cleanup points are locked.
7. Ecosystem governance docs mark BTR-0 as DONE and BTR-1 as UNBLOCKED.

---

## 17. BTR Security Claims

> **Scope:** These claims apply ONLY when both peers have negotiated
> `bolt.transfer-ratchet-v1` via §4 capability intersection. Without BTR,
> only the static ephemeral properties of §11 apply.
>
> **Authority:** Normative. Claims are bound to invariants in §16.6
> (BTR-INV-01 through BTR-INV-11) and validated by conformance vectors
> in Appendix C.

### 17.1 Threat Model

#### Assumed Attacker Capabilities

BTR is designed to resist an active network-level attacker who can:

- **Intercept** all messages between peers (full passive eavesdropping).
- **Inject** arbitrary messages into the transport channel.
- **Replay** previously observed messages in any order or quantity.
- **Reorder** or **delay** messages arbitrarily.
- **Truncate** or **drop** messages selectively.
- **Compromise infrastructure** (rendezvous server, relay) — these are
  explicitly untrusted (§12).

#### Assumptions (Required for Claims to Hold)

| Assumption | Rationale |
|------------|-----------|
| Endpoints are not compromised during an active session | BTR protects wire traffic, not local memory. A compromised endpoint can read plaintext directly. |
| X25519 is computationally secure (CDH assumption) | DH ratchet steps and ephemeral handshake depend on the hardness of the elliptic-curve Diffie-Hellman problem over Curve25519. |
| HKDF-SHA256 is a secure PRF in the extract-then-expand model | All key derivation (session root, transfer root, chain advancement) uses HKDF-SHA256 (§16.3). |
| XSalsa20-Poly1305 (NaCl box / secretbox) provides IND-CCA2 security | Envelope encryption (§6.1) and BTR chunk encryption (§16.4) rely on this construction. |
| CSPRNG produces uniformly random output | Nonces (§6.1), ephemeral keypairs (§3, §16.3), and transfer IDs depend on cryptographic randomness. |
| Transport provides ordered, reliable delivery with message boundaries | BTR's no-gap chain index policy (ORDER-BTR, §11) depends on §1.1 Wire Model guarantees. |

#### Explicit Non-Goals

The following are NOT security goals of BTR:

- **Metadata privacy.** Transfer timing, sizes, peer codes, and connection
  patterns are observable by network adversaries and infrastructure (§12).
- **Post-compromise local secrecy.** If an endpoint is compromised, the
  attacker can read all plaintext in local memory. BTR protects wire traffic
  only.
- **Deniability.** BTR does not provide cryptographic deniability for
  transferred content.
- **Quantum resistance.** All DH operations use classical Curve25519.

### 17.2 Security Goals (Normative)

| Goal | Claim | Mechanism | Binding Invariant(s) |
|------|-------|-----------|---------------------|
| **Confidentiality** | An attacker who observes all wire traffic learns nothing about plaintext chunk content. | Each chunk is encrypted with a unique `message_key` derived via the BTR chain (§16.3). Keys are never reused (BTR-INV-04). | BTR-INV-03, BTR-INV-04, BTR-INV-11 |
| **Integrity / Authenticity** | Receiver detects any modification, truncation, or injection of chunk data. | Poly1305 MAC on every envelope (§6.1). BTR-encrypted chunks use NaCl secretbox which is authenticated encryption. `chain_index` validation rejects insertions (BTR-INV-07). | BTR-INV-07, BTR-INV-11 |
| **Per-chunk forward secrecy** | Compromise of `chain_key[n]` does not reveal `message_key[k]` for any `k < n`. | Symmetric chain advancement is one-way: `chain_key[n]` derives `chain_key[n+1]` via HKDF; the reverse is computationally infeasible. Old chain keys are zeroized immediately (§16.5). | BTR-INV-03 |
| **Per-transfer forward secrecy (self-healing)** | Compromise of all key material for transfer T does not reveal keys for transfer T+1 or later. | A fresh X25519 DH ratchet step at each transfer boundary (§16.3) introduces new entropy that an attacker without the private key cannot derive. | BTR-INV-05, BTR-INV-09 |
| **Transfer isolation** | Compromise of transfer A's keys does not enable decryption of transfer B's chunks. | Each transfer derives an independent `transfer_root_key` via HKDF bound to a unique `transfer_id` (§16.3). No derivation path connects distinct transfer key trees. | BTR-INV-02 |
| **Replay resistance** | Replayed envelopes from prior generations or duplicate chain indices are rejected. | Tuple `(transfer_id, ratchet_generation, chain_index)` is validated. `ratchet_generation` is monotonically increasing (BTR-INV-06). Duplicate `chain_index` is rejected (BTR-INV-07). | BTR-INV-06, BTR-INV-07 |
| **Downgrade safety** | A peer cannot be silently downgraded from BTR to static ephemeral mode. | Capability negotiation (§4) is performed inside encrypted HELLO envelopes. Mismatch between advertised and observed behavior triggers `RATCHET_DOWNGRADE_REJECTED` (§16.7). | BTR-INV-10 |
| **Key separation** | Session keys, transfer keys, chain keys, and message keys occupy distinct cryptographic domains. | Each derivation step uses a unique HKDF `info` string (§16.3): `bolt-btr-session-root-v1`, `bolt-btr-transfer-root-v1`, `bolt-btr-message-key-v1`, `bolt-btr-chain-advance-v1`. | BTR-INV-01, BTR-INV-02 |
| **Zeroization** | Sensitive key material is not retained beyond its useful lifetime. | Memory-only policy (§16.5): no BTR key material is persisted to disk. Chain keys, message keys, and ephemeral private keys are zeroized at defined cleanup points. | BTR-INV-03, BTR-INV-04, BTR-INV-08, BTR-INV-09 |

### 17.3 Out-of-Scope / Not Guaranteed

The following are explicitly NOT guaranteed by BTR. Implementations and users
MUST NOT rely on BTR for these properties:

1. **Traffic analysis resistance.** BTR does not pad, delay, or otherwise
   obscure traffic patterns. An observer can infer transfer activity, file
   sizes, and timing.

2. **Anonymous transfer.** Peer codes and identity keys are exchanged during
   signaling and handshake. BTR does not provide sender or receiver anonymity.

3. **Protection against compromised endpoints.** If either peer's device is
   under attacker control, all plaintext is accessible locally. BTR's
   guarantees are wire-only.

4. **Resistance to malicious peer behavior.** BTR assumes both peers are
   honest-but-curious at worst. A malicious peer who has completed the
   handshake can always read the plaintext they receive.

5. **Post-quantum security.** All DH operations (ephemeral handshake and
   inter-transfer ratchet) use Curve25519. A quantum adversary with a
   sufficiently powerful quantum computer could break these.

6. **Session resumption.** BTR v1 does not support session resume. Each
   new connection requires a full ephemeral handshake. Prior BTR state MUST
   NOT carry over (BTR-NG1, §16.5).

7. **Partial delivery guarantees.** BTR does not provide reliable delivery.
   If the transport drops a chunk, the receiver will reject subsequent chunks
   due to chain index gap (ORDER-BTR). Recovery requires retransmission at
   the transport layer or transfer restart.

### 17.4 Primitive and Construction Rationale

| Primitive | Usage in BTR | Rationale |
|-----------|-------------|-----------|
| **X25519** | Ephemeral handshake DH (§3); inter-transfer DH ratchet step (§16.3) | Widely deployed, well-analyzed, constant-time implementations available. Provides 128-bit security level. Compatible with existing Bolt v1 handshake. |
| **HKDF-SHA256** | Session root derivation, transfer root derivation, symmetric chain advancement (§16.3) | Standard extract-then-expand KDF (RFC 5869). Domain separation via distinct `info` strings prevents cross-purpose key confusion. SHA-256 provides collision resistance sufficient for 128-bit security. |
| **NaCl secretbox (XSalsa20-Poly1305)** | BTR chunk encryption (§16.4) | Same AEAD construction used by Bolt v1 envelope encryption (§6.1). Avoids introducing a second cipher suite. Nonce misuse is prevented by single-use message keys (BTR-INV-04). |
| **No skipped-key buffer** | Chain index gap rejection (ORDER-BTR) | Bolt's wire model (§1.1) guarantees ordered delivery. A skipped-key buffer would add implementation complexity, expand the attack surface (state accumulation), and serve no purpose given the transport guarantee. |
| **Per-transfer DH ratchet (not per-chunk)** | Inter-transfer ratchet step (§16.3) | Per-chunk DH would add one X25519 scalar multiplication per chunk — prohibitive for large file transfers. Per-transfer DH provides self-healing forward secrecy at transfer boundaries while keeping per-chunk overhead to symmetric-only operations. |

### 17.5 Conformance and Evidence Requirements

#### Claim-to-Evidence Mapping

| Claim | Invariants | Conformance Vectors (Appendix C) | Cross-Language Suite |
|-------|-----------|----------------------------------|---------------------|
| Confidentiality | BTR-INV-03, 04, 11 | `btr-key-schedule`, `btr-chain-advance` | Rust↔TS interop: Rust-encrypted chunks MUST decrypt in TS and vice versa |
| Integrity | BTR-INV-07, 11 | `btr-replay-reject` | Both implementations MUST reject malformed BTR messages identically |
| Per-chunk FS | BTR-INV-03 | `btr-chain-advance` | Chain advancement outputs MUST match across implementations |
| Per-transfer FS | BTR-INV-05, 09 | `btr-transfer-ratchet` | Independent transfers MUST produce independent key chains |
| Transfer isolation | BTR-INV-02 | `btr-transfer-ratchet` | Same session, different transfer_ids → different transfer_root_keys |
| Replay resistance | BTR-INV-06, 07 | `btr-replay-reject` | Cross-transfer and within-transfer replay rejection verified |
| Downgrade safety | BTR-INV-10 | `btr-downgrade-negotiate` | All 6 negotiation matrix paths verified |
| Key separation | BTR-INV-01, 02 | `btr-key-schedule`, `btr-transfer-ratchet` | Derivation with distinct info strings produces distinct outputs |
| Zeroization | BTR-INV-03, 04, 08, 09 | — (runtime property, not vector-testable) | Implementation review required |

#### Evidence Requirements for Future Changes

Any change to the BTR key schedule, primitives, or security claims MUST
provide:

1. Updated conformance vectors demonstrating the new behavior.
2. Cross-language test results (Rust and TypeScript at minimum).
3. At least one adversarial test case per affected claim.
4. Updated invariant table (§16.6) if invariant semantics change.

### 17.6 Change-Control Security Policy

1. **No claim expansion without evidence.** A new security claim MUST NOT be
   added to §17.2 unless accompanied by:
   - A binding invariant in §16.6.
   - At least one conformance vector in Appendix C.
   - Cross-language test coverage.

2. **Claim-impacting changes.** Any protocol change that weakens, strengthens,
   or alters a claim in §17.2 MUST:
   - Update the normative spec text in §16 and §17.
   - Update or add conformance vectors in Appendix C.
   - Include a compatibility statement (does the change break interop with
     prior versions?).
   - Include a security impact note (which claims are affected and how?).

3. **Primitive substitution.** Replacing a primitive listed in §17.4 (e.g.,
   X25519 → X448, SHA-256 → SHA-3) requires:
   - Updated HKDF info strings to prevent cross-version key confusion.
   - New conformance vectors for the replacement primitive.
   - Capability version increment (`bolt.transfer-ratchet-v2` or higher).
   - Security analysis demonstrating the replacement meets or exceeds the
     original security level.

4. **Deprecation.** Removing a security claim MUST be documented with:
   - Rationale for removal.
   - Migration guidance for implementations relying on the claim.
   - Transition period specified in the capability negotiation matrix.

---

## Appendix A: Profile System

- Bolt Core is transport-agnostic
- Profiles define rendezvous, transport, encoding, framing, limits, and policies
- Known profiles: LocalBolt Profile v1, ByteBolt Profile (future)
- A ByteBolt implementation SHOULD support the LocalBolt Profile for LAN interoperability

## Appendix B: Key Rotation

**Identity key rotation** is not defined in Bolt Core v1. Implementations re-key
identity by unpairing and re-pairing, replacing pinned keys. v2 will define
explicit `KEY_ROTATE` messages with SAS confirmation.

**Session key rotation** is provided by the Bolt Transfer Ratchet (§16) when
`bolt.transfer-ratchet-v1` is negotiated. BTR rotates encryption keys at two
granularities:

- **Per-chunk:** Symmetric chain advancement derives a fresh message key per chunk.
- **Per-transfer:** DH ratchet step at each transfer boundary derives a fresh
  session root key, providing self-healing forward secrecy.

Without BTR, a single ephemeral shared secret is used for all messages in a
session (v1 static ephemeral model).

## Appendix C: Conformance Tests

### Core Conformance (v1)

1. Handshake gating: reject `FILE_OFFER` before HELLO completion -> `ERROR(INVALID_STATE)`
2. Key mismatch: pinned key differs -> `ERROR(KEY_MISMATCH)` then close
3. Replay: duplicate `(transfer_id, chunk_index)` -> `ERROR(REPLAY_DETECTED)`
4. Limit: oversize offer -> `ERROR(LIMIT_EXCEEDED)`
5. SAS vectors: raw identity keys + envelope-header ephemeral keys -> expected 6 hex chars
6. Envelope: protected message outside envelope -> reject
7. Plaintext leak: `PING`/`PONG` contain no sensitive data

### BTR Conformance (when `bolt.transfer-ratchet-v1` implemented)

Test vector categories required for BTR conformance. Vectors are generated by
the Rust reference implementation (BTR-1) and consumed by all implementations
for parity verification.

| Category | Schema | Purpose | Min Vectors |
|----------|--------|---------|-------------|
| `btr-key-schedule` | `{ ephemeral_shared_secret, expected_session_root_key }` | Session root derivation via HKDF | 3 |
| `btr-transfer-ratchet` | `{ session_root_key, transfer_id, expected_transfer_root_key }` | Transfer key derivation bound to transfer_id | 4 |
| `btr-chain-advance` | `{ chain_key, expected_message_key, expected_next_chain_key }` | Per-chunk symmetric chain KDF | 5 |
| `btr-replay-reject` | `{ ratchet_generation, chain_index, expected_reject }` | Cross-transfer and within-transfer replay rejection | 4 |
| `btr-downgrade-negotiate` | `{ local_caps, remote_caps, expected_mode, expected_log }` | All 6 negotiation matrix paths | 6 |

**Conformance pass criteria:**

1. All vector categories MUST pass in both Rust and TypeScript implementations.
2. Cross-language interop: Rust-encrypted chunks MUST decrypt in TS, and vice versa.
3. Transfer isolation: independent transfers MUST produce independent key chains
   (identical session but different transfer_ids → different transfer_root_keys).
4. Adversarial: both implementations MUST reject malformed BTR messages identically
   (same error code for same violation condition).
5. Downgrade: both implementations MUST correctly fall back to v1 static ephemeral
   when BTR is not negotiated, with no BTR fields present.
