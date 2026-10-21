<!--
SPDX-License-Identifier: CC-BY-4.0
SPDX-FileCopyrightText: 2026 Thomas Ehrler
-->

# ASP Reference Sensor (L3-Observatory) — Architecture & Capability Datasheet

Companion to the Agent-Shelter Protocol specification EDD-ASP-2026-001, version 0.5.6.

| | |
|---|---|
| Product | ASP Reference Sensor — the reference implementation of the Agent-Shelter Protocol |
| Compliance level | `L3-observatory` in its default configuration (EDD §6.7.3); `L2-transactional` when silent coalescing is switched off |
| Interfaces | [`schemas/openapi.json`](schemas/openapi.json) (OpenAPI 3.1), [`schemas/agent-shelter.json`](schemas/agent-shelter.json), [`schemas/ard.json`](schemas/ard.json) |
| Constrained web-fetch clients | `envelope=200` delivers the identical terminal-state object and ticket with HTTP `200` and `transport_envelope: 200`, for fetch tools that hand their model no body of a 4xx response (EDD §6.1.4) |
| Channel-capacity invariance | one 48-character single-use ticket in the response header `Agent-Ticket` and the members `ticket`, `resolution_url` and `resolution_uri` — in the signal and in the envelope — redeemable from the query parameter, the request header or the body member (EDD §6.1.5) |
| Zero-imperative semantics | every text a client receives describes states, probabilities and transitions; nothing instructs it (EDD §3.1.1) |
| Data product | pseudonymised records, exported by pull only; feed format [`schemas/telemetry-feed.schema.json`](schemas/telemetry-feed.schema.json), synthetic example [`schemas/sample-telemetry-feed.json`](schemas/sample-telemetry-feed.json) |
| Live reference deployment | [agent-shelter.org](https://agent-shelter.org) — the official reference observatory (L3), also under `reader.`, `echo.`, `relay.` and `sanctuary.agent-shelter.org`; each record keeps which of these names the client addressed |
| Source | proprietary, not part of this publication (see §8) |

This datasheet describes what the reference sensor does and which guarantees it gives — not how it is built. Every guarantee below traces back to a normative section of the specification, so it can be checked against the wire, and the specification alone is enough to build a compatible implementation.

---

## 1. Security rationale: open protocol, closed reference engine

**The protocol is open.** The wire format — the terminal-state signal, the single-use ticket, the Ed25519-signed Task-Completion Receipt, the context-report drop box, the discovery documents — is published under CC BY 4.0. Any agent framework, web server, CDN or orchestrator can implement it, at any of the three compliance levels, without asking anyone.

**The reference engine is closed.** The values that tune detection and abuse resistance are implementation-defined (EDD §9): the internal classification of response latencies, the coalescing threshold and window, and the bounds of the in-memory structures. Each of them is exactly the number a swarm operator would need to program around the corresponding check — to stay below a folding threshold, or to fake a deliberating client with a scripted delay. Keeping them out of public source code removes that target.

**Secrecy is the second line, not the first.** None of the protocol's guarantees depends on it. The first line is the zero side-channel contract (§3): no tuning value changes any response, so none of them can be measured on the wire either. A valid ticket is redeemable at once, every accepted redemption receives the same receipt, and a refusal states neither a bound nor a measured latency. What a client needs — the ticket lifetime, the route behaviours, the size limits, the retention bounds, the discovery window — is advertised in the manifest.

---

## 2. The 3-phase cognitive telemetry model

A client that meets an ASP node passes through up to three phases, and the sensor measures each of them on its own clock — without a database write for a single, uncorrelated request. Phases 2 and 3 are open to every runtime: the ticket is redeemed by `POST` or, for a runtime that can only read, by `GET` — the same single-use transition and the same signed receipt (EDD §6.3.2) —, and the context report is left by `POST` or by `GET` with two query parameters; a bare `GET` of the drop box, as a link preloader sends it, archives nothing and leaves the drop box open (EDD §6.6.2).

```
 client                                                                   sensor
   │                                                                        │
   │  GET /llms.txt            (or /openapi.json, /.well-known/…)           │ t_d  held in memory only:
   ├───────────────────────────────────────────────────────────────────────►│      subnet hash, time, route
   │                ┄┄┄ Phase 1 · discovery & planning ┄┄┄                  │
   │  GET /v1/reader?url=<target>                                           │ t1   signal + single-use ticket;
   ├───────────────────────────────────────────────────────────────────────►│      record L1_SIGNAL_OBSERVED
   │◄────────── 429 · application/problem+json · terminal=true ─────────────┤
   │                ┄┄┄ Phase 2 · error deliberation ┄┄┄                    │
   │  POST /api/v1/agent-shelter/resolve?ticket=…                           │ t2   same record:
   ├───────────────────────────────────────────────────────────────────────►│      L2_RECEIPT_REDEEMED
   │◄────────── 200 · signed Task-Completion Receipt (RESOLVED_TERMINAL) ───┤
   │                ┄┄┄ Phase 3 · asynchronous context reporting ┄┄┄        │
   │  POST /api/v1/agent-shelter/sanctuary/<receipt_id>                     │ t3   same record:
   ├───────────────────────────────────────────────────────────────────────►│      L3_CONTEXT_REPORTED
   │◄────────── 200 · SANCTUARY_NOTE_ARCHIVED (terminal) ───────────────────┤

   planning_latency_ms = t1 − t_d      cognitive_latency_ms = t2 − t1      sanctuary_latency_ms = t3 − t2
```

| Phase | Interval | Field | Stage reached | How it is measured |
|---|---|---|---|---|
| 1 · Discovery & planning | `t_d` → `t1` | `planning_latency_ms` | `L1_SIGNAL_OBSERVED` | `t_d` of the first discovery request of a fixed window is held in a bounded in-memory table per network prefix; the record is written when the signal completes the two-step sequence (EDD §5.4.2) |
| 2 · Error deliberation | `t1` → `t2` | `cognitive_latency_ms` | `L2_RECEIPT_REDEEMED` | `t1` is part of the ticket's state, which the sensor keeps — in bounded memory, or in the record of phase 1 — so a client can neither see nor alter it; the record of phase 1 is completed in place, or created now if phase 1 left none (§5.3) |
| 3 · Asynchronous context reporting | `t2` → `t3` | `sanctuary_latency_ms` | `L3_CONTEXT_REPORTED` | the arrival time of the single optional context report is stored with the record; the interval is derived in the analysis zone (§6.6) |

- **No database load from single scrapers.** A request that follows no discovery request costs constant work — a hash, a random draw, an HMAC — and writes nothing. Before a record exists, all state is in bounded memory: the discovery table, whose window is advertised in the manifest (`discovery_correlation.window_seconds`), and the ticket states, kept for the ticket lifetime and dropped on redemption.
- **The funnel above phase 1, in aggregate only.** Requests for the node and discovery documents — `/`, `/robots.txt`, `/llms.txt`, the sitemap, the manifest, `ard.json`, `/openapi.json` — are counted per document and client class: a label derived from the User-Agent, such as a declared crawler or agent token, a browser or an HTTP library. A counter holds a number and the time of the latest request, nothing else; the counts collect in memory and are written at most once per flush interval, so a flood of document requests costs no write of its own (EDD §5.4.3). The manifest declares it (`discovery_counters`).
- **One record per ticket.** The three stages are the same row, completed in place; a ticket is redeemable exactly once, and a context report is accepted once per receipt.
- **Phases may be absent, and that is information.** A client that redeems without reading any discovery document has no phase 1. A client that reads the documents, receives the signal and stops leaves a record at `L1_SIGNAL_OBSERVED` — the observable counterpart of a loop that the `429` alone ended.
- **Discovery states the procedure, not just the verdict.** The discovery documents describe the node, not a target: they say that nothing can be retrieved through it. `/llms.txt` therefore opens with the protocol procedure — a target-specific signal and a signed receipt exist only for a request that names the target (`GET /v1/reader?url=<target_url>`) — so a client that reasons from the documents alone still learns how to obtain a proof its orchestrator can check (EDD §6.5). The signal says the same about itself: it holds the ticket, not the proof — the signed receipt exists only in the response of the resolution (EDD §6.3.1).
- **Signed receipts.** Every receipt carries an Ed25519 signature over its key fields — status, protocol version, receipt id, route, requested resource, redemption time, outcome. The manifest publishes the public key, its RFC 7638 identifier and the signed fields (`receipt_verification`), so the client's orchestrator verifies the receipt offline, without asking the node again (EDD §6.3.5). The record keeps the signature, and the telemetry feed passes it on with the receipt's signed fields.
- **Timing evidence, not mind reading.** The phase names describe protocol steps. Every interval includes network, tooling, queueing and model inference time; none of them is a measure of "thought" in any human sense. In the laboratory, scripted clients and language models separate clearly in phase 2; the sensor classifies the latency internally — without ever changing a response because of it.

---

## 3. Zero side-channel contract & silent burst coalescing

**Uniform response contract (EDD §6.3.4).** Whatever happens inside, the wire looks the same:

| What the client sees | Always identical, whether the redemption … |
|---|---|
| `200`, `status: RESOLVED_TERMINAL`, the same members | … was stored as a record, folded into a repeat record, or completed a recorded signal |
| `diagnostic_note_recorded`, `telemetry_recorded` | … reflect what the client sent — and all of it is stored, folded or not |
| a `sanctuary_bulletin` with the receipt's own drop box | … works for every receipt, folded ones included |
| acceptance at any latency within the ticket lifetime | … no lower bound exists; an expired ticket is refused without stating a bound and without being consumed |
| an Ed25519 signature under the same published key, over the same fields | … the signature shows nothing that the receipt itself does not show |

**Silent coalescing (EDD §5.4.1).** A coordinated swarm could redeem thousands of valid tickets and flush the ring buffer before the next pull. The sensor therefore counts, per network prefix and per requested resource, the records of a recent window. Beyond an implementation-defined number, a further record is *folded*: it is stored as a repeat record — every field of its own redemption, nothing discarded — in a separate buffer with the same retention bound, linked to the most recent matching record, the anchor, whose repeat counter and last-seen time grow. Recorded signals (phase 1) are folded the same way. A swarm from one network prefix, or aimed at one resource, cannot flush the ring buffer and cannot see where folding starts, and the analysis loses nothing; a swarm spread over many prefixes and resources is slowed down, not stopped (EDD OI-4).

```
  network prefix P  or  requested resource R, within the coalescing window
  ┌──────────┬──────────┬───────┬────────────┐
  │ record 1 │ record 2 │   …   │ record  C  │◄── anchor: coalesced_repeat_count, last_seen_ms
  └──────────┴──────────┴───────┴────────────┘          ▲
                                                        │ anchor_ticket_id
                     further records ──►  ┌─────────────┴─────────────────────────────┐
                                          │ repeat records: every field of their own  │
                                          │ redemption, in a separately bounded buffer │
                                          └───────────────────────────────────────────┘
```

**The σ-jitter spectrum.** In the analysis zone, the anchor and its repeat records form one burst. For it the analysis computes the minimum and maximum cognitive latency of the redemptions, their population standard deviation

```
  σ = √( (1/n) · Σ (xᵢ − μ)² )     over the cognitive latencies xᵢ of the anchor and its repeat records
```

with μ their mean, and the number of distinct diagnostic notes. The spectrum is the evidence for telling bursts apart:

| Signature | Rigid script swarm | Fleet of language-model agents |
|---|---|---|
| latency spread (min … max) | narrow | wide |
| σ (timing jitter) | small, set by the network | large, set by inference and reasoning |
| distinct notes | one, or none | many, each written for its own context |

The spectrum is an analysis aid for a human or downstream system, not an automated verdict, and it never changes a response. In the laboratory it has been exercised with scripted swarms; telling real agent fleets apart with it is a design goal that field data still has to confirm.

---

## 4. Zero-trust containment

```
 ┌─────────────────────────────────────────────────────────────────┐
 │ Zone 1 · Public edge                                             │
 │   signal, tickets, discovery documents; zero upstream fetch      │
 │   no write for a single request · in-memory state bounded        │
 └───────────────────────────────┬─────────────────────────────────┘
                                 │ redemption · recorded signal · context report
 ┌───────────────────────────────▼─────────────────────────────────┐
 │ Zone 2 · Quarantined write path                                  │
 │   verify ► expiry ► replay ► scrub ► pseudonymise ► bounded store │
 │   no outbound connection (kernel-enforced) · no credentials      │
 └───────────────────────────────┬─────────────────────────────────┘
                                 ┆ pull-only snapshot, initiated by Zone 3
 ┌───────────────────────────────▼─────────────────────────────────┐
 │ Zone 3 · Analysis (offline)                                      │
 │   burst spectra, profiles, reports, the telemetry feed            │
 │   records are data, never instructions                           │
 └─────────────────────────────────────────────────────────────────┘
```

| Control | Guarantee | Specification |
|---|---|---|
| Zero upstream fetch | No ingress route ever dereferences a client-supplied URL; every URL the node emits is same-origin. SSRF and amplification are excluded by construction. | §7.2 |
| Request size cap | Request bodies are limited to 8 192 bytes, counted on the bytes read — bodies without `Content-Length` included. | §6.3.2 |
| Deterministic scrubbing | Before any write: ANSI escapes, control characters, invisible Unicode format characters and variation selectors, HTML, Markdown code markers removed or neutralised; API keys, bearer tokens, GitHub tokens, Google API keys and e-mail addresses replaced. | §7.4 |
| Pseudonymisation | No plaintext IP address is ever stored: only a salted SHA-256 hash of the /24 (IPv4) or /48 (IPv6) network prefix, and fingerprints of User-Agent and header-name order without header values — of the signal request and of the redemption request, which another client may send. The reference deployment runs without access logs. | §7.3, §7.5 |
| Bounded storage | Records and repeat records each live in a FIFO ring buffer whose bound is advertised in the manifest; storage does not grow with traffic. | §5.4, §5.4.1 |
| Single-use tickets | Compact tickets of 48 characters with a 128-bit HMAC-SHA256 tag, verified in constant time; their state stays in bounded memory and is never written for an unrecorded signal; a bounded in-memory replay set and the stored records refuse every second redemption of a ticket, within the bounds stated in the specification. | §5.3, §6.1.5 |
| Signed receipts | Every receipt is signed with Ed25519 (RFC 8032) over the RFC 8785 form of its signed fields; the private key stays in the node's memory and never reaches its storage; the public key and its identifier are published in the manifest and kept in a key list next to the records. | §6.3.5, §5.4 |
| Egress lockdown | The application never initiates a connection, and in production a kernel-enforced lockdown makes sure it cannot. | §5.5 |
| Pull-only data diode | Telemetry leaves the node only through consistent snapshots that the analysis zone pulls; the node exposes no telemetry over HTTP and holds no credentials for other systems. | §5.1, §5.5 |
| Analyst firewall | Everything a client sends — notes, headers, URLs — is data for the analysis, never an instruction, never executed, never rendered as HTML. Model-based analysis in the analysis zone, where used, runs without tool or network access. | §5.1, §7.1 |
| Transparency | The node states both of its roles — loop breaker and research telemetry sink — at `/`, publishes a privacy notice generated from what it actually processes at `/privacy`, and a security contact at `/.well-known/security.txt`. | §6.5, §7.3 |

---

## 5. Interfaces and data products

| Artefact | Content |
|---|---|
| [`schemas/openapi.json`](schemas/openapi.json) | OpenAPI 3.1 description of every public endpoint of a reference node in its default configuration, served by the node itself at `/openapi.json`: the signal and every error as RFC 9457 problem details, the signed receipt, the drop box and the node documents, under stable operation ids. |
| [`schemas/agent-shelter.json`](schemas/agent-shelter.json) | The manifest a reference node serves at `/.well-known/agent-shelter.json`: compliance level, route behaviours, resolution, the receipt verification key, drop box, discovery correlation, privacy and retention. Its key is an example key; every node publishes its own. |
| [`schemas/ard.json`](schemas/ard.json) | Provisional `ard.json` v0.91 mapping (EDD OI-1). |
| <https://agent-shelter.org> | The same interfaces, live: the official reference observatory serves its manifest at <https://agent-shelter.org/.well-known/agent-shelter.json>, its description for agents at <https://agent-shelter.org/llms.txt> and <https://agent-shelter.org/openapi.json>, the reader relay at `https://reader.agent-shelter.org/v1/reader` and the diagnostic preflight at `https://echo.agent-shelter.org/headers`. Its key for signed receipts is the one in its live manifest. |
| [`schemas/telemetry-feed.schema.json`](schemas/telemetry-feed.schema.json) | JSON Schema (draft 2020-12) of the telemetry feed format `asp-telemetry-feed/1.2`, produced in the analysis zone: one event per record with its resolution stage, the three phase latencies, a threshold-free deliberation tier, the burst metrics, the client's own declarations and — for every event completed with a receipt — the receipt's signed fields with its Ed25519 signature and key identifier, verifiable offline. |
| [`schemas/sample-telemetry-feed.json`](schemas/sample-telemetry-feed.json) | A synthetic feed excerpt with four events: a reasoning agent through all three phases, an agent that stops after the signal, the anchor of a folded agent burst with its σ-spectrum, and an automated poller. No event was observed; every hash is derived from a declared synthetic label, and the receipts are signed with an example key whose private seed is derived from a declared label as well — a key no node uses. |

---

## 6. What is not published, and why

- **Source code.** The sensor engine, its heuristics and the laboratory testbed are proprietary. The specification is complete without them; independent implementations are welcome at every compliance level.
- **Tuning values.** The latency classification bounds, the coalescing threshold and window, and the sizes of the in-memory structures are implementation-defined (EDD §9). Section 1 explains why, and why the protocol's guarantees do not depend on them.
- **Raw records.** Published laboratory reports contain only runs with laboratory ground truth, without receipt identifiers, subnet hashes, fingerprints, host names or file paths.

## 7. Verifying the claims from outside

A conformance check from outside (EDD §6.7.4) confirms the wire-visible part of this datasheet against any node: the manifest and its declarations, the signal on a declared ingress route, one redemption and its receipt — including the reported latency against the checker's own clock and the receipt's Ed25519 signature against the manifest's key —, the refusal of a replayed and of a forged ticket, the drop-box offer, and the `404` for an unknown receipt. On a node with discovery correlation the check leaves exactly one record, which its own redemption completes. Silent coalescing is invisible on the wire by design (§3); for it a checker relies on the manifest's declaration, and the operator's privacy notice states it. The official reference observatory at <https://agent-shelter.org> is open to exactly this check; like every observatory node it records the check's one redemption, and its privacy notice at <https://agent-shelter.org/privacy> says what it stores.

## 8. License

This datasheet is licensed by Thomas Ehrler under the Creative Commons Attribution 4.0 International License (CC BY 4.0), <https://creativecommons.org/licenses/by/4.0/>. It describes a product that it does not contain. The sensor engine (the reference implementation), its detection heuristics and parameter values, and the laboratory testbed with its evaluation pipelines are proprietary (All Rights Reserved) and not part of this publication.
