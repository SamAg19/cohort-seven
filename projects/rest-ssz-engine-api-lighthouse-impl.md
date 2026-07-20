# REST-SSZ Engine API for Lighthouse

Add the new REST + SSZ Engine API transport (`execution-apis#793`) to Lighthouse, alongside the existing JSON-RPC transport

## Motivation

The Engine API is how a beacon node drives its execution client: importing blocks, updating the head, producing payloads, and fetching blobs. Today it runs over JSON-RPC, which carries three costs:

- **Larger and slower on the wire.** The consensus client holds these structures in SSZ, then converts them to hex-encoded JSON to send. SSZ is roughly 30–50% smaller and faster to parse, which matters most for blob data and the new Glamsterdam block access lists.
- **Version sprawl.** Every fork adds a new numbered version of each method.
- **A recurring source of bugs.** Encoding fields back and forth between formats has repeatedly caused subtle defects.

`execution-apis#793` replaces the hot path with an HTTP/REST API that sends SSZ bodies and selects the fork with a request header instead of a method name. This project implements that transport on the Lighthouse (consensus) side.

## Project description

The work **adds** a second Engine API transport without removing anything. The JSON-RPC path stays fully operational, and both run on the same port. The REST-SSZ transport is **opt-in behind a command-line flag**, so operators who don't enable it see no change. When enabled, Lighthouse probes the execution client once and keeps that choice for the run.

The transport choice stays **hidden inside the execution-layer crate** in the Lighthouse codebase. The rest of the beacon node calls the same methods and gets the same results, unaware of which transport ran. Lighthouse's existing safety checks (recomputing the block hash, verifying blob commitments, and validating payload status) sit above the transport and are reused unchanged.

The project also implements an endpoint that returns a block's execution witness together with its validation result, so stateless and zkVM verifiers get both in one round-trip.

## Specification

The transport comprises a REST-SSZ client, its SSZ data types, a seam that selects between transports, capability handling, fork sourcing for the fork-choice header, a one-time capability probe, and an opt-in flag.

**1. Wire.** The numbered JSON-RPC methods collapse onto REST endpoints:

| Old JSON-RPC methods | New REST endpoint |
|----------------------|-------------------|
| `engine_newPayloadV{1..5}` | `POST /payloads` |
| `engine_forkchoiceUpdatedV{1..4}` | `POST /forkchoice` |
| `engine_getPayloadV{1..6}` | `GET /payloads/{id}` |
| `engine_getPayloadBodiesByHashV{1,2}` | `	POST /{fork}/bodies/hash` |
| `engine_getPayloadBodiesByRangeV{1,2}` | `GET /{fork}/bodies?from=...&count=...` |
| `engine_getBlobsV{1..4}` | `POST /blobs/v{1..4}` |
| `engine_getClientVersionV1` | `GET /identity` |
| `engine_exchangeCapabilities` | `GET /capabilities` |

Hot-path calls send SSZ bodies and pick the fork with an `Eth-Execution-Version` header. Diagnostic endpoints stay JSON, and `eth_*` calls always use JSON-RPC.

**2. Client and data types.** A new client mirrors the JSON-RPC client, with one request chokepoint plus thin per-method wrappers, reusing today's JWT auth. Existing SSZ types are reused wherever they already match the wire format. New types are added only for genuine differences (e.g. the response dropping `INVALID_BLOCK_HASH`), each converting into an existing internal type so downstream logic is shared.

**3. Seam and selection.** A small component holds both clients and dispatches per call, using REST-SSZ once confirmed supported and JSON-RPC otherwise. With the flag on, Lighthouse probes `/capabilities` on the first successful health check. Success selects REST-SSZ, and a missing endpoint or failure falls back to JSON-RPC. The choice is frozen for the run.

**4. Fork and payload-id handling.** Capability tracking is generalized to work for both transports. REST needs the exact fork on each fork-choice update, sourced from information already in scope. It also uses an opaque, server-assigned payload token that can expire. The seam detects an expired token and transparently re-requests it, leaving the block-production path unchanged.

**5. Execution witness.** Stateless clients and zkVM provers need a block's execution witness, which today takes a separate call and lags a block behind. A new endpoint returns the validation result and SSZ witness in one request, following the same REST-SSZ conventions.

