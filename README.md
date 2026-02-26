# Bolt Protocol

Normative specification for the Bolt encrypted device-to-device file transfer protocol.

## What This Is

This repository contains the **specification only**. No implementation code lives here.

- **PROTOCOL.md** — Bolt Core v1. Transport-agnostic protocol specification.
- **LOCALBOLT_PROFILE.md** — LocalBolt Profile v1. Transport binding for the browser peer channel.

## Role in the Ecosystem

Bolt Protocol is the trust layer. It defines identity, pairing, encrypted envelopes, message semantics, state machines, and conformance rules independently of any transport.

All implementations depend on this specification for correctness.

## Dependencies

None. This is the root specification.

## Depends On This

- [bolt-core-sdk](https://github.com/the9ines/bolt-core-sdk) — Reference implementation
- [localbolt](https://github.com/the9ines/localbolt) — Open-source lite app
- [localbolt-app](https://github.com/the9ines/localbolt-app) — Open-source native app
- [localbolt-v3](https://github.com/the9ines/localbolt-v3) — Web application

## Conformance

- [docs/CONFORMANCE.md](docs/CONFORMANCE.md) — Spec-to-implementation conformance matrix.

## Status

| Document | Version | Status |
|----------|---------|--------|
| Bolt Core v1 | 1.0.0 | Draft |
| LocalBolt Profile v1 | 1.0.0 | Draft |

## License

MIT
