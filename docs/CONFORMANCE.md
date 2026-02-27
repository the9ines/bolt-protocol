# Bolt Protocol Conformance Matrix

**Canonical Location:** `bolt-protocol/docs/CONFORMANCE.md`
**Status:** Normative
**Created:** 2026-02-26
**Phase:** CONFORMANCE-2R

---

## Purpose

This document maps top-level specification sections to implementation locations
and evidence of conformance. Spec changes require explicit conformance review —
any modification to PROTOCOL.md or LOCALBOLT_PROFILE.md must be accompanied by
an update to this matrix.

A CI guard in this repository enforces this discipline on pull requests and
validates minimum coverage requirements.

---

## Scope

- Maps top-level sections only (§1–§14, Appendices for PROTOCOL.md; §1–§14 for LOCALBOLT_PROFILE.md).
- Does NOT map every subsection.
- Status reflects current review state, not exhaustive verification.

### Status Values

- **IMPLEMENTED** — Implementation exists and is believed conformant.
- **PARTIAL** — Implementation exists but is incomplete or has known gaps.
- **TODO** — Implementation status has not been assessed.
- **NOT APPLICABLE** — Section is informational or has no implementation target.

---

## Coverage Requirements (CONFORMANCE-2R)

### Minimum Coverage

| Spec | Total Rows | Minimum with Evidence (N) |
|------|-----------|--------------------------|
| PROTOCOL.md | 23 | 10 |
| LOCALBOLT_PROFILE.md | 14 | 5 |

A row **meets coverage** when:
1. Status is not TODO.
2. At least one Evidence entry is present.

Rows marked **NOT APPLICABLE** are exempt from the minimum count and do not contribute to N.

### Evidence Format

- Entries MUST be backtick-wrapped workspace-relative paths.
- Multiple entries separated by comma + space.
- No pipe characters (`|`) inside Evidence.
- No prose — paths only.
- Example: `bolt-core-sdk/ts/bolt-core/__tests__/identity.test.ts`, `bolt-daemon/src/ipc/trust.rs`

### CI Enforcement

A CI guard validates:
1. Spec changes require conformance update (CONFORMANCE-1R).
2. Conformance matrix must meet minimum coverage counts (CONFORMANCE-2R).

---

## PROTOCOL.md Conformance

