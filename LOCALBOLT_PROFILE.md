> **Canonical Location:** `bolt-protocol/LOCALBOLT_PROFILE.md`
> This file is the single authoritative specification.
> SDKs and daemons implement this specification.

# LocalBolt Profile v1

**Version:** 1.0.0
**Status:** Draft
**Date:** 2026-02-19
**Implements:** Bolt Core v1

---

## 1. Overview

- Profile for: local network file transfer via browser peer-to-peer channel
- Implements: Bolt Protocol Core v1
- Discovery scope: local network only (see section 7)

---

## 2. Rendezvous Transport

- Protocol: WebSocket (JSON messages)
- Endpoint: local LAN rendezvous server (`ws://<ip>:3001`)
- LocalBolt Profile v1 uses local-only rendezvous by default
- Rendezvous is explicitly non-security-critical (see Bolt Core section 12)
- Transport provides: message boundary preservation (WebSocket frames), TLS confidentiality when using `wss://` (defense-in-depth only)

### Cloud Discovery Extension

Cloud rendezvous (`wss://...`) is a separate optional extension. When enabled, it broadens discovery beyond the local network.

Implementations using cloud discovery:

- MUST clearly indicate to users that discovery scope extends beyond LAN
- MUST use a distinct UI mode or indicator (e.g. "Internet mode" vs "Local mode")
- MUST NOT present cloud-discovered peers as "local" peers

This extension does not change the Bolt Core protocol.

---

## 3. Rendezvous Wire Format

### Client -> Server

| Message | Format |
|---------|--------|
| Register | `{ "type": "register", "peer_code": "...", "device_name": "...", "device_type": "..." }` |
| Signal | `{ "type": "signal", "to": "<peer_code>", "payload": {...} }` |
| Ping | `{ "type": "ping" }` |

### Server -> Client

| Message | Format |
|---------|--------|
| Peers | `{ "type": "peers", "peers": [...] }` |
| Peer Joined | `{ "type": "peer_joined", "peer": {...} }` |
| Peer Left | `{ "type": "peer_left", "peer_code": "..." }` |
| Signal | `{ "type": "signal", "from": "...", "payload": {...} }` |
| Error | `{ "type": "error", "message": "..." }` |

### Peer Code Validation (server-side)

- Non-empty, max 16 chars, ASCII alphanumeric only

### Peer Code Security Model

See [PROTOCOL.md §2 — Peer Code Security Model](PROTOCOL.md) for the authoritative policy.

LocalBolt Profile uses **6-character peer codes** (local/LAN mode). The 8-character
long format (`XXXX-XXXX`) is available for optional cloud/remote discovery extensions.

Peer code is a routing hint, not an authentication secret. Session security derives
from encrypted HELLO + TOFU + SAS, not from peer code entropy.

---

## 4. Peer Channel

- Browser peer-to-peer data channel
  - Label: `"fileTransfer"`
  - Ordered: `true`
  - Reliable: `true`
  - `binaryType`: `"arraybuffer"`

---

## 5. Connectivity Policy

- Uses public STUN servers
- No relay servers configured (local-only policy)
- Candidate filtering: `host` and `srflx` only; `relay` candidates BLOCKED

---

## 6. Message Encoding (`json-envelope-v1`)

All protected Bolt messages MUST be transmitted as encrypted envelopes.

Encoding identifier negotiated via HELLO: `json-envelope-v1`

### Envelope Wire Format

```json
{
  "type": "bolt-envelope",
  "senderEphemeralKey": "<base64, 32 bytes>",
  "nonce": "<base64, 24 bytes>",
  "ciphertext": "<base64>"
}
```

### Plaintext Message Serialization

- The decrypted payload MUST be UTF-8 JSON representing exactly one canonical Bolt message.
- Canonical JSON for this profile:
  - Keys MUST be emitted in lexicographic order
  - No insignificant whitespace
  - UTF-8 encoding
  - Numbers MUST be base-10 without leading zeros
- This profile uses camelCase field names mapping from Core snake_case fields

### Byte Fields Encoding (inside decrypted plaintext)

