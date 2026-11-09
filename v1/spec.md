# cheqd Trust Extensions for A2A — v1

> **DRAFT.** Not proposed to the A2A project. Has no maintainer sponsorship and is not official,
> experimental or recognised in any A2A process — see [A2A Extension and Protocol Binding
> Governance](https://a2a-protocol.org/latest/topics/extension-and-binding-governance/). This
> document describes behaviour that is already built and tested end-to-end (in both TypeScript and
> Python — see [`cheqd/agent-toolkit`](https://github.com/cheqd/agent-toolkit)), not a proposal
> for behaviour yet to be built. **Open items that still need resolving before this can be
> finalised or proposed anywhere are listed in [§7](#7-open-items--not-done).** Read that section
> before relying on anything here.

## 0. Status of this document

This specification was written by reading back the implementation in `cheqd/agent-toolkit`
(`packages/a2a/` and `python/packages/a2a/`), not the other way around. Every normative statement
below corresponds to code that has been run — against the real `@a2a-js/sdk` and `@kya-os/mcp` on
the TypeScript side, the real `a2a-sdk` on the Python side, and cross-language (TS client ↔ Python
server and the reverse) via `conformance/cross-language-a2a-wire.mjs` /
`python/packages/a2a/conformance_cross_language.py`, run on every push. Where this document and
the implementation diverge, the implementation is the bug to fix, not this document — but as of
this draft, no divergence is known.

Research and the evidence behind every design decision below live in
[`docs/research/a2a-trust-binding-findings-2026-11.md`](https://github.com/cheqd/agent-toolkit/blob/main/docs/research/a2a-trust-binding-findings-2026-11.md)
in `agent-toolkit`. This document is the normative distillation of those findings; that document
is the evidence trail.

## 1. Two independent extensions

| Extension | URI (this draft) | Owns |
|---|---|---|
| Request/response proof | `https://kya-os.org/a2a/ext/proof/v1` | Binding the DIF TAAWG `org.kya-os/proof.v1` holder-of-key profile to A2A's `SendMessage`/`SendStreamingMessage` envelope ([§4](#4-requestresponse-proof-extension)) |
| cheqd trust extension | `https://cheqd.io/a2a/ext/trust/v1` | Reciprocal `alsoKnownAs` linkage, DID-Linked Resource manifest resolution, and credential status, scoped to a caller already verified by §4 ([§5](#5-cheqd-trust-extension)) |

They compose but neither requires the other: a deployment can activate the proof extension alone
against any `did:web`-resolvable caller, with no cheqd involvement at all.

**URI note:** the repository's top-level README currently proposes `https://cheqd.io/a2a/ext/identity/v1`
for the second extension; the actual implementation in `agent-toolkit` uses
`https://cheqd.io/a2a/ext/trust/v1`. This document follows the implementation. The top-level
README needs updating to match, or the implementation needs renaming — tracked in [§7](#7-open-items--not-done).

## 2. Design: why a verifying `userBuilder`, not an `AgentExecutor` decorator

A request-proof check **MUST NOT** be implemented as an `AgentExecutor` decorator. `@a2a-js/sdk`'s
`DefaultRequestHandler` appends an incoming message to a task's history and persists it *before*
the executor runs; a throwing executor publishes a synthetic `FAILED` task rather than raising an
error. An unproofed message is written into task history either way.

The chosen shape is two cooperating pieces:

- A **verifying `userBuilder`** that reads the transport's wire body (never the SDK's typed
  objects — see [§4.2](#42-the-covered-form)), verifies the request proof, and returns either the
  verified DID as the SDK's `user`, or an unauthenticated user carrying a machine-readable
  rejection reason. It **MUST NOT** throw: a throwing `userBuilder` surfaces as an opaque HTTP 500,
  which defeats the point of a structured rejection.
- An **enforcing wrapper** around the request handler, transport-agnostic, that rejects any write
  without a verified identity before the inner handler runs, and — for `SendMessage` only — attaches
  the response proof over the handler's *final* result.

Consequence: failures surface as JSON-RPC errors under HTTP 200 (a missing proof and an invalid
proof both get application-level JSON-RPC error codes, not HTTP 401/403), because the SDK has no
authentication-specific error code of its own, and a throwing `userBuilder` would otherwise become
an HTTP 500. **HTTP-status-code monitoring alone will not see a rejection** — monitor the
JSON-RPC/REST error body instead. REST exposes rejections as an HTTP 400 (visible at the HTTP
layer, unlike JSON-RPC's HTTP 200).

Each transport needs its own `userBuilder` adapter: JSON-RPC and REST share one implementation
(the method is in the body for JSON-RPC, implied by the route for REST); gRPC's `UserBuilder` seam
reads the already-deserialised protobuf request off `call.request`, discriminated by `call.path`,
and needs the request run back through the SDK's own `SendMessageRequest.toJSON` before the
covered form is computed from it — hashing the deserialised protobuf object directly produces a
structurally different shape (numeric enums, `oneof`s as `{content: {$case, value}}`, default
fields present) than the wire JSON a JSON-RPC/REST client sent, and the two will not agree.

## 3. The response proof

The response proof, when made, **MUST** be computed over the transport's own JSON-serialised
result (`Task.toJSON(result)` on the TypeScript side; the equivalent `MessageToDict` on the Python
side) — never the SDK's in-memory typed object. The typed object carries explicit `undefined`
(TypeScript) values on unset optional fields that RFC 8785 JCS canonicalisation cannot handle.

The response proof **MUST** be made by the enforcing wrapper over the handler's *final* result,
never inside the executor. The SDK trims the executor's history according to the caller's
requested `historyLength` *after* the executor runs; a proof made inside the executor can describe
a result the client never actually receives.

The response proof is Ed25519-only in this draft (the reused `@kya-os/mcp` session-based
`ProofGenerator` does not support ES256 for this side, even though the request-proof path does).
The client **MUST** resolve and pin the server's public key itself before verifying a response
proof — the verifier takes a public JWK directly, with no DID-membership check of its own, and
that JWK **MUST** carry a `kid` or verification fails.

**Streaming carries no response proof, by construction.** The enforcing wrapper's streaming branch
(`sendMessageStream`) runs the same request-proof enforcement as `SendMessage`, but never attaches
a response proof — there is no single final result to prove over mid-stream. This is a known,
intentional gap, not an oversight; see [§7](#7-open-items--not-done) for what would need deciding
before closing it.

## 4. Request/response proof extension

Activation URI: `https://kya-os.org/a2a/ext/proof/v1`. Declared in the Agent Card's
`capabilities.extensions` with `required: true` for deployments that always demand a proof.

### 4.1 Scope

Applies to `SendMessage` and `SendStreamingMessage` only. Every other A2A method (task
cancellation, push-notification config, card fetch, …) passes through the enforcing wrapper
unchanged and unverified in this draft — a known gap tracked in [§7](#7-open-items--not-done) as
the "task-addressed-method identity gap."

### 4.2 The covered form

The request proof covers `{ method, params }`, where:

- `method` **MUST** be the wire method name (`SendMessage` / `SendStreamingMessage`) — never an
  alternate spelling such as `message/send`.
- `params` **MUST** be the transport's own canonical-JSON serialisation of the request (e.g.
  `SendMessageRequest.toJSON(...)` on the TypeScript side; the Python side's equivalent
  `MessageToDict` round trip), with `message.metadata[PROOF_META_KEY]` — and its legacy mirror,
  `org.kya-os/proof@1` — removed. If removing that key empties `message.metadata` entirely, the
  whole `metadata` field is removed too, so the covered form is identical whether or not an
  implementation ever had a metadata object to begin with.

The response proof's covered form is the final result's JSON-serialised form with
`metadata[RESPONSE_PROOF_META_KEY]` removed (and `metadata` removed entirely if that empties it),
by the same rule.

Metadata keys:

| Key | Carries |
|---|---|
| `org.kya-os/request-proof` | The request proof, in `message.metadata` |
| `org.kya-os/proof@1` | Legacy mirror of the above, stripped the same way, never re-added |
| `org.kya-os/response-proof` | The response proof, in the result's `metadata` |

### 4.3 Transport header

A client activating this extension **MUST** set the `A2A-Extensions` request header to this
extension's URI.

### 4.4 Request proof construction (client)

The client:

1. Serialises the outgoing `SendMessage`/`SendStreamingMessage` request to its wire JSON form via
   the transport SDK's own canonical serialiser (never a hand-built object standing in for the
   SDK's typed request — see the implementation note in `agent-toolkit`'s
   `conformance/cross-language-a2a-wire.mjs`, which found that a hand-rolled literal silently drops
   fields the SDK's real serialiser needs a union-typed internal shape to populate).
2. Computes the covered form ([§4.2](#42-the-covered-form)) from that wire JSON.
3. Builds a detached proof over the covered form per the `org.kya-os/proof.v1` profile (RFC 8785
   JCS canonicalisation, SHA-256 request binding, detached JWS, RFC 9421 HTTP Message Signature
   sibling, RFC 7638 `cnf` thumbprint fusion where used), with `audience` set to the server's DID
   and a fresh nonce.
4. Writes the proof into `message.metadata[org.kya-os/request-proof]` on the request it actually
   sends, and sets the `A2A-Extensions` header ([§4.3](#43-transport-header)).

### 4.5 Response proof verification (client)

The client **MUST** independently verify the response proof against the result as received — it
**MUST NOT** trust a server's bare claim that a response proof exists or validates. The client
**MUST** resolve and pin the server's public key itself before verifying; the verifier performs no
DID-membership check on the server's side of the exchange.

### 4.6 Server-side verification and enforcement

On receiving a `SendMessage`/`SendStreamingMessage` request, the verifying `userBuilder`
([§2](#2-design-why-a-verifying-userbuilder-not-an-agentexecutor-decorator)):

1. Reads the method and params from the transport's wire body (JSON-RPC: both from the body;
   REST: method from the route, params as the bare body; gRPC: both from the deserialised request,
   re-serialised per [§2](#2-design-why-a-verifying-userbuilder-not-an-agentexecutor-decorator)).
2. Returns an unauthenticated user for any method outside `SendMessage`/`SendStreamingMessage`.
3. Returns an unauthenticated user with rejection `missing` if no proof is present in
   `message.metadata[org.kya-os/request-proof]`.
4. Computes the covered form ([§4.2](#42-the-covered-form)).
5. Verifies the proof's signature against the resolved signer key.
6. Verifies the proof's `audience` matches the server's own DID.
7. Verifies the proof's nonce has not been consumed before (replay protection via a nonce cache;
   `@kya-os/mcp`'s in-memory cache is dev-only — production deployments should use a persistent,
   file- or store-backed cache, as `agent-toolkit`'s `sample/run.mjs` demonstrates with
   `FileNonceCacheProvider`).
8. Verifies `cnf` sender-constraint fusion, when present on the proof — **optional**; a proof
   verified without `cnf` is correctly accepted at a lower assurance level with a non-fatal
   warning, never silently promoted to the assurance level `cnf` would have given it, and never
   silently rejected for lacking it.
9. On any failure, returns an unauthenticated user carrying the specific rejection reason — never
   throws.
10. On success, if a [trust extension](#5-cheqd-trust-extension) is activated for this deployment,
    its per-DID checks are additionally consulted by the enforcing wrapper (not the `userBuilder`
    itself) before the request is allowed through — see [§5.2](#52-per-did-checks).

The enforcing wrapper then:

1. Rejects (fails closed) any request whose `userBuilder` did not return a verified identity —
   whether because the proof was missing, invalid, or no `userBuilder` was installed at all. A
   missing `userBuilder` **MUST** fail exactly like a missing proof, by design, not by accident of
   configuration.
2. Runs the inner handler only for a verified, (if applicable) trust-extension-passing caller.
3. For `SendMessage` only, attaches the response proof ([§3](#3-the-response-proof)) over the
   handler's final result before returning it.

### 4.7 Agent Card discovery

A server declaring this extension's `https://kya-os.org/a2a/ext/proof/v1` URI in its Agent Card
**MUST** have that card itself verifiable before a client trusts the declaration. Known constraints
on Agent Card signature verification, carried over from the implementation's findings:

- A key-resolution callback used to verify a card's signature **MUST** ignore any `jku` header the
  signer supplies and resolve the expected signer's key by DID instead — a callback that trusts
  `jku` lets a signer nominate its own trust root.
- `@a2a-js/sdk`'s card-signature verification accepts a card if *any one* of multiple signatures on
  it verifies; it does not bind the result to a single identity by itself. Callers needing a
  specific signer identity must check that identity against the returned result themselves.
- The extension's own `params` block is covered by the card signature (tampering is rejected); an
  unrelated unknown top-level field added after signing is not covered and is accepted. Do not rely
  on card-signature coverage for anything outside the extension's own declared `params`.

## 5. cheqd trust extension

Activation URI: `https://cheqd.io/a2a/ext/trust/v1` (see the URI note in [§1](#1-two-independent-extensions)).
Optional per Agent Card — a deployment may activate the proof extension ([§4](#4-requestresponse-proof-extension))
without this one.

### 5.1 Activation parameters

Declared per Agent Card (or per-deployment configuration), carrying up to three independent
checks, each individually optional:

| Parameter | Shape | Check |
|---|---|---|
| `anchoredDid` | A `did:cheqd:...` the caller claims | Reciprocal `alsoKnownAs` linkage between the caller's resolved DID and this one ([§5.2](#52-per-did-checks)) |
| `trustConfigRef` | `did:cheqd:(mainnet\|testnet):<id>?resourceName=<name>&resourceType=TrustConfigManifest\|CapabilityManifest\|ConformanceManifest` | DID-Linked Resource manifest resolution |
| `statusCheck` | `required` \| `optional` | Credential/ledger-anchored revocation status |

### 5.2 Per-DID checks

For a caller already verified by [§4](#4-requestresponse-proof-extension), with this extension
activated:

- If `anchoredDid` is set: resolve both the caller's canonical DID document and the claimed
  anchored DID's document; verify reciprocal `alsoKnownAs` linkage between them. Resolution
  failure on either side is a failure (`linkage_resolution_failed`), not a skip.
- If `trustConfigRef` is set: parse it per [§5.1](#51-activation-parameters)'s grammar — a
  malformed reference is a failure (`malformed_trust_config_ref`), not a skip — then resolve the
  named DID-Linked Resource and verify its manifest.
- If `statusCheck` is `required`: run the configured status checker against the caller's canonical
  DID; a revoked or non-fresh result is a failure. A `required` check with no status checker
  configured is itself a failure (`status_check_unavailable`), never silently skipped.

Every check in this section **fails closed**: a declared check this implementation cannot complete
(resolver error, missing dependency, malformed input) is a failure, never treated as equivalent to
the check not being declared at all.

### 5.3 Status-list verification: which construction

cheqd anchors status lists as DID-Linked Resources, but as of this draft there is no real
cheqd-issued `BitstringStatusListCredential` available whose proof actually covers `encodedList` —
that depends on upstream `did-provider-cheqd` work (tracked in [§7](#7-open-items--not-done)).
Until a signed list exists, two different status-checking approaches exist and are **not**
interchangeable:

- A ledger-anchoring check (`verifyLedgerAnchoredStatus`) that treats the DID-Linked Resource
  mechanism itself — not any credential proof — as the trust anchor: it checks the resolver's own
  checksum against the bytes actually returned, requires the resource to belong to the expected
  issuer's collection and be the latest version, and optionally requires multiple independent
  resolvers to agree. This works against real cheqd resources today, with no dependency on a
  signing change landing anywhere, at the cost of being cheqd-resolver-shaped rather than a
  portable W3C Bitstring Status List verifier.
- A proof-checked approach, for once a properly signed list exists: `@kya-os/mcp` ships **two
  incompatible** verifiers for this (`createStatusListProofVerifier`, generic
  `DataIntegrityProof`/`eddsa-jcs-2022`; and `createCheqdStatusListResolver`, a different,
  cheqd-specific construction entirely) that do not type-check against each other and reject each
  other's output. Once a cheqd list is properly signed, use the generic
  `createStatusListProofVerifier` + a revocation checker, not `createCheqdStatusListResolver` —
  this is a recorded upstream inconsistency, not a choice this spec is free to resolve unilaterally.

This extension's `statusCheck` parameter is deliberately agnostic to which of the above a given
deployment uses — it is a configuration choice for the `checkStatus` dependency passed to the trust
extension, not something this spec pins to one implementation.

### 5.4 Caching

Per-DID trust-extension results (not the request proof itself) **MUST** be cached and refreshed on
a TTL, never recomputed per call. These are discovery-time checks about a caller's standing, not
per-request authentication. A 5-minute default TTL is used in the implementation, matching the
manifest cache's own default; deployments may choose a different value. Streaming/high-throughput
deployments in particular would be materially harmed by resolving a manifest or status list on
every call.

## 6. Transports

| Transport | Status |
|---|---|
| JSON-RPC | Fully specified above; rejections surface as JSON-RPC errors under HTTP 200. |
| REST | Same `userBuilder`/wrapper design, same covered form; rejections surface as HTTP 400 (visible at the HTTP layer, unlike JSON-RPC). |
| gRPC, unary | Has its own `UserBuilder` seam (reads `call.request`, discriminates via `call.path`); the covered form must be computed from that request run back through the transport SDK's own canonical-JSON serialiser, not from the raw deserialised protobuf object — see [§2](#2-design-why-a-verifying-userbuilder-not-an-agentexecutor-decorator). A typed SDK error thrown from the gRPC gate reaches the client as a named error class; a plain `Error` surfaces only as gRPC status `UNKNOWN`. |
| gRPC, streaming (`sendStreamingMessage`'s server-streaming RPC) | **Not tested.** Only gRPC unary calls have been exercised against a real `@grpc/grpc-js` client/server. |
| Reverse proxy / sidecar | **Not tested.** Expected to be language-agnostic and independent of SDK internals, reusing the same verification logic, but no implementation or test exists yet. |

## 7. Open items / not done

These are the items that should be resolved, in roughly descending order of how much they block
this document from being proposed anywhere or finalised as a stable v1:

1. **Namespace ownership.** `org.kya-os/proof.v1` and its proposed URI
   (`https://kya-os.org/a2a/ext/proof/v1`) sit in the DIF TAAWG's namespace, not cheqd's. This
   draft proceeds on the assumption that namespace will hold; if it does not, the activation URI
   embedded in Agent Cards, the `A2A-Extensions` header and every conformance vector changes, and
   every deployed Agent Card declaring the old URI needs updating.
2. **URI mismatch for the trust extension.** This document uses
   `https://cheqd.io/a2a/ext/trust/v1`, matching the actual `agent-toolkit` implementation; the
   top-level repository README still proposes `.../identity/v1`. One needs to change to match the
   other before this is internally consistent.
3. **Not proposed to the A2A project.** No decision has been made on whether to propose either
   extension upstream, or on what timing relative to further interoperability testing.
4. **No external architectural review yet.** Tracked separately as a release gate for
   `agent-toolkit` and this repository both.
5. **Streaming carries no response proof** ([§3](#3-the-response-proof)). Whether that is
   acceptable long-term, or whether a chunk-level or final-chunk proof scheme should be designed
   for streaming, is an open design question, not yet attempted.
6. **The task-addressed-method identity gap** ([§4.1](#41-scope)): methods other than
   `SendMessage`/`SendStreamingMessage` (task cancellation, push-notification configuration, card
   fetch) pass through the enforcing wrapper entirely unverified in this draft.
7. **gRPC streaming calls** have not been tested at all (only unary).
8. **Reverse proxy / sidecar deployment** has not been implemented or tested.
9. **OAuth/DPoP token flow** for `cnf` sender-constraint fusion has not been exercised against a
   real OAuth authorization server — only a synthetic thumbprint.
10. **CRISP scope matchers** in delegation chains have not been tested — only flat scopes.
11. **`did:cheqd` discovery end to end** (Entity Card fetch, reciprocal linkage, manifest
    resolution against live resources together, as a single flow) has not been tested as a whole,
    only its individual pieces.
12. **No real signed `BitstringStatusListCredential`** exists yet for cheqd's status-list checks to
    run against for real (depends on `cheqd/did-provider-cheqd#657`/`#658`, open and unmerged as of
    this draft) — [§5.3](#53-status-list-verification-which-construction)'s ledger-anchoring check
    has only been run against a fake resolver built to mimic the real one's response shapes, not a
    live call, and is not yet integrated with the gate at all.
13. **`@kya-os/mcp`'s two incompatible status-list verifiers** ([§5.3](#53-status-list-verification-which-construction))
    remain unreconciled upstream; this spec does not and cannot resolve that unilaterally.

## License

[Apache 2.0](../LICENSE)