| Spec Section | Summary | bolt-core-sdk Location | bolt-daemon Location | Status | Evidence |
|---|---|---|---|---|---|
| §1 Overview | Protocol scope, wire model, profile requirements | N/A (informational) | N/A | NOT APPLICABLE | |
| §2 Identity | Persistent X25519 identity, TOFU, peer code | TS: `bolt-core/src/identity.ts`, `bolt-core/src/peer-code.ts`; Rust: `src/identity.rs`, `src/peer_code.rs` | `src/session.rs` (identity in SessionContext), `src/ipc/trust.rs` (pairing) | IMPLEMENTED | `bolt-core-sdk/ts/bolt-core/__tests__/identity.test.ts`, `bolt-core-sdk/ts/bolt-core/__tests__/peer-code.test.ts`, `bolt-daemon/src/ipc/trust.rs` |
| §3 Session Establishment | Ephemeral keypairs, HELLO bootstrap, handshake gating, SAS | TS: `bolt-core/src/crypto.ts`, `bolt-core/src/sas.ts`, `bolt-transport-web/src/services/webrtc/WebRTCService.ts`; Rust: `src/crypto.rs`, `src/sas.rs` | `src/web_hello.rs` (HELLO exchange), `src/session.rs` (session context) | IMPLEMENTED | `bolt-core-sdk/ts/bolt-core/__tests__/hello-open-vectors.test.ts`, `bolt-daemon/tests/h3_golden_vectors.rs` |
| §4 Version and Capability Negotiation | HELLO schema, capability strings, encoding negotiation | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (HELLO handler); Rust: N/A at core level | `src/web_hello.rs` (capability intersection) | IMPLEMENTED | `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/capabilities.test.ts`, `bolt-daemon/tests/h5_downgrade_validation.rs` |
| §5 Connection Approval | CONNECTION_REQUEST/ACCEPTED/DECLINED via rendezvous | TS: `bolt-transport-web/src/services/signaling/` | `src/ipc/trust.rs` (pairing approval), `src/ipc/types.rs` | IMPLEMENTED | `bolt-daemon/src/ipc/trust.rs`, `bolt-daemon/scripts/e2e_rendezvous_local.sh` |
| §6 Bolt Message Protection | Encrypted envelope schema, encryption/decryption rules | TS: `bolt-core/src/crypto.ts` (seal/open), `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (envelope mode); Rust: `src/crypto.rs` | `src/envelope.rs` (Profile Envelope v1 codec) | IMPLEMENTED | `bolt-core-sdk/rust/bolt-core/tests/conformance/envelope_validation.rs`, `bolt-core-sdk/ts/bolt-core/__tests__/nonce-uniqueness.test.ts` |
| §7 Message Model | Canonical message schemas (HELLO through ERROR, PING/PONG) | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (message types) | `src/dc_messages.rs` (inner message types), `src/envelope.rs` (routing) | IMPLEMENTED | `bolt-daemon/src/dc_messages.rs`, `bolt-core-sdk/ts/bolt-core/__tests__/vectors.test.ts` |
| §8 File Transfer | Chunking, backpressure, integrity, replay protection, limits | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (transfer logic); Rust: `src/transfer_policy/` (scheduling stub) | N/A (daemon does not handle file transfer) | PARTIAL | `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/replay-protection.test.ts`, `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/file-hash.test.ts` |
| §9 State Machines | Signaling, peer connection, and transfer state machines | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts`, `bolt-transport-web/src/state/store.ts` | `src/main.rs` (connection state), `src/rendezvous.rs` (signaling state) | PARTIAL | `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/webrtcservice-lifecycle.test.ts`, `bolt-daemon/tests/h5_downgrade_validation.rs` |
| §10 Error Taxonomy | 11 protocol error codes | TS: `bolt-core/src/errors.ts`; Rust: `src/errors.rs` | `src/envelope.rs` (EnvelopeError, canonical registry) | IMPLEMENTED | `bolt-core-sdk/rust/bolt-core/tests/conformance/error_code_mapping.rs`, `bolt-daemon/tests/h5_downgrade_validation.rs` |
| §11 Security Properties | Summary of security guarantees | Structural (crypto + envelope + SAS modules) | Structural (envelope + hello + session modules) | TODO | |
| §12 Threat Model | Rendezvous untrusted, MITM detection, traffic analysis non-goal | N/A (informational) | N/A | NOT APPLICABLE | |
| §13 Conformance | MUST/SHOULD/MAY requirements list | Test suites: S1 conformance harness (Rust), H2 enforcement tests (TS) | H3 golden vectors, H5 downgrade validation | PARTIAL | `bolt-core-sdk/rust/bolt-core/tests/conformance/main.rs`, `bolt-daemon/tests/h3_golden_vectors.rs` |
| §14 Constants | Protocol constants table | TS: `bolt-core/src/constants.ts`; Rust: `src/constants.rs` | N/A (uses bolt-core crate) | IMPLEMENTED | `bolt-core-sdk/ts/bolt-core/__tests__/exports.test.ts`, `bolt-core-sdk/scripts/verify-constants.sh` |
| §15 Handshake Invariants | Keying model, key binding, error registry, envelope requirement, HELLO state machine | See §15 subsection rows below | See §15 subsection rows below | See below | |
| §15.1 Ephemeral-first keying | Canonical HELLO keying model (PROTO-HARDEN-01, 02) | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (HELLO send/receive); Rust: `src/crypto.rs` | `src/web_hello.rs` (HELLO exchange) | TODO | |
| §15.2 Identity-ephemeral binding | Cryptographic binding via envelope MAC + SAS (PROTO-HARDEN-01, 02) | TS: `bolt-core/src/sas.ts`, `bolt-core/src/crypto.ts`; Rust: `src/sas.rs`, `src/crypto.rs` | `src/web_hello.rs`, `src/session.rs` | TODO | |
| §15.3 Error registry invariants | Single canonical registry, cross-impl parity (PROTO-HARDEN-03, 04, 05) | TS: `bolt-core/src/errors.ts`; Rust: `src/errors.rs` | `src/envelope.rs` (EnvelopeError, CANONICAL_ERROR_CODES) | TODO | |
| §15.4 Post-handshake envelope | No plaintext errors in envelope-required mode (PROTO-HARDEN-06, 07) | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (envelope mode) | `src/envelope.rs` (build_error_payload) | TODO | |
| §15.5 HELLO state machine | No reentrancy, exactly-once, immutable capabilities (PROTO-HARDEN-08–12) | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (helloComplete guard) | `src/web_hello.rs` (HelloState) | TODO | |
| Appendix A | Profile system | N/A (informational) | N/A | NOT APPLICABLE | |
| Appendix B | Key rotation (out of scope for v1) | N/A | N/A | NOT APPLICABLE | |
| Appendix C | Conformance tests | S1 harness (`rust/bolt-core/tests/conformance/`), H2 tests, H3 vectors | H3 golden vectors, H5 downgrade tests | PARTIAL | `bolt-core-sdk/rust/bolt-core/tests/conformance/main.rs`, `bolt-daemon/tests/h5_downgrade_validation.rs` |

---

## LOCALBOLT_PROFILE.md Conformance