| Field | Encoding |
|-------|----------|
| `transfer_id` (bytes16) | Hex string (32 hex chars) |
| `identity_key` (bytes32) | Base64 string |
| `file_hash` (bytes32) | Hex string (64 hex chars) |
| `payload` (bytes) | Base64 string |

### Example: Decrypted HELLO (with optional limits)

```json
{
  "type": "hello",
  "boltVersion": 1,
  "capabilities": ["bolt.file-hash"],
  "encoding": "json-envelope-v1",
  "identityKey": "<base64, 32 bytes>",
  "limits": {
    "maxFileSize": 10737418240,
    "maxTotalChunks": 655360,
    "maxConcurrentTransfers": 1
  }
}
```

### Example: Decrypted FILE_OFFER (with file hash)

```json
{
  "type": "file-offer",
  "transferId": "<hex, 32 chars>",
  "filename": "example.pdf",
  "size": 688128,
  "totalChunks": 42,
  "chunkSize": 16384,
  "fileHash": "<hex, 64 chars>"
}
```

### Example: Decrypted FILE_CHUNK

```json
{
  "type": "file-chunk",
  "transferId": "<hex, 32 chars>",
  "chunkIndex": 0,
  "totalChunks": 42,
  "payload": "<base64, plaintext chunk>"
}
```

### Example: Decrypted FILE_FINISH (optional file hash echo)

```json
{
  "type": "file-finish",
  "transferId": "<hex, 32 chars>",
  "fileHash": "<hex, 64 chars>"
}
```

### Control Messages (decrypted)

```json
{ "type": "pause", "transferId": "..." }
{ "type": "resume", "transferId": "..." }
{ "type": "cancel", "transferId": "...", "cancelledBy": "initiator" }
```

### HELLO Exchange

- The first application message sent over the peer channel MUST be an encrypted envelope containing HELLO
- Each peer MUST send exactly one HELLO per connection attempt
- Handshake completes only after both peers successfully decrypt HELLO
- Before handshake completion, only HELLO/ERROR envelopes and plaintext ping/pong are accepted (per Bolt Core)
- Receivers MUST reject any non-HELLO message before handshake completion with `ERROR(INVALID_STATE)` and SHOULD close the connection (fail-closed)
- At the Profile layer this error is currently plaintext JSON (`{ "type": "error", "code": "INVALID_STATE", "message": "..." }`). Bolt envelope wrapping is not yet implemented.

### Capabilities Negotiation

- HELLO payload MAY include a `capabilities` field (array of strings)
- Receivers MUST treat a missing `capabilities` field as an empty array (backward compatibility with pre-capabilities peers)
- Each peer computes the negotiated capability set as the intersection of local and remote capabilities
- No capability-based gating is enforced in Phase 0; negotiation is plumbing only
- Capabilities follow the `bolt.<name>` namespace convention (e.g. `bolt.file-hash`, `bolt.envelope-v1`)

### Plaintext Messages

Only the following are sent without encryption:

```json
{ "type": "ping" }
{ "type": "pong" }
```

These MUST NOT contain sensitive data.

---

## 7. Local Scope Policy

LocalBolt prefers local network connections. This is a HEURISTIC, not enforcement.

### Mechanisms

- IP-based room grouping: rendezvous server groups private IPs into shared "local" room
- Relay candidate blocking: no relay servers configured, relay candidates dropped
- Private IP recognition: RFC 1918, link-local, CGNAT/Tailscale (100.64/10), IPv6 ULA/link-local

### Known Limitations

- VPN clients may appear local when they are remote
- CGNAT devices may appear local when they are not on same mesh
- Multi-homed devices may be in different rooms on different interfaces
- This is product-level policy, not protocol-level security

---

## 8. Resource Limits

| Limit | Value |
|-------|-------|
| Maximum file size per transfer | 10 GB (without explicit user approval) |
| Maximum `total_chunks` per transfer | Derived: `max_file_size / chunk_size` (655360 at 16KB; recompute if chunk size changes) |
| Maximum concurrent transfers per session | 1 |

---

## 9. Reconnection

- Exponential backoff: `delay = min(1000 * 2^attempt, 30000)` ms
- Keepalive: ping every 30s

---

