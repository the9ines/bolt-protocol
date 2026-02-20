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
