# Bolt Protocol Conformance Matrix

**Canonical Location:** `bolt-protocol/docs/CONFORMANCE.md`
**Status:** Normative
**Created:** 2026-02-26
**Phase:** CONFORMANCE-1R

---

## Purpose

This document maps top-level specification sections to implementation locations.
Spec changes require explicit conformance review — any modification to PROTOCOL.md
or LOCALBOLT_PROFILE.md must be accompanied by an update to this matrix.

A CI guard in this repository enforces this discipline on pull requests.

---

## Scope

- Maps top-level sections only (§1–§14, Appendices for PROTOCOL.md; §1–§14 for LOCALBOLT_PROFILE.md).
- Does NOT map every subsection.
- Status reflects current review state, not exhaustive verification.

### Status Values

- **IMPLEMENTED** — Implementation exists and is believed conformant.
- **PARTIAL** — Implementation exists but is incomplete or has known gaps.
- **NOT REVIEWED** — Implementation status has not been assessed.
- **NOT APPLICABLE** — Section is informational or has no implementation target.

---

## PROTOCOL.md Conformance

| Spec Section | Summary | bolt-core-sdk Location | bolt-daemon Location | Status |
|--------------|---------|------------------------|----------------------|--------|
| §1 Overview | Protocol scope, wire model, profile requirements | N/A (informational) | N/A | NOT APPLICABLE |
| §2 Identity | Persistent X25519 identity, TOFU, peer code | TS: `bolt-core/src/identity.ts`, `bolt-core/src/peer-code.ts`; Rust: `src/identity.rs`, `src/peer_code.rs` | `src/session.rs` (identity in SessionContext), `src/ipc/trust.rs` (pairing) | IMPLEMENTED |
| §3 Session Establishment | Ephemeral keypairs, HELLO bootstrap, handshake gating, SAS | TS: `bolt-core/src/crypto.ts`, `bolt-core/src/sas.ts`, `bolt-transport-web/src/services/webrtc/WebRTCService.ts`; Rust: `src/crypto.rs`, `src/sas.rs` | `src/web_hello.rs` (HELLO exchange), `src/session.rs` (session context) | IMPLEMENTED |
| §4 Version and Capability Negotiation | HELLO schema, capability strings, encoding negotiation | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (HELLO handler); Rust: N/A at core level | `src/web_hello.rs` (capability intersection) | IMPLEMENTED |
| §5 Connection Approval | CONNECTION_REQUEST/ACCEPTED/DECLINED via rendezvous | TS: `bolt-transport-web/src/services/signaling/` | `src/ipc/trust.rs` (pairing approval), `src/ipc/types.rs` | IMPLEMENTED |
| §6 Bolt Message Protection | Encrypted envelope schema, encryption/decryption rules | TS: `bolt-core/src/crypto.ts` (seal/open), `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (envelope mode); Rust: `src/crypto.rs` | `src/envelope.rs` (Profile Envelope v1 codec) | IMPLEMENTED |
| §7 Message Model | Canonical message schemas (HELLO through ERROR, PING/PONG) | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (message types) | `src/dc_messages.rs` (inner message types), `src/envelope.rs` (routing) | IMPLEMENTED |
| §8 File Transfer | Chunking, backpressure, integrity, replay protection, limits | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (transfer logic); Rust: `src/transfer_policy/` (scheduling stub) | N/A (daemon does not handle file transfer) | PARTIAL |
| §9 State Machines | Signaling, peer connection, and transfer state machines | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts`, `bolt-transport-web/src/state/store.ts` | `src/main.rs` (connection state), `src/rendezvous.rs` (signaling state) | PARTIAL |
| §10 Error Taxonomy | 11 protocol error codes | TS: `bolt-core/src/errors.ts`; Rust: `src/errors.rs` | `src/envelope.rs` (EnvelopeError, canonical registry) | IMPLEMENTED |
| §11 Security Properties | Summary of security guarantees | Structural (crypto + envelope + SAS modules) | Structural (envelope + hello + session modules) | NOT REVIEWED |
| §12 Threat Model | Rendezvous untrusted, MITM detection, traffic analysis non-goal | N/A (informational) | N/A | NOT APPLICABLE |
| §13 Conformance | MUST/SHOULD/MAY requirements list | Test suites: S1 conformance harness (Rust), H2 enforcement tests (TS) | H3 golden vectors, H5 downgrade validation | PARTIAL |
| §14 Constants | Protocol constants table | TS: `bolt-core/src/constants.ts`; Rust: `src/constants.rs` | N/A (uses bolt-core crate) | IMPLEMENTED |
| Appendix A | Profile system | N/A (informational) | N/A | NOT APPLICABLE |
| Appendix B | Key rotation (out of scope for v1) | N/A | N/A | NOT APPLICABLE |
| Appendix C | Conformance tests | S1 harness (`rust/bolt-core/tests/conformance/`), H2 tests, H3 vectors | H3 golden vectors, H5 downgrade tests | PARTIAL |