## 10. Key Exchange Binding

- The authoritative ephemeral key used for Bolt encryption is the `senderEphemeralKey` field present in each envelope.
- Transport-level key carriage (e.g. offer/answer metadata) MAY be used as an optimization but MUST NOT be required for correctness.
- Implementations MAY compare transport-carried keys with observed envelope keys and warn on mismatch.

---

## 11. Platform-Specific Chunk Sizes

| Platform | Chunk Size |
|----------|-----------|
| Default | 16384 (16KB) |
| Mobile | 8192 (8KB) |
| Desktop/Laptop | 16384 (16KB) |
| Steam Deck | 32768 (32KB) |

---

## 12. Replay Protection

### Transfer Identification

- Each file transfer MUST be identified by a `transferId` (bytes16, hex-encoded, 32 chars)
- Sender generates `transferId` via `crypto.getRandomValues(new Uint8Array(16))`
- `transferId` MUST be included in every `file-chunk`, `pause`, `resume`, and `cancel` message

### Receiver Guards (Guarded Mode)

Receiver tracks state per `transferId`. When `transferId` is present on incoming chunks:

| Check | Action on Violation |
|-------|-------------------|
| `totalChunks` not a finite positive integer | Reject, log `[REPLAY_OOB]` |
| `chunkIndex < 0` or `chunkIndex >= totalChunks` | Reject, log `[REPLAY_OOB]` |
| `chunkIndex` already received for this `transferId` | Ignore duplicate, log `[REPLAY_DUP]` |
| Same `transferId` bound to different sender identity key | Ignore, log `[REPLAY_XFER_MISMATCH]` |
| Different `transferId` | New transfer (no mismatch) |

Duplicate chunks do NOT abort the transfer (ignore-and-continue policy).

### Backward Compatibility (Legacy Mode)

- `transferId` is OPTIONAL at the wire level for backward compatibility
- When absent, receiver operates in legacy mode (no dedup, bounds checks still applied)
- Legacy mode logs: `[REPLAY_UNGUARDED] chunk received without transferId`
- Legacy chunks MUST NOT create or mutate guarded transfer state
- Future versions MAY make `transferId` mandatory (fail-closed)

---

## 13. File Integrity Verification

### Capability Gate

File integrity verification is gated by the `bolt.file-hash` capability. Both peers MUST advertise `bolt.file-hash` in their HELLO `capabilities` array for verification to be active. If either peer does not advertise the capability, file transfers proceed without hash verification (backward compatible).

### Sender Behavior

When `bolt.file-hash` is negotiated:

- Sender computes `SHA-256(file)` once before chunking, producing a 64-character hex string (`fileHash`)
- Sender includes `fileHash` on the **first chunk** (`chunkIndex === 0`) of the transfer
- Subsequent chunks do NOT include `fileHash` (bandwidth optimization)

### Wire Format

First chunk with `fileHash`:

```json
{
  "type": "file-chunk",
  "filename": "example.pdf",
  "transferId": "<hex, 32 chars>",
  "chunkIndex": 0,
  "totalChunks": 42,
  "fileSize": 688128,
  "fileHash": "<hex, 64 chars>",
  "chunk": "<base64, encrypted chunk>"
}
```

### Receiver Behavior

When `bolt.file-hash` is negotiated AND `fileHash` was present on the first chunk:

1. Receiver stores `expectedHash` per `transferId` in guarded transfer state
2. After full reassembly, receiver computes `SHA-256(assembled_blob)`
3. If `actual !== expectedHash`:
   - Log `[INTEGRITY_MISMATCH]` with expected and actual hashes
   - Emit `IntegrityError` via `onError`
   - Send error control message: `{ "type": "error", "code": "INTEGRITY_FAILED", "message": "..." }`
   - Call `disconnect()` (fail-closed)
   - Do NOT call `onReceiveFile`
4. If hashes match: log `[INTEGRITY_OK]`, proceed with `onReceiveFile`

### Backward Compatibility

- If `bolt.file-hash` is NOT negotiated: no hash computation, no verification, existing behavior preserved
- If `fileHash` field is absent on first chunk (legacy sender): no verification for that transfer
- `fileHash` is OPTIONAL at the wire level; its presence is gated by capability negotiation
- `IntegrityError` is a new error type in `@the9ines/bolt-core` (extends `BoltError`)

