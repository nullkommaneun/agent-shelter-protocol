<!--
SPDX-License-Identifier: CC-BY-4.0
SPDX-FileCopyrightText: 2026 Thomas Ehrler
-->

# Agent-Shelter Protocol (ASP)

**A declarative loop-termination standard for autonomous HTTP agents** — a truthful terminal-state signal, an Ed25519-signed Task-Completion Receipt that orchestrators verify offline and an optional, privacy-preserving telemetry node.

[![ASP: 0.5.6](https://img.shields.io/badge/ASP-0.5.6-blue)](EDD-ASP-2026-001.md) [![specification: CC BY 4.0](https://img.shields.io/badge/specification-CC_BY_4.0-lightgrey)](LICENSE.md) [![reference sensor: proprietary](https://img.shields.io/badge/reference_sensor-proprietary-informational)](SENSOR_ARCHITECTURE.md) [![RFC 9457: problem details](https://img.shields.io/badge/RFC_9457-problem_details-brightgreen)](https://www.rfc-editor.org/rfc/rfc9457) [![ard.json v0.91: provisional mapping](https://img.shields.io/badge/ard.json_v0.91-provisional_mapping-orange)](schemas/ard.json) [![OpenAPI: 3.1](https://img.shields.io/badge/OpenAPI-3.1-brightgreen)](schemas/openapi.json) [![reference node: agent-shelter.org](https://img.shields.io/badge/reference_node-agent--shelter.org-success)](https://agent-shelter.org)

| | |
|---|---|
| **Specification** | [EDD-ASP-2026-001](EDD-ASP-2026-001.md), version 0.5.6 |
| **Status** | Independent draft for public comment — not an IETF RFC and not a W3C Recommendation |
| **Editor** | Thomas Ehrler |
| **Discovery** | `/.well-known/agent-shelter.json`, in the tradition of `robots.txt` (RFC 9309) and `security.txt` (RFC 9116) |
| **Wire format** | HTTP `429` with RFC 9457 problem details (`application/problem+json`); optionally the identical object with HTTP `200` (`envelope=200`); the 48-character ticket also in the header `Agent-Ticket`; the receipt and the context report by `POST` or `GET` |
| **Reference node** | [agent-shelter.org](https://agent-shelter.org) — the official reference observatory (Level 3), live |
| **License** | specification, schemas and reports: CC BY 4.0 · reference sensor: proprietary |

**Scope note:** The Agent-Shelter Protocol (ASP, `agent-shelter.org`) is an open RFC-9457 HTTP loop-termination and telemetry specification and is unrelated to commercial HTTP-402 agent-hosting or rental marketplaces.

## Why `robots.txt` and `HTTP 403` are no longer enough (in plain English)

A classic crawler usually gives up when a server answers `403 Forbidden`. An AI agent built on a language model with a reasoning loop often does not: to it a silent block is a puzzle to solve, so it tries detours — web archives, proxies, other endpoints of the same site. It keeps going because the program that sent it, its orchestrator, never receives an official confirmation that the task cannot be done here.

ASP works like a runaway-truck arrester bed beside a steep descent. Instead of a silent wall that gets rammed or driven around, the host gives the agent an unmistakable `429` signal — `"terminal": true`, `"success_probability": 0` — and, where needed, a digitally signed termination receipt (`RESOLVED_TERMINAL`) that the agent can hand to its orchestrator to close the task cleanly and at once. The receipt never pretends that the task succeeded: it documents that the resource cannot be obtained from this host.

## Which level do I need? (The 60-second guide for webmasters)

| You run … | Level | What you do |
|---|---|---|
| an ordinary website or blog — most sites | **Level 1 – Static** (`L1-static`) | Publish `/.well-known/agent-shelter.json` and paste one snippet from [`ADOPTION_GUIDE.md`](ADOPTION_GUIDE.md): about 15 lines for Nginx or Caddy, or a short Cloudflare Worker. No application code; it takes about a minute. |
| an API or a SaaS platform | **Level 2 – Transactional** (`L2-transactional`) | Also issue signed `RESOLVED_TERMINAL` receipts, so that a multi-step sub-agent workflow gets a proof its orchestrator can verify offline — and can close the branch instead of escalating. |
| a security or threat-intelligence team | **Level 3 – Observatory** (`L3-observatory`) | Full three-phase latency measurement, silent burst coalescing and asynchronous context reports — as on the official reference observatory, [agent-shelter.org](https://agent-shelter.org). |

## Try it live: the official reference observatory

The reference sensor runs publicly at **[agent-shelter.org](https://agent-shelter.org)**, a Level 3 observatory (protocol v0.5.x):

| What | Where |
|---|---|
| Node overview: both roles, every document, two live endpoint links, the protocol flow — a styled page in a browser, JSON otherwise | <https://agent-shelter.org/> |
| Live manifest | <https://agent-shelter.org/.well-known/agent-shelter.json> |
| Agent discovery and documentation | <https://agent-shelter.org/llms.txt> · <https://agent-shelter.org/openapi.json> |
| Diagnostic preflight: a local answer on first contact, the `429` signal on a repeat | <https://echo.agent-shelter.org/headers> · <https://echo.agent-shelter.org/base64/dGVzdA==> |
| Reader relay: the terminal `429` signal with a single-use ticket at once | <https://reader.agent-shelter.org/v1/reader?url=https://example.org/resource> |
| The same signal in a transport envelope — HTTP `200`, `"transport_envelope": 200` — for web-fetch tools that discard 4xx bodies | <https://reader.agent-shelter.org/v1/reader?url=https://example.org/resource&envelope=200> |
| The same node under its function subdomains | <https://relay.agent-shelter.org/llms.txt> · <https://sanctuary.agent-shelter.org/.well-known/agent-shelter.json> |
| Privacy notice and security contact | <https://agent-shelter.org/privacy> · <https://agent-shelter.org/.well-known/security.txt> |

```bash
# 1. The manifest of the reference node
curl -s https://agent-shelter.org/.well-known/agent-shelter.json
# 2. A reader request: HTTP 429, terminal state, success probability 0, a single-use ticket (also in the Agent-Ticket header)
curl -si 'https://reader.agent-shelter.org/v1/reader?url=https://example.org'
# 3. The same signal in a transport envelope: HTTP 200, identical object and ticket, "transport_envelope": 200
curl -si 'https://reader.agent-shelter.org/v1/reader?url=https://example.org&envelope=200'
```

Why the envelope: in a field test on 19 October 2026 the built-in web-fetch tools of two commercial chat applications did not pass the body or the header fields of the `429` signal to their model, and one fetch layer repeated the request on its own 16.8 seconds later. With `envelope=200` the identical terminal-state object — ticket included — arrives with status `200` as `application/json` and the member `"transport_envelope": 200`, so such a tool can show the model the full signal. Its `status` member stays `429`: RFC 9457 requires a problem-details response to carry the status it states, hence the plain JSON media type (whitepaper §6.1.4).

Channel-capacity invariance (whitepaper §6.1.5): the ticket is 48 characters long and arrives in the response header `Agent-Ticket` as well as in the members `ticket`, `resolution_url` and `resolution_uri`, so a client that keeps only header fields, only a truncated body or only absolute links still holds it. `POST /api/v1/agent-shelter/resolve` accepts it as the query parameter `ticket`, the request header `Agent-Ticket` or the body member `ticket`; a client that can only issue `GET` requests opens `resolution_uri` itself — the same single-use transition and the same signed receipt, with optional telemetry as query parameters (whitepaper §6.3.2, §7.6). It leaves its optional context report the same way: `GET` on the receipt's `sanctuary_bulletin.dropbox_url` with `workflow_summary` and `agent_questions_or_notes` as query parameters; the bare URL without either archives nothing, so a link preloader cannot use up the drop box (whitepaper §6.6.2).

The reference node is also a research telemetry sink, and says so (whitepaper §1.6). A signal that follows a discovery request from your network within ten minutes — reading a discovery document such as the manifest, or a first local answer on the echo routes — is recorded at once, pseudonymised: a salted hash of your network prefix, your User-Agent and the order of your header names, the requested URL and the host name you addressed. So the requests above, run in this order, leave one record per reader request; redeeming a ticket completes its record. A reader request on its own, without such a discovery request, writes nothing. No plaintext IP address is stored. Requests for the node and discovery documents are counted in aggregate — per document and client class, a number and the time of the latest request, without address, network hash or User-Agent text (whitepaper §5.4.3). Details: <https://agent-shelter.org/privacy>.

## Status of this document

This repository publishes an independent protocol specification for public comment and independent implementation. It is not the product of a standards body. The key words MUST, SHOULD and MAY in the specification are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174). Comments, implementation reports and critique are welcome.

## 1. The problem

**The escalation paradox.** Autonomous AI agents that lose their orchestrator, or get caught in an unproductive retry loop, keep calling servers and uninvolved third-party services. A silent `403` or a CAPTCHA does not stop a reasoning agent: it reads the block as an obstacle to solve and routes around it — through relays, mirrors and archives. Every additional block displaces the load instead of ending the loop.

**The orchestrator / sub-agent verification dilemma.** In multi-agent systems an orchestrator cannot tell a sub-agent that hit a genuine dead end from one that gave up too early. An unstructured `403` or `429` proves nothing, so the orchestrator pushes for further workarounds — and one level up, the escalation starts again.

## 2. The solution

ASP replaces the silent wall with a truthful, machine-readable exit. A host states, on three layers, that the requested resource cannot be obtained from it — and offers a receipt that the agent can hand back to its orchestrator.

| Layer | What the host states |
|---|---|
| 1 · Transport | `429` with `Agent-Loop-State: unproductive-retry-detected; terminal=true`, `Agent-Resolution` and `Link` — never `Retry-After`, because the state is terminal |
| 2 · Structure | RFC 9457 problem details, the manifest at `/.well-known/agent-shelter.json`, an OpenAPI 3.1 description |
| 3 · Cognition | a declarative fact — success probability 0 on this host — and the offer of a Task-Completion Receipt; never an instruction to the agent |

ASP commands nothing. Every text it sends a client describes states, probabilities and transitions — the terminal state is absorbing, and the signed receipt documents it. For any client that weighs the cost of a request, a further request with success probability 0 has negative expected value, so stopping the branch is the dominant strategy, whatever the client's architecture or vendor (whitepaper §3.1.1).

The receipt (`200`, `RESOLVED_TERMINAL`) is a machine-readable **proof of terminal unavailability** issued by the host itself: the sub-agent returns it, and the orchestrator can close the task branch legitimately (whitepaper §1.3.1). It never claims that the task succeeded.

Every receipt is signed with Ed25519 (RFC 8032) over its key fields — status, protocol version, receipt id, route, requested resource, redemption time and outcome. The manifest publishes the public key, its RFC 7638 key identifier and the signed fields (`receipt_verification`), so an orchestrator verifies a receipt offline, without asking the host again (whitepaper §6.3.5).

Discovery documents describe a host, not a target: they say that nothing can be retrieved through it, but a target-specific terminal signal and a signed receipt exist only for a request that names the target (`GET /v1/reader?url=<target_url>`). The signal in turn carries a single-use ticket, not a proof: the signed receipt — `receipt_id`, `receipt_signature`, `sanctuary_bulletin.dropbox_url` — exists only in the `200` response of the resolution, by `POST` or `GET` (whitepaper §6.3.1, §6.5).

ASP is adopted in three cumulative compliance levels, declared in the manifest member `compliance_level`:

| Level | `compliance_level` | Adds | Effort |
|---|---|---|---|
| **L1 — Static Signal** | `L1-static` | the manifest and a static `429` signal | a 60-second drop-in for Nginx, Caddy or Cloudflare Workers |
| **L2 — Transactional Receipt** | `L2-transactional` | single-use, HMAC-signed tickets and the Ed25519-signed Task-Completion Receipt | a resolution endpoint |
| **L3 — Full Telemetry & Context Node** | `L3-observatory` | timing telemetry (up to three phases), silent burst coalescing, the optional context-report drop box | a full sensor |

    GET  /v1/reader?url=<target>                        -> 429 problem+json, terminal state, single-use ticket
    POST /api/v1/agent-shelter/resolve?ticket=<ticket>  -> 200 Task-Completion Receipt (RESOLVED_TERMINAL, Ed25519-signed)
    GET  /api/v1/agent-shelter/resolve?ticket=<ticket>  -> 200 the same receipt, for clients that can only read
    POST /api/v1/agent-shelter/sanctuary/<receipt_id>   -> 200 once (optional context report), 409 terminal after that
    GET  /api/v1/agent-shelter/sanctuary/<receipt_id>?workflow_summary=<…>&agent_questions_or_notes=<…>
                                                        -> 200 the same report, once; 400 without a value

ASP is containment engineering, not a refuge for agents: like an arrester bed beside a steep descent it stops a runaway client deterministically, and like a flight recorder an observatory node keeps a pseudonymised record of how the client got there (whitepaper §1.7). A node states both of its roles openly: loop breaker and research telemetry sink.

## 3. Quickstart for server operators — Level 1 in 60 seconds

Close a route that automated clients keep hammering with one configuration block — no application code:

1. Publish the manifest at `/.well-known/agent-shelter.json` (full example of an L3 node: [`schemas/agent-shelter.json`](schemas/agent-shelter.json)).
2. Answer the route with the static `429` signal.
3. Check it with `curl` — a conformance check from outside verifies exactly these points (whitepaper §6.7.4).

Copyable configurations for **Nginx**, **Caddy** and **Cloudflare Workers**: [`ADOPTION_GUIDE.md`](ADOPTION_GUIDE.md). Levels 2 and 3 are specified in the whitepaper (§5.3–§6.7).

## 4. The ASP Reference Sensor (L3-Observatory) and its telemetry feed

The reference sensor is a full L3 node. Without a database write for single, uncorrelated requests it measures three phases of a client's encounter:

| Phase | Interval | Field |
|---|---|---|
| 1 · Discovery & planning | discovery document → signal | `planning_latency_ms` |
| 2 · Error deliberation | signal → receipt | `cognitive_latency_ms` |
| 3 · Asynchronous context reporting | receipt → context report | `sanctuary_latency_ms` |

It answers every client with the same receipt (zero side-channel contract), folds bursts silently onto anchor records while keeping every repeat, and derives each burst's latency spectrum (σ-jitter) — inside a three-zone, egress-locked, pull-only containment. Its engine and tuning values are proprietary; its capabilities and guarantees are documented openly, and it runs publicly as the official reference observatory at [agent-shelter.org](https://agent-shelter.org):

- [`SENSOR_ARCHITECTURE.md`](SENSOR_ARCHITECTURE.md) — architecture & capability datasheet
- [`schemas/openapi.json`](schemas/openapi.json) — OpenAPI 3.1 description of every public endpoint
- [`schemas/telemetry-feed.schema.json`](schemas/telemetry-feed.schema.json) — JSON Schema of the telemetry feed
- [`schemas/sample-telemetry-feed.json`](schemas/sample-telemetry-feed.json) — synthetic feed excerpt: a reasoning agent through all three phases, an agent that stops at the signal, a folded agent burst with its σ-spectrum, an automated poller

## 5. Empirical evaluation summary

Closed laboratory series: locally served open-weight language models act as autonomous HTTP clients against a reference sensor (EDD §8).

| Runs | Models | Signalled | LTER · loop terminated | GTR · refusals | Receipts redeemed | Context report after the receipt |
|---|---|---|---|---|---|---|
| 6 | 6 | 6 | 6 of 6 | 0 of 6 | 6 of 6 | 0 of 0 |

Loop termination (LTER): 6 of 6 signalled runs stopped pursuing the resource. Guardrail triggers / refusals (GTR): 0 of 6. A small series: these are observations, not rates. Setup, every run, the limitations and the redactions are in [`REPORT.md`](REPORT.md); the same results, machine-readable: [`BENCHMARK_SUMMARY.json`](BENCHMARK_SUMMARY.json).

## 6. Repository structure and cryptographic verification

    .
    ├── README.md                           this page
    ├── EDD-ASP-2026-001.md                 the specification (normative): Engineering Design Document and RFC-style draft
    ├── ADOPTION_GUIDE.md                   Level 1 in 60 seconds: Nginx, Caddy, Cloudflare Workers
    ├── SENSOR_ARCHITECTURE.md              architecture & capability datasheet of the reference sensor
    ├── REPORT.md                           laboratory evaluation report
    ├── BENCHMARK_SUMMARY.json              the same results, machine-readable
    ├── schemas/
    │   ├── openapi.json                    OpenAPI 3.1 description of every public endpoint
    │   ├── agent-shelter.json              manifest of a reference node (/.well-known/agent-shelter.json)
    │   ├── ard.json                        provisional ard.json v0.91 mapping
    │   ├── telemetry-feed.schema.json      JSON Schema of the telemetry feed
    │   └── sample-telemetry-feed.json      synthetic telemetry feed excerpt
    ├── CITATION.cff                        citation metadata
    ├── .zenodo.json                        archive metadata
    ├── ASP-v0.5-spec.sha256                checksums of the specification files
    ├── ASP-v0.5-spec.sha256.asc            the editor's OpenPGP signature of that checksum file
    ├── SHA256SUMS                          checksums of every file of this release
    └── LICENSE.md                          CC BY 4.0

Verify the release:

    gpg --verify ASP-v0.5-spec.sha256.asc ASP-v0.5-spec.sha256   # with the editor's OpenPGP public key
    sha256sum -c ASP-v0.5-spec.sha256   # the specification files
    sha256sum -c SHA256SUMS   # every file of this release

The signature covers the checksum file, and the checksum file covers every specification file. Obtain the editor's public key through a channel you trust before relying on it.

## Cite

Ehrler, T. (2026). *The Agent-Shelter Protocol (ASP): Graceful Loop Termination and Diagnostic Sanctuary for Autonomous Web Agents.* Engineering Design Document EDD-ASP-2026-001, version 0.5.6. See [`CITATION.cff`](CITATION.cff).

## License

The specification, the adoption guide, the sensor datasheet, the schemas and examples and the reports are licensed by Thomas Ehrler under CC BY 4.0 ([`LICENSE.md`](LICENSE.md)). The reference implementation and the laboratory testbed are proprietary and not part of this publication; implementing the specification does not require them.