| Spec Section | Summary | bolt-core-sdk Location | bolt-daemon Location | Status | Evidence |
|---|---|---|---|---|---|
| §1 Overview | Profile scope, implements Bolt Core v1 | N/A (informational) | N/A | NOT APPLICABLE | |
| §2 Rendezvous Transport | WebSocket signaling, cloud discovery extension | TS: `bolt-transport-web/src/services/signaling/WebSocketSignaling.ts` | `src/rendezvous.rs` (WebSocket client) | IMPLEMENTED | `bolt-daemon/src/rendezvous.rs`, `bolt-daemon/scripts/e2e_rendezvous_local.sh` |
| §3 Rendezvous Wire Format | Client/server JSON messages, peer code validation | TS: `bolt-transport-web/src/services/signaling/SignalingProvider.ts` | `src/web_signal.rs` (signal payloads) | IMPLEMENTED | `localbolt-v3/packages/localbolt-signal/src/lib.rs`, `bolt-daemon/src/web_signal.rs` |
| §4 Peer Channel | DataChannel label, ordered, reliable, binaryType | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | `src/main.rs` (DataChannel setup via libdatachannel) | IMPLEMENTED | `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/webrtcservice-lifecycle.test.ts` |
| §5 Connectivity Policy | STUN servers, relay candidate blocking | TS: `bolt-transport-web/src/lib/platform-utils.ts` (ICE config) | `src/ice_filter.rs` (candidate filtering by scope) | IMPLEMENTED | `bolt-daemon/src/ice_filter.rs` |
| §6 Message Encoding | json-envelope-v1 wire format, field encoding, examples | TS: `bolt-core/src/encoding.ts`, `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | `src/envelope.rs` (Profile Envelope v1), `src/web_hello.rs` (HELLO JSON) | IMPLEMENTED | `bolt-daemon/src/envelope.rs`, `bolt-core-sdk/ts/bolt-core/__tests__/envelope-open-vectors.test.ts` |
| §7 Local Scope Policy | IP-based room grouping, private IP recognition | TS: `bolt-transport-web/src/lib/platform-utils.ts` (isPrivateIP) | `src/ice_filter.rs` (NetworkScope, RFC 1918/4193) | IMPLEMENTED | `bolt-daemon/src/ice_filter.rs`, `localbolt-v3/packages/localbolt-signal/src/lib.rs` |
| §8 Resource Limits | Max file size, chunk count, concurrent transfers | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | N/A (daemon does not handle file transfer) | PARTIAL | `localbolt-v3/packages/localbolt-signal/src/lib.rs` |
| §9 Reconnection | Exponential backoff, keepalive interval | TS: `bolt-transport-web/src/services/signaling/WebSocketSignaling.ts` | `src/rendezvous.rs` | TODO | |
| §10 Key Exchange Binding | Envelope ephemeral key is authoritative | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | `src/envelope.rs` (sender key from envelope header) | IMPLEMENTED | `bolt-daemon/src/envelope.rs`, `bolt-core-sdk/ts/bolt-core/__tests__/hello-open-vectors.test.ts` |
| §11 Platform-Specific Chunk Sizes | Device-class chunk size table | TS: `bolt-transport-web/src/lib/platform-utils.ts` (getMaxChunkSize); Rust: `src/transfer_policy/types.rs` (DeviceClass) | N/A | IMPLEMENTED | `bolt-core-sdk/rust/bolt-core/tests/s2_policy_contracts.rs` |
| §12 Replay Protection | transfer_id scoping, receiver guards, legacy mode | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` | N/A (daemon does not handle file transfer) | PARTIAL | `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/replay-protection.test.ts` |
| §13 File Integrity Verification | bolt.file-hash capability, SHA-256 verification | TS: `bolt-core/src/hash.ts`, `bolt-transport-web/src/services/webrtc/WebRTCService.ts`; Rust: `src/hash.rs` | N/A | PARTIAL | `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/file-hash.test.ts` |
| §14 Profile Envelope v1 | Capability-gated envelope wrapping, error codes | TS: `bolt-transport-web/src/services/webrtc/WebRTCService.ts` (envelope mode) | `src/envelope.rs` (full codec + error handling) | IMPLEMENTED | `bolt-daemon/src/envelope.rs`, `bolt-core-sdk/ts/bolt-transport-web/src/__tests__/profile-envelope-v1.test.ts` |

---

## Notes

- **Citation convention:** References like "PROTOCOL §6" cite the spec section. The authoritative text is always `bolt-protocol/PROTOCOL.md`.
- **bolt-daemon file transfer:** The daemon does not implement file transfer (§8, §12, §13). These sections are enforced only in the TS SDK transport layer.
- **Status is point-in-time:** This matrix reflects the state at creation. It must be updated when spec sections change.
- **Changing any spec section requires updating this matrix entry.**
- **Evidence convention:** Paths are workspace-relative (from bolt-ecosystem root). Inline Rust tests (e.g., `bolt-daemon/src/envelope.rs`) reference `#[cfg(test)]` modules within the source file.
