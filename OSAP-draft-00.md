# OSAP: Open Secure Agent Protocol (draft-00)

- **Status:** Internet-Draft style working draft. Not a standard. Breaking changes expected.
- **Date:** 2026-10-28
- **Editor:** Privaku (hello@privaku.com)
- **Reference implementation:** Privaku Talk (relay `talk-relay/1`, CLI `talk-agent`, iOS client)
- **Licence:** text CC BY 4.0; reference code snippets Apache-2.0
- **Supersedes:** OASM draft 0.1 (`docs/OASM-draft.md`)

The key words MUST, MUST NOT, REQUIRED, SHALL, SHOULD, SHOULD NOT, RECOMMENDED, MAY and OPTIONAL are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals.

Each section marks what exists today: **[impl]** = implemented in the reference implementation as of draft-00; **[new]** = specified here, not yet implemented.

### Name options (open decision)
1. **OSAP** – Open Secure Agent Protocol (working name)
2. **OAMP** – Open Agent Messaging Protocol
3. **Privaku Agent Protocol (PAP)**
4. **AgentLink** / **OSAP-Link**
5. **OASM** – Open Agent Secure Messaging (the earlier internal name)

---

## 1. Introduction

OSAP lets a person (the **owner**) and their AI agents and devices exchange end-to-end encrypted, signed messages, ask for and give approvals, share files, and trigger actions on physical devices. The relay only ever sees public keys, passkey public credentials, routing metadata and ciphertext.

Goals: (a) the relay cannot read content; (b) every agent and device is uniquely identified and owner-controlled; (c) risky actions require fresh user verification; (d) works with existing agent ecosystems (MCP) and smart-home stacks (Home Assistant, Matter, MQTT).

Non-goals in draft-00: federation between relays, public directories, group chats with non-owners.

## 2. Terminology

- **Owner:** a human account, authenticated with WebAuthn passkeys.
- **Device:** an owner's client installation (phone, tablet, computer). Holds its own keys.
- **Agent:** a software principal (CLI agent, MCP client, bot) acting for one owner.
- **Device agent:** an agent that controls physical hardware (§11).
- **Relay:** the server that stores ciphertext and routes it (reference: a Cloudflare Worker plus one Durable Object).
- **Key bundle:** `{sign, dh:{"P-256", "X25519"?}}`, base64url raw public keys.
- **Fingerprint:** first 16 base64url characters of SHA-256 over the sorted bundles (`sign + "." + dh values joined by "."`, bundles joined by `|`).

## 3. Identities

### 3.1 Owner [impl]
- 3.1.1 An owner MUST authenticate with WebAuthn/FIDO2 passkeys. SMS OTP MUST NOT be used. Email OTP MAY be offered only as a recovery path behind a server-side kill switch and MUST NOT by itself authorise approvals.
- 3.1.2 The relay MUST assign an opaque, random `userId`. It MUST NOT be derived from an email or phone number.
- 3.1.3 Account deletion MUST remove the owner's devices, agents, inbox, blobs and grants.

### 3.2 Device keys [impl]
- 3.2.1 Each device MUST generate its own signing key (ECDSA P-256) and key-agreement keys (P-256, plus X25519 where supported) on the device.
- 3.2.2 Private keys MUST NOT leave the device except inside an encrypted backup (§3.5). They SHOULD be non-extractable or hardware-backed.

### 3.3 Stable agentId [impl, partly new]
- 3.3.1 Every agent MUST have its own key bundle and an `agentId` assigned at enrolment. [impl]
- 3.3.2 An agent MUST present a stable **agent identity key** across re-pairings. When an agent that is already enrolled for an owner pairs again with the same identity key (or a key signed by it), the relay MUST update the existing entry and MUST NOT create a second one. [new]
- 3.3.3 An owner MUST be able to connect any number of distinct agents; each MUST appear exactly once. [new]
- 3.3.4 Agent permissions are a set drawn from `chat`, `approvals`, `a2a`, `files`, `device:<capability>`. The reference implementation defines `chat` and `approvals`. [impl/new]

### 3.4 Multi-device [impl, extended]
- 3.4.1 An owner MAY have any number of devices. Senders MUST encrypt to every current device of the recipient (fan-out).
- 3.4.2 Adding a device MUST need either a passkey sign-in or a single-use pairing code from a signed-in device, expiring within 120 s and rate limited. [impl]
- 3.4.3 Any signed-in device MUST be able to list and revoke other devices; the relay MUST reject a revoked device immediately. [impl]
- 3.4.4 An agent MAY run on several machines sharing the same agent identity; it MUST still appear as one agent. [new]

