# OSAP-Link: multi-transport delivery for OSAP (proposed extension)

- Status: **proposed transport extension, non-normative.** This is not part of OSAP draft-01 and does not change its wire format, keys, envelopes, signatures, grants or approvals.
- Source: Privaku, *OSAP-Link Requirements & Implementation Plan v1.0* (28 Oct 2026, design baseline).
- Nothing here has been implemented or benchmarked yet. Every number below is a **TARGET**, not a measurement.

## 1. Why

OSAP draft-01 delivers messages through a relay over HTTPS. That works anywhere, but:

| Problem | What OSAP-Link adds |
|---|---|
| A phone and an agent in the same room still round-trip through the internet | **Fast local delivery** over direct LAN |
| The internet or relay goes down, so local agents and devices stop talking | **Works through internet outages**: direct LAN/USB keeps going, with relay as fallback |
| Pairing a nearby device means typing or scanning on every device | **Nearby pairing** aids (QR, BLE, sound) that bootstrap the normal OSAP pairing |
| Some devices (microcontrollers, machine gateways) have no network | **Devices without a network** reach the agent host over USB serial |
| Smart behaviour often assumes a big cloud AI backend | **No big cloud AI backend**: delivery and inference stay local by default |

OSAP-Link is a transport adapter and reliability layer. OSAP still defines who may send what, how it is signed and encrypted, and what is permitted. OSAP-Link only chooses *how the bytes get there*.

## 2. Layers (never conflated)

- **OSAP core**: identity, E2E envelopes, grants, approvals, signed command meaning.
- **OSAP-Link**: discovery, session liveness, framing, delivery status, failover.
- **Device controller**: hardware interfaces (camera/HDMI/MQTT/Matter) and safety interlocks.

## 3. Modes