**6. Testing.** Tests reuse Lighthouse's Engine API test machinery, varying only the wire format: byte-level round-trips for the new types, request-format checks for the fork header and paths, an SSZ handler for the mock client, and the new REST behaviors. Integration is then exercised against real clients that implement the API (Erigon, Nethermind, Ethrex).

## Roadmap

The fellowship development period runs from **week 5 to week 21**.

**Weeks 5–7: Transport wiring & robustness.** A seam dispatches each call to the right transport, choosing REST-SSZ or falling back to JSON-RPC from a one-time `/capabilities` probe on the first health check and then freezing that choice. Capability tracking moves from a bool struct to a `JsonRpc` / `Ssz` enum. The remaining robustness follows the spec: fork-choice updates are serialized, expired opaque payload ids are transparently re-requested and retried, and bodies are gated per fork so an unsupported fork errors out instead of silently falling back.

**Weeks 8–9: Test apparatus & mock coverage.** Share the `MockExecutionLayer` core between transports, add an SSZ handler and a `/capabilities` responder, and cover the new REST behaviors, SSZ types, transport selection, and a switch to run existing suites over REST. Existing JSON-RPC tests must still pass.

**Weeks 10–11: Integration testing.** Test end to end against real execution clients that implement REST-SSZ (Erigon, Nethermind, Ethrex) and resolve interoperability issues.

**Weeks 12–13: Research & metrics.** Measure REST-SSZ against JSON-RPC on the devnets (wire size, parse and round-trip latency, and block-import throughput, especially for blob-heavy payloads), quantifying the improvement and flagging any regressions to fix.

**Weeks 14–15: Glamsterdam follow-ups.** Track Lighthouse's Glamsterdam PRs that affect the Engine API (e.g. [`get_blobs_v4` #9438](https://github.com/sigp/lighthouse/pull/9438) and [custody columns #9547](https://github.com/sigp/lighthouse/pull/9547)). Rework the SSZ structures for [EIP-7688](https://eips.ethereum.org/EIPS/eip-7688) (forward-compatible progressive containers) once it moves from CFI to SFI.

**Weeks 16–17: Execution-witness endpoint.** Implement the REST-SSZ payload-with-witness endpoint in Lighthouse and test it against a supporting execution client.

**Weeks 18–20: Spec buffer.** Reserved time to absorb revisions to the still-evolving spec.

**Weeks 21+: Report, demo & handoff.** Final EPF report, demo and presentation, and maintainer handoff.

Several of the items above (the witness endpoint and the Glamsterdam follow-ups) are externally gated and picked up whenever their dependencies clear.

## Possible challenges

- **The spec is still evolving.** Some `execution-apis#793` details (token lifetime, request ordering, size limits) are unsettled. The design stays robust to how they resolve, and the weeks 18–20 buffer absorbs revisions.
- **External dependencies.** The Glamsterdam follow-ups (the `get_blobs_v4` / custody-columns PRs, and EIP-7688 once it reaches SFI) land on their own timelines, as tracked follow-ups rather than blockers.
- **Cross-client interop.** Real clients already implement REST-SSZ, so integration testing will surface discrepancies to triage or report upstream.
- **The opaque payload-id behavior** is the one new semantic difference, handled without leaking into callers.
- **Encapsulation.** No REST-specific detail may creep into a public method.

## Goal of the project

**Minimal goal.** REST-SSZ working against a real execution client on local Kurtosis devnets, with the flag off leaving behavior unchanged.

**Main goal.** Full integration testing and a complete REST-SSZ implementation matching the finalized spec, merged into Lighthouse and ready for Glamsterdam.

**Stretch goal.** Use the execution-witness endpoint to explore stateless block building with an execution client and contribute that support to Lighthouse.

## Collaborators

### Fellows

[Sameer](https://github.com/SamAg19)

### Mentors

[Mac Ladson](https://github.com/macladson) - Lighthouse, Sigma Prime.

## Resources

- [`execution-apis#793`](https://github.com/ethereum/execution-apis/pull/793): the REST + SSZ Engine API transport proposal.
- [`execution-apis#773`](https://github.com/ethereum/execution-apis/pull/773): REST + SSZ payload validation with execution witness, the basis for the witness endpoint implemented in this project.
- [Lighthouse](https://github.com/sigp/lighthouse): the work lands in the `beacon_node/execution_layer` crate.