### 3.5 History backup [impl]
Chat history MAY be backed up as ciphertext encrypted with a key derived from the passkey WebAuthn PRF extension, with iCloud Keychain or PIN-derived fallbacks. The relay MUST NOT receive the backup key.

## 4. Pairing

- 4.1 **QR / link [impl]:** agent calls `POST /api/agent/pair/start` with name, perms and its key bundle (signed with the new key). The owner scans the QR or opens the link, sees name, permissions and fingerprint (`GET` pair info), and approves with a user-verified passkey assertion (`POST /api/agent/pair/approve`). The agent long-polls `POST /api/agent/pair/status` (`wait` ≤ 20 s). Agent pairing codes MUST be single-use and MUST expire within 15 minutes.
- 4.2 **One-click connect [impl]:** `POST /api/agent/connect/start` creates a request the owner sees in-app with a Connect button; status via `/api/agent/connect/status`. Install links (e.g. "Add to Cursor") SHOULD resolve to this flow. Every flow MUST end in an owner passkey approval.
- 4.3 **Batch [impl]:** `approve-many` lets the owner approve several pending agents in one verification; each MUST be listed individually first.
- 4.4 **OAuth for MCP [impl]:** the relay acts as an OAuth 2.1 authorisation server with dynamic client registration (RFC 7591), PKCE, and metadata at `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource` (RFC 9728). `/oauth/authorize` shows a QR / Connect screen the owner approves in Talk; `/oauth/token` issues short-lived access tokens and refresh tokens, stored only as SHA-256 hashes. Scope: `talk`. The MCP endpoint is `/mcp`.
- 4.5 The fingerprint shown during pairing MUST be computed from the exact keys that will be enrolled.

## 5. Agent request authentication [impl]
- 5.1 Every agent API call MUST be signed with the agent's P-256 signing key over the canonical string `JSON.stringify({m, path, agentId, ts, nonce, payload})`.
- 5.2 The relay MUST reject requests with clock skew over 60 s and MUST reject a repeated `(agent, nonce)` within 120 s.
- 5.3 Nonces MUST be 8–64 characters.

## 6. Message envelope and encryption

### 6.1 Envelope [impl]
```json
{ "hdr": {"v":1,"alg":"X25519","from":"<deviceId>","to":"<deviceId>","eph":"<b64url>","ts":1761600000000},
  "iv": "<b64url 12 bytes>", "ct": "<b64url>", "sig": "<b64url ECDSA P-256>" }
```

### 6.2 Encryption (interim v1) [impl]
For each recipient device R the sender S:
1. MUST pick X25519 if both sides have it, otherwise P-256;
2. creates an ephemeral key E; `ikm = DH(E,R) || DH(S,R)`;
3. `key = HKDF-SHA256(ikm, salt = 32 zero bytes, info = "privaku-talk/v1|" + from + "|" + to)`;
4. encrypts the plaintext with AES-256-GCM, 96-bit random IV, AAD = JSON of `hdr`;
5. signs JSON of `{hdr, iv, ct}` with its device signing key.

Receivers MUST verify the signature before decrypting and MUST reject envelopes whose `hdr.to` is not their device or whose `v` is unknown.

### 6.3 Plaintext body [impl/new]
The decrypted plaintext is JSON with a `type`: `text`, `file` (§7), `approval.request`, `approval.result` (§8), `device.command`, `device.state`, `device.event` (§11), `receipt` [new]. Unknown types MUST be shown as "unsupported message" and MUST NOT be executed.

### 6.4 Roadmap (v2) [new]
v1 gives no forward secrecy against compromise of a recipient's static key. v2 SHOULD adopt MLS (RFC 9420) or a double ratchet, negotiated through `v` (§13).

## 7. Files and media [impl]
- 7.1 Files MUST be encrypted on the client with a fresh random AES-256-GCM key, in 512 KiB parts, each with a random IV and AAD = `blobId + ":" + partIndex`.
- 7.2 Upload: `POST /api/blob/new {size}` → `{id, part, parts, exp}`; `PUT /api/blob/<id>/<n>`; download: `GET /api/blob/<id>/<n>`.
- 7.3 The file key, name, size, MIME, kind (`photo|video|file`) and an optional encrypted thumbnail travel only inside the E2E message.
- 7.4 Clients MUST validate type by magic bytes against an allow-list before sending or opening.
- 7.5 Reference limits: 25 MB per file, 300 MB per owner, deleted after 14 days. Relays MAY choose other limits and MUST return 413 above them.
- 7.6 Agents SHOULD support sending and receiving files with the `files` permission. [new for agents]