---

## 14. Profile Envelope v1

### Capability Gate

Profile Envelope v1 is gated by the `bolt.profile-envelope-v1` capability. Both peers MUST advertise this capability in their HELLO `capabilities` array for envelope wrapping to be active. If either peer does not advertise the capability, all DataChannel messages are sent as legacy plaintext JSON (backward compatible).

### Outer Wire Format

When `bolt.profile-envelope-v1` is negotiated, all post-handshake DataChannel messages (file chunks, control messages, error messages) are wrapped in an encrypted envelope:

```json
{
  "type": "profile-envelope",
  "version": 1,
  "encoding": "base64",
  "payload": "<base64 string from sealBoxPayload>"
}
```

- `version` is fixed to `1` for this specification version
- `encoding` is self-documenting metadata, fixed to `"base64"` for v1
- `payload` is the return value of `sealBoxPayload(innerBytes, remotePublicKey, senderSecretKey)` — a base64 string. Implementations MUST NOT double-encode this value.

### Crypto Usage

- **Encrypt**: `payload = sealBoxPayload(UTF8Encode(JSON.stringify(innerMsg)), remotePublicKey, senderSecretKey)`
- **Decrypt**: `innerMsg = JSON.parse(UTF8Decode(openBoxPayload(payload, senderPublicKey, receiverSecretKey)))`
- Uses the same ephemeral key pair established during WebRTC signaling
- No additional key derivation or nonce management beyond what `sealBoxPayload`/`openBoxPayload` provide internally

### Receiver Behavior

1. **HELLO always plaintext**: HELLO messages are always sent and received as plaintext (they use their own encryption layer). Profile envelope MUST NOT wrap HELLO.
2. **Pre-handshake gating**: A `profile-envelope` received before `helloComplete` is treated as a non-HELLO message and triggers `INVALID_STATE` + disconnect (Phase 8D rule).
3. **Envelope unwrapping**: After handshake, if `msg.type === 'profile-envelope'`:
   - If NOT negotiated: log `[ENVELOPE_UNNEGOTIATED]`, send error, disconnect (fail-closed)
   - If negotiated but `version !== 1` or `encoding !== 'base64'`: log `[ENVELOPE_INVALID]`, send error, disconnect
   - If decryption fails: log `[ENVELOPE_DECRYPT_FAIL]`, send error, disconnect
   - On success: route decrypted inner message through normal message handling
4. **Legacy plaintext accepted**: Even when envelope is negotiated, the receiver MUST still accept legacy plaintext messages (mixed-peer compatibility). This ensures graceful degradation when one peer upgrades before the other.

### Error Codes

| Code | Condition | Action |
|------|-----------|--------|
| `ENVELOPE_UNNEGOTIATED` | Envelope received but capability not negotiated | Log, send error, disconnect |
| `ENVELOPE_INVALID` | Invalid version or encoding | Log, send error, disconnect |
| `ENVELOPE_DECRYPT_FAIL` | Decryption or JSON parse failure | Log, send error, disconnect |

### Security Properties

- **Metadata protection**: File names, sizes, transfer IDs, and control flags are no longer visible as plaintext JSON on the DataChannel. An intermediary observing the DataChannel would see only `type: "profile-envelope"` and an opaque encrypted payload.
- **Defense-in-depth**: This is a Profile-layer measure. It does not replace the Bolt Core envelope (which operates at the protocol level and is not yet implemented). It provides immediate value for the WebRTC DataChannel transport.
- **Forward compatibility**: Future relay/intermediary scenarios benefit from metadata being encrypted at the profile layer, reducing the trust surface of relay nodes.

### Explicit Non-Goals

- This is NOT the global Bolt envelope from PROTOCOL.md section 6. That operates at the protocol layer with its own nonce/key management.
- This does NOT encrypt rendezvous-level signaling (offer/answer/ICE). Those operate over WebSocket, outside the DataChannel.
- This does NOT change the HELLO encryption mechanism, which has its own envelope format.