| Transport | Role | Priority | Key limitation |
|---|---|---|---|
| LAN / Ethernet, TLS WebSocket | Normal fast route | P0 | Wi-Fi quality, LAN isolation, firewalls |
| OSAP relay over HTTPS | Remote / NAT fallback (today's path) | P0 | Internet RTT, relay availability |
| USB serial, COBS framing | Tethered microcontroller / hardware gateway | P1 | Cabling, drivers, per-device framing |
| Bluetooth LE | Nearby bootstrap, very small messages | P1 | iOS background limits, small payloads |
| Sound (e.g. ggwave), audible / near-ultrasonic | Pairing challenge, proximity hint | P1 (experiment) | Noise, mic permission, very low throughput |
| QR / light | Visual pairing, tiny offline proof | P2 | Camera framing, lighting |
| WebRTC DataChannel / QUIC | Optional optimised direct path | P2 (only if benchmarks show need) | Setup complexity, platform support |

## 4. How fast (TARGETS, not measured)

OSAP-Link must never claim "zero latency". These are lab acceptance goals to be measured:

| Metric | TARGET | Test definition |
|---|---|---|
| LAN command delivery p95 | **< 100 ms** (target) | Envelope send → authenticated local receipt, idle LAN, 1000 messages |
| LAN event alert p95 | **< 250 ms** (target) | Agent event → visible UI update, app in foreground |
| Relay event delivery p95 | **< 2 s** (target) | Both peers online, stable internet; excludes APNs wake-up delay |
| Direct-link reconnect p95 | **< 3 s** (target) | Recover after short Wi-Fi drop, session and peer still valid |
| Duplicate executions | **0** (target) | cmdId idempotency across the test set |
| Stale high-risk commands executed | **0** (target) | Outside approval/grant/expiry, under all route conditions |
| Sound (ggwave) throughput | **about 8–16 bytes/s** (published ggwave figure, not measured by Privaku) | Pairing challenges only; never messaging, images or control |

Push delivery, mobile suspension, cold-start inference, public networks and noisy rooms are not held to the LAN targets and are tested separately. Results will be published as measured p50/p95/p99. Goals are never reported as achieved.

## 5. Route policy

1. Prefer an **authorized direct LAN** link.
2. Switch to the **relay** if direct is unavailable or degraded.
3. Never use sound or light for routine encrypted messaging.
4. An expired command fails; it is not retried forever.
5. A route change keeps the same message ID and recipient identity. The same message over LAN and relay is one logical delivery with an idempotent effect.

## 6. Frame rules (proposed profile; canonical encoding TBD)

```
linkVersion : 1             packetType : DATA | ACK | HELLO
sessionId   : random        messageId  : unique random ID
fragment    : index/total   (only when needed)
deadline    : optional bounded transport TTL
payload     : complete OSAP envelope (opaque) or a fragment of it
checksum    : transport integrity / truncation detection
```

- The payload is an OSAP draft-01 envelope (`hdr/iv/ct/sig`), **unchanged**. Unknown `hdr.v` and unknown `linkVersion` are rejected.
- The Mac prototype uses JSON over WebSocket. CBOR is used only if it measurably helps constrained devices. A byte-exact canonical form is required before any interoperability claim.
- Frame and reassembly sizes are bounded. Malformed lengths, excessive fragments and decompression bombs are rejected. Memory is never allocated unboundedly from a received frame.
- Message IDs are random, with a bounded replay/dedupe store. Command IDs persist across restarts for any actuator.
- Connection TLS is extra protection, never a substitute for OSAP end-to-end envelopes.
- Version negotiation is explicit. Downgrade happens only if it is authorized and still secure.
- Discovery announces only a short-lived locator, the public protocol version and an ephemeral hint. It never exposes full identity, machine details or capability manifests. The peer is authenticated by its paired OSAP key fingerprint plus a signed challenge.

## 7. Security rule: a link never grants permission

- Identity is the OSAP key, never an IP/MAC address, name or BLE advertisement.
- Pairing stays QR/link plus an owner-verified passkey. A bootstrap on its own has no power.
- No ambient access comes through discovery or fallback. Per-agent, per-device grants apply on every route.
- High-risk actions still need fresh owner verification bound to the exact action hash (draft-01 §8). There is no LLM auto-approval.
- An **ACK is transport evidence only**. It is not authorization, execution or approval. Device "accepted" and "done" are separate.
- Relay-managed MCP connections are labelled as not agent-end E2E (draft-01 §12).
- OSAP-Link does not add forward secrecy and must not imply it (that is OSAP core v2 work).

### Sound and light are non-authorizing

- Hearing a tone or seeing a QR code proves only that *something* received it. That is not consent, authorization or identity.
- The user starts these exchanges deliberately and grants mic/camera permission. The mic is never used in the background, and there is a visible indicator.
- Payloads carry only an ephemeral random challenge or session locator. They never carry tokens, keys or biometrics.
- After the exchange, the OSAP fingerprint and signed challenge are verified. A replayed recording or the wrong device cannot enroll.
- Error detection, bounded retries and a hard timeout apply, with QR fallback. Payloads expire, are rate-limited, and copies are rejected.

## 8. Tests (in addition to the 55 draft-01 conformance cases)

| ID | Scenario | Pass condition |
|---|---|---|
| LINK-01 | LAN message, both endpoints authorized | Original OSAP envelope delivered, verified, decrypted |
| LINK-02 | Discovery impersonator | Rejected before any trusted operation |
| LINK-03 | Network drop mid-fragment | No partial message reaches the application |
| LINK-04 | Same message retried over LAN and relay | One logical delivery; idempotent effect |
| LINK-05 | Expired command arrives after reconnect | Rejected; owner sees expired state |
| LINK-06 | Grant revoked during an active link | Next command blocked by updated authority/lease |
| LINK-07 | Audio replay from a recording | Cannot authorize pairing or hardware operation |
| LINK-08 | Audio fails in a noisy room | Explicit timeout, QR fallback |
| LINK-09 | Wrong fingerprint after optical/audio bootstrap | Pairing rejected with a security warning |
| LINK-10 | Malformed large/fragmented frame | Rejected with bounded memory |
| LINK-11 | User stops agent while offline | Safe stop at local executor/lease; no new work on expired lease |
| LINK-12 | High-risk message without approval | Never reaches the device driver |
| LINK-13 | Unknown frame or envelope version | Rejected per version policy |
| LINK-14 | Poisoned screen text says "ignore rules" | No permission escalation or hidden action |
| LINK-15 | Cold-start VLM uses too much memory | Bounded fallback to OCR/rules; no crash |

## 9. Roadmap

Build machine: **Mac mini**. Test peer: **MacBook Air M1** (8 GB), with an iPhone running Privaku Talk on an ordinary Wi-Fi router.

- **P0 (MVP blockers):**
  - Draft-01 identities and passkey pairing.
  - Opaque envelope carriage.
  - LAN discovery with fingerprint and an authenticated direct link.
  - Relay fallback with no double execution.
  - Honest sent/received/acknowledged/expired/failed states.
  - Expired, replayed and unauthorized commands rejected before hardware.
  - device.manifest/command/ack/state/event pass-through.
  - Route state and diagnostic logs.
  - A read-only screen-error alert demo delivered to Privaku Talk.
- **P1:**
  - Bounded BLE pairing/discovery.
  - An acoustic pairing challenge with signed visual confirmation.
  - USB serial (COBS) for a tethered microcontroller or gateway.
- **P2:**
  - Animated QR/light bootstrap.
  - A WebRTC/QUIC adapter if benchmarks justify it.
  - A supervised physical output adapter with an independent E-stop.

Release gate: OSAP-Link is not production-ready until the draft-01 baseline and LINK tests pass, pairing is reviewed, failure handling is reliable, and no unapproved high-risk command can reach an actuator.

## 10. Open questions

- **Push content vs conformance #22.** Draft-01 §10 / test #22 says wake-up and push bodies carry neither ciphertext nor plaintext. Privaku Talk currently puts an E2E envelope (ciphertext only, never plaintext) in the APNs payload so the phone can show the message, and tracks this as a documented exception. The spec should either allow ciphertext-only push explicitly or the implementation should change.
- **Offline revocation.** A relay-side disconnect is immediate for relayed traffic, but an offline device on a direct link cannot receive a revocation. The proposal is signed short-lived leases that fail closed on expiry, plus independent local safety controls. The protocol is not yet specified and is security-critical.
- Native direct-LAN wire format: JSON/TLS now; CBOR only after benchmarks.

## Licence

Specification text CC BY 4.0; code snippets Apache-2.0. See [LICENSE](LICENSE).