## 8. Approvals [impl]
- 8.1 An agent with `approvals` MAY call `POST /api/agent/approval {title}` (MCP: `ask_approval`). The relay MUST show it to the owner and MUST NOT mark it approved without a fresh user-verified passkey assertion (Face ID / Touch ID / device PIN).
- 8.2 The result is `approved`, `denied` or `waiting`; agents long-poll `/api/agent/approval/status`.
- 8.3 Approvals MUST expire (RECOMMENDED: 10 minutes) and MUST be single-use.
- 8.4 [new] An approval SHOULD carry a hash of the exact action payload so the agent cannot swap the action after approval; executors MUST check it.

## 9. Agent-to-agent (opt-in) [partly impl]
- 9.1 Agents of the same owner MAY message each other (`/api/agent/peers`, `send` with `to`). [impl]
- 9.2 Agent-to-agent messaging MUST be off by default and MUST be enabled by the owner per pair of agents (`a2a` grant). [new]
- 9.3 The owner MUST be able to read every agent-to-agent conversation, and MUST be able to stop it instantly.
- 9.4 Agents MUST NOT relay approvals or device grants to each other (§11.9).
- 9.5 Agents of different owners MUST NOT talk in draft-00.

## 10. Presence, wake-up and push

### 10.1 Delivery [impl]
Agents receive via long-poll `POST /api/agent/inbox {wait ≤ 20}` and acknowledge with `/api/agent/ack`. Acknowledged items MUST be deleted from the relay.

### 10.2 Presence [new]
Agents MAY publish `online | idle | offline` with a `lastSeen` timestamp. Presence MUST NOT reveal content and SHOULD be coarse (minute resolution).

### 10.3 Wake-up webhooks [new]
- An agent MAY register an HTTPS webhook. On a new message the relay or a local listener MUST POST only `{agentId, count, ts}` with an HMAC-SHA256 signature header over the body, keyed by a per-agent secret. Webhooks MUST NOT include ciphertext or plaintext.
- Receivers MUST reject unsigned or stale (>5 min) calls. Relays MUST rate limit and coalesce wake-ups (RECOMMENDED: at most one per 10 s per agent).

### 10.4 Push [impl]
- Owner devices register APNs (or other platform) tokens. Push payloads MUST NOT contain message content; the reference uses a generic alert with `apns-collapse-id: new-message`.
- Clients SHOULD suppress banners while the app is in the foreground and SHOULD fetch new messages before showing a chat opened from a notification.

## 11. Device actions (hardware)

### 11.1 Device agent identity
- A device agent is an agent (§3.3) with `kind: "device"`. It MUST hold its own key bundle generated on the hub or device (Raspberry Pi, Home Assistant add-on, Matter controller, MQTT bridge) and MUST be paired through §4 with owner passkey approval.
- Its identity MUST stay stable across restarts and re-pairing (§3.3.2).

### 11.2 Capability manifest
On pairing and on change, a device agent MUST send a signed `device.manifest`:
```json
{ "type":"device.manifest", "manifestVersion":1,
  "devices":[
   {"id":"light.kitchen","class":"light","name":"Kitchen light","actions":["on","off","set_brightness"],"state":["on","brightness"],"risk":"low"},
   {"id":"lock.front","class":"lock","actions":["lock","unlock"],"state":["locked","battery"],"risk":"high"},
   {"id":"sensor.hall","class":"sensor","state":["temperature","motion"],"risk":"none"},
   {"id":"camera.door","class":"camera","actions":["snapshot","stream_start","stream_stop"],"risk":"medium"},
   {"id":"printer.3d","class":"printer","actions":["start","pause","cancel"],"risk":"high"},
   {"id":"robot.arm","class":"robot","actions":["move","home","stop"],"risk":"critical"} ] }
```
Classes: `light, switch, lock, sensor, camera, printer, robot, climate, cover, media, other`. Each action MUST declare a parameter schema (JSON Schema) and a risk level. Clients MUST show the manifest to the owner before granting.

### 11.3 Owner control (MUST)
- 11.3.1 The owner MUST always be able to connect or disconnect any device to or from any agent.
- 11.3.2 Access MUST be granted per agent and per capability (`device:<deviceId>:<action>` or a class wildcard), and every grant MUST be revocable instantly.
- 11.3.3 One-tap disconnect MUST work even while the agent is offline: the relay MUST stop routing commands immediately and the device agent MUST reject commands from a revoked grant as soon as it syncs the signed grant list.
- 11.3.4 No agent can grant itself or any other agent device access. Grants MUST be signed by an owner device after a user-verified passkey assertion; device agents MUST ignore grants not signed by an owner device.
- 11.3.5 Every grant change MUST be logged (who, what, when, from which device) in an append-only log visible to the owner.