---

## LOCALBOLT_PROFILE.md Conformance

| Spec Section | Summary | bolt-core-sdk Location | bolt-daemon Location | Status |
|--------------|---------|------------------------|----------------------|--------|
| §1 Overview | Profile scope, implements Bolt Core v1 | N/A (informational) | N/A | NOT APPLICABLE |
| §2 Rendezvous Transport | WebSocket signaling, cloud discovery extension | TS: `bolt-transport-web/src/services/signaling/WebSocketSignaling.ts` | `src/rendezvous.rs` (WebSocket client) | IMPLEMENTED |
| §3 Rendezvous Wire Format | Client/server JSON messages, peer code validation | TS: `bolt-transport-web/src/services/signaling/SignalingProvider.ts` | `src/web_signal.rs` (signal payloads) | IMPLEMENTED |
| §4 Peer Channel | DataChannel label, ordered, reliable, binaryType | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | `src/main.rs` (DataChannel setup via libdatachannel) | IMPLEMENTED |
| §5 Connectivity Policy | STUN servers, relay candidate blocking | TS: `bolt-transport-web/src/lib/platform-utils.ts` (ICE config) | `src/ice_filter.rs` (candidate filtering by scope) | IMPLEMENTED |
| §6 Message Encoding | json-envelope-v1 wire format, field encoding, examples | TS: `bolt-core/src/encoding.ts`, `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | `src/envelope.rs` (Profile Envelope v1), `src/web_hello.rs` (HELLO JSON) | IMPLEMENTED |
| §7 Local Scope Policy | IP-based room grouping, private IP recognition | TS: `bolt-transport-web/src/lib/platform-utils.ts` (isPrivateIP) | `src/ice_filter.rs` (NetworkScope, RFC 1918/4193) | IMPLEMENTED |
| §8 Resource Limits | Max file size, chunk count, concurrent transfers | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | N/A (daemon does not handle file transfer) | PARTIAL |
| §9 Reconnection | Exponential backoff, keepalive interval | TS: `bolt-transport-web/src/services/signaling/WebSocketSignaling.ts` | `src/rendezvous.rs` | NOT REVIEWED |
| §10 Key Exchange Binding | Envelope ephemeral key is authoritative | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | `src/envelope.rs` (sender key from envelope header) | IMPLEMENTED |
| §11 Platform-Specific Chunk Sizes | Device-class chunk size table | TS: `bolt-transport-web/src/lib/platform-utils.ts` (getMaxChunkSize); Rust: `src/transfer_policy/types.rs` (DeviceClass) | N/A | IMPLEMENTED |
| §12 Replay Protection | transfer_id scoping, receiver guards, legacy mode | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | N/A (daemon does not handle file transfer) | PARTIAL |
| §13 File Integrity Verification | bolt.file-hash capability, SHA-256 verification | TS: `bolt-core/src/hash.ts`, `bolt-transport-web/src/services/webrtc/WebRTCService.ts`; Rust: `src/hash.rs` | N/A | PARTIAL |
| §14 Profile Envelope v1 | Capability-gated envelope wrapping, error codes | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (envelope mode) | `src/envelope.rs` (full codec + error handling) | IMPLEMENTED |

---

## Notes

- **Citation convention:** References like "PROTOCOL §6" cite the spec section. The authoritative text is always `bolt-protocol/PROTOCOL.md`.
- **bolt-daemon file transfer:** The daemon does not implement file transfer (§8, §12, §13). These sections are enforced only in the TS SDK transport layer.
- **Status is point-in-time:** This matrix reflects the state at creation. It must be updated when spec sections change.
- **Changing any spec section requires updating this matrix entry.**