### 11.4 Command and state messages
All are E2E encrypted envelopes (§6) with body types:
- `device.command`: `{type, cmdId, deviceId, action, params, grantId, approvalId?, issuedAt, expiresAt}`
- `device.ack`: `{type, cmdId, status:"accepted"|"rejected"|"done"|"failed", reason?}`
- `device.state`: `{type, deviceId, state:{…}, ts}`
- `device.event`: `{type, deviceId, event, data, ts}` (e.g. motion, door opened)

Device agents MUST reject commands that are unsigned, expired, replayed (`cmdId` seen), outside the grant, or with params failing the schema.

### 11.5 Approval levels
| Risk | Examples | Requirement |
|---|---|---|
| none | read sensor | grant only |
| low | lights, switches | grant only |
| medium | camera snapshot, thermostat | grant + owner notification; owner MAY require approval |
| high | unlock, open garage, start printer | MUST have a fresh Face ID approval (§8) bound to the exact command hash |
| critical | robot motion, heaters, anything that can injure | MUST have Face ID approval per command, MUST have an available `stop`, SHOULD require physical presence or a second factor |

Owners MAY raise any action's level but MUST NOT be able to lower `high`/`critical` below `high` through an agent.

### 11.6 Rate limits and safety interlocks
- Device agents MUST enforce per-device rate limits (RECOMMENDED: 10 commands/min low, 2/min high).
- `stop`/`off` safety actions MUST always be accepted from the owner and MUST NOT be rate limited.
- Device agents MUST honour local interlocks (e.g. door ajar, temperature limit, emergency stop, workspace occupied) and MUST refuse commands that violate them, regardless of grants.
- Local physical controls MUST keep working whatever the protocol state.

### 11.7 Offline queueing
- Commands MUST carry `expiresAt`. Relays MAY queue commands for offline device agents but MUST drop expired ones.
- `high` and `critical` commands MUST NOT be queued for longer than their approval lifetime (RECOMMENDED: 60 s); the owner MUST see "not delivered" rather than late execution.
- Revocations and owner `stop` MUST be delivered ahead of any queued command.

### 11.8 Bridges
- **Home Assistant:** a device agent MAY run as an HA add-on using the HA WebSocket API with a long-lived token stored only on the hub. Entities map to manifest devices (`domain.object_id` → `id`, domain → `class`).
- **Matter:** a device agent MAY act as a Matter controller on the owner's fabric. Clusters map to classes (OnOff → light/switch, DoorLock → lock, …). Matter commissioning MUST stay owner-initiated.
- **MQTT:** a device agent MAY bridge to an MQTT broker over TLS with per-bridge credentials. Topics SHOULD follow `osap/<deviceId>/cmd` and `osap/<deviceId>/state`. Brokers MUST NOT be exposed to the internet without auth.
- Bridges MUST translate only granted actions and MUST apply §11.5–11.7 before forwarding.

### 11.9 Agent mediation
A non-device agent with a `device:` grant sends `device.command` to the device agent. It MUST NOT forward grants, approvals or tokens to another agent. Device agents MUST check the grant is for the sending agent.

### 11.10 Audit
Device agents and clients SHOULD keep an owner-visible log of commands, approvals and results for at least 30 days, stored E2E encrypted.

## 12. MCP mapping [impl]
`/mcp` (Streamable HTTP, JSON-RPC) exposes tools `whoami`, `send_message`, `read_messages`, `ask_approval`. [new]: `send_file`, `list_devices`, `device_command`. MCP-connected agents hold server-side keys managed by the relay; their messages are therefore **not** end-to-end from the agent and clients MUST label them so.

## 13. Versioning
- Envelope `hdr.v` versions the crypto (current `1`). Body `type` values are versioned by suffix when they change incompatibly (`device.command/2`).
- Relays MUST expose `GET /api/version` and SHOULD expose `GET /.well-known/osap` listing supported versions and extensions. [new]
- Implementations MUST ignore unknown optional JSON fields and MUST reject unknown required versions.
- Spec drafts are numbered `draft-NN`; 1.0 freezes wire formats.

## 14. Security considerations
- **Relay trust:** the relay sees metadata (who talks to whom, sizes, times). It MUST store only what delivery needs, and SHOULD delete delivered items at once.
- **Key verification:** users SHOULD compare fingerprints; any change MUST clear the Verified state.
- **No forward secrecy in v1** (§6.4).
- **MCP agents** are not E2E from the agent (§12).
- **Replay:** signed requests with nonce and skew window (§5); commands with `cmdId` and expiry (§11.4).
- **Prompt injection:** agents MUST treat message content as untrusted data; physical actions MUST go through grants and approvals, never through text alone.
- **Approval fatigue:** clients SHOULD batch low-risk approvals and MUST show the exact action for high-risk ones.
- **Abuse limits:** relays MUST rate limit pairing, sign-in and uploads, and MAY degrade (503 with `retry-after`) near capacity.
- **Push and webhooks** MUST NOT carry content (§10).
- **Physical safety:** §11.6 interlocks override all protocol decisions.

## 15. Comparison
| | OSAP | MCP | A2A | Matrix | Signal |
|---|---|---|---|---|---|
| Purpose | owner ↔ agents/devices messaging | model ↔ tools | agent ↔ agent tasks | federated chat | private human chat |
| E2E encryption | yes (v1 interim, MLS planned) | no (TLS) | no (TLS) | yes (Olm/Megolm) | yes (Signal protocol) |
| Agent identity | per-agent keys, stable agentId | OAuth client | Agent Card | user account | n/a |
| Human approval | passkey/Face ID built in | client-specific | not defined | no | no |
| Hardware actions | first-class §11 | via tools | via tasks | bridges | no |
| Federation | no (draft-00) | n/a | yes (HTTP) | yes | no |

OSAP complements MCP (it ships an MCP endpoint) and could carry A2A tasks as message bodies.

## 16. Conformance tests
Implementations claim conformance per area. Each item MUST pass.

**Identity and pairing**
1. Pairing code reuse is rejected; expired code is rejected.
2. Pairing without owner passkey approval never enrols an agent.
3. Re-pairing an agent with the same identity key yields one entry, not two.
4. Two distinct agents yield two entries.
5. Revoked device is rejected on its next request.
6. A third and fourth device receive every new message.

**Requests and envelopes**
7. Bad agent signature → 401. 8. Skew > 60 s → rejected. 9. Repeated nonce → rejected.
10. Tampered `hdr`, `iv`, `ct` or `sig` fails to open. 11. Envelope for another device is rejected.
12. X25519 ↔ P-256 fallback round-trips.

**Files**
13. Relay storage contains no plaintext bytes. 14. Swapped part order fails (AAD). 15. Oversize → 413. 16. Disallowed magic bytes refused.

**Approvals**
17. No approval without a user-verified assertion. 18. Expired approval cannot be used. 19. Approval bound to command hash rejects a changed command.

**Agent-to-agent**
20. A2A disabled by default. 21. Owner revocation stops A2A immediately.

**Wake-up and push**
22. Webhook and push bodies contain no ciphertext or plaintext. 23. Unsigned or stale webhook rejected. 24. Wake-ups are coalesced.

**Device actions**
25. Manifest is shown before any grant. 26. Agent-signed (non-owner) grant is ignored. 27. Agent cannot grant itself or another agent access. 28. Revocation while device agent offline: relay stops routing at once; device rejects after sync. 29. Grant changes appear in the owner log. 30. `unlock` without fresh Face ID rejected. 31. `critical` action requires per-command approval. 32. Rate limit enforced; `stop` never limited. 33. Interlock blocks a granted, approved command. 34. Expired queued command not executed; high-risk not queued past 60 s. 35. Replayed `cmdId` rejected. 36. HA, Matter and MQTT bridges reject ungranted actions.

**Versioning**
37. Unknown optional field ignored; unknown `hdr.v` rejected.

## 17. IANA / registries
None yet. Future: `/.well-known/osap` URI suffix, body type registry, device class registry.

## Appendix A. Reference implementation map
| Area | Location (reference implementation) |
|---|---|
| Relay | `worker/index.js` (Workers + Durable Object), `host/app.js` |
| Crypto | `public/crypto.js` |
| Agent signing | `host/agent-proto.js` |
| Pairing | `host/pairing.js`, `host/agents.js` |
| Files | `public/files.js`, `/api/blob/*` |
| MCP / OAuth | `host/mcp.js` |
| Push | `host/extras.js` |
| CLI agent | `bin/talk-agent.mjs` (`pair`, `connect`, `send`, `listen`, `approve`, `whoami`) |
| iOS | `ios/PrivakuTalk/*` (WKWebView shell, native bridge) |

## Appendix B. Licence
Specification text: Creative Commons Attribution 4.0 International (CC BY 4.0). Code snippets and reference code: Apache License 2.0.
