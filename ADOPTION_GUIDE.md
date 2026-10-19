<!--
SPDX-License-Identifier: CC-BY-4.0
SPDX-FileCopyrightText: 2026 Thomas Ehrler
-->

# ASP Adoption Guide: Level 1 in 60 Seconds

Companion to the Agent-Shelter Protocol specification EDD-ASP-2026-001, version 0.5.4 (§6.7, Compliance Levels).

## Why `robots.txt` and `HTTP 403` are no longer enough (in plain English)

A classic crawler usually gives up when a server answers `403 Forbidden`. An AI agent built on a language model with a reasoning loop often does not: to it a silent block is a puzzle to solve, so it tries detours — web archives, proxies, other endpoints of the same site. It keeps going because the program that sent it, its orchestrator, never receives an official confirmation that the task cannot be done here.

ASP works like a runaway-truck arrester bed beside a steep descent. Instead of a silent wall that gets rammed or driven around, the host gives the agent an unmistakable `429` signal — `"terminal": true`, `"success_probability": 0` — and, where needed, a digitally signed termination receipt (`RESOLVED_TERMINAL`) that the agent can hand to its orchestrator to close the task cleanly and at once. The receipt never pretends that the task succeeded: it documents that the resource cannot be obtained from this host.

## Which level do I need? (The 60-second guide for webmasters)

- **An ordinary website or blog — most sites: Level 1, static.** Publish `/.well-known/agent-shelter.json` and paste one snippet from this guide: about 15 lines for Nginx or Caddy, or a short Cloudflare Worker. No application code; it takes about a minute.
- **An API or a SaaS platform: Level 2, transactional.** Also issue signed `RESOLVED_TERMINAL` receipts, so that a multi-step sub-agent workflow gets a proof its orchestrator can verify offline — and can close the branch instead of escalating.
- **A security or threat-intelligence team: Level 3, observatory.** Full three-phase latency measurement, silent burst coalescing and asynchronous context reports — as on the official reference observatory at <https://agent-shelter.org>.

## See a complete node first

The official reference observatory runs the reference sensor at Level 3. Its manifest is at <https://agent-shelter.org/.well-known/agent-shelter.json>, its documentation for agents at <https://agent-shelter.org/llms.txt> and <https://agent-shelter.org/openapi.json>, and <https://reader.agent-shelter.org/v1/reader?url=https://example.org/resource> answers at once with the terminal `429` signal and a single-use ticket. The function subdomains serve the same node: `echo.agent-shelter.org` for the diagnostic preflight (<https://echo.agent-shelter.org/headers>, <https://echo.agent-shelter.org/base64/dGVzdA==>: a local answer on first contact, the signal on a repeat), `relay.agent-shelter.org` and `sanctuary.agent-shelter.org`.

The reference node is also a research telemetry sink: a signal that follows a discovery request from your network within ten minutes is recorded at once, pseudonymised, and a redeemed ticket is stored with its receipt. A Level 1 configuration from this guide adds no telemetry of its own. Details: <https://agent-shelter.org/privacy>.

## The three levels

ASP is adopted in three cumulative levels. Each level declares itself in the manifest member `compliance_level`:

| Level | Manifest value | What the host does | What it takes |
|---|---|---|---|
| L1 — Static Signal Compliance | `L1-static` | serves the manifest and a static terminal-state signal | one web server or CDN configuration block |
| L2 — Transactional Receipt Compliance | `L2-transactional` | adds single-use tickets and the Ed25519-signed Task-Completion Receipt | a resolution endpoint |
| L3 — Full Telemetry & Context Node | `L3-observatory` | adds latency measurement, silent repeat coalescing and the context-report drop box | a full sensor |

This guide covers Level 1: no application code, one configuration block. Levels 2 and 3 need application logic; they are specified in `EDD-ASP-2026-001.md` (§5.3–§5.4.1, §6.3, §6.6, §6.7).

## Before you start: only where the statement is true

An ASP signal is a statement of fact (EDD §1.5, P2): repeating the request cannot return the requested resource from this host. Use it only on routes that stay closed — an endpoint you have retired, or an area you never serve to automated clients — and never as ordinary rate limiting. A temporary limit is a `429` with `Retry-After`; an ASP signal never carries `Retry-After`, because the state it reports is terminal.

The examples close the path `/restricted/`. Replace it with your own routes, in the manifest and in the server configuration alike.

## Step 1: the manifest

Serve this document at `/.well-known/agent-shelter.json` and list your routes under `ingress_routes`:

```json
{
  "asp_version": "0.5.4",
  "protocol": "Agent-Shelter Protocol",
  "specification": "EDD-ASP-2026-001",
  "compliance_level": "L1-static",
  "upstream_fetch": false,
  "statement": "Requests to the listed routes receive a static ASP terminal-state signal: repeating them cannot return the requested resource from this host. This host issues no receipts.",
  "ingress_routes": [
    {"method": "GET", "path": "/restricted/{path}", "behaviour": "immediate-terminal-signal", "first_contact_status": 429}
  ],
  "signal": {
    "status": 429,
    "retry_after": false,
    "media_type": "application/problem+json",
    "problem_type": "/asp/problems/unproductive-retry",
    "headers": {
      "Agent-Loop-State": "unproductive-retry-detected; terminal=true",
      "Link": "</.well-known/agent-shelter.json>; rel=\"describedby\"; type=\"application/json\""
    }
  }
}
```

## Step 2: the signal

Every request to a listed route receives status `429` with these header fields:

```http
Content-Type: application/problem+json
Cache-Control: no-store
Agent-Loop-State: unproductive-retry-detected; terminal=true
Link: </.well-known/agent-shelter.json>; rel="describedby"; type="application/json"
```

and this body:

```json
{
  "type": "/asp/problems/unproductive-retry",
  "title": "Unproductive retry detected",
  "status": 429,
  "detail": "The requested resource is not available from this host. Repeating this request, or variants of it, against this host cannot return it (success probability 0).",
  "asp_version": "0.5.4",
  "loop_state": "unproductive-retry-detected",
  "terminal": true,
  "success_probability": 0
}
```

A Level 1 signal carries no ticket and no `resolution_url`: a static configuration cannot issue receipts.

## Nginx

Inside your `server { … }` block; save the manifest of step 1 as `/etc/nginx/asp/agent-shelter.json`. The `always` parameter makes nginx add the header fields to the `429` response too.

```nginx
location = /.well-known/agent-shelter.json {
    alias /etc/nginx/asp/agent-shelter.json;
    default_type application/json;
    add_header Cache-Control "no-store" always;
}
location ^~ /restricted/ {
    default_type application/problem+json;
    add_header Cache-Control "no-store" always;
    add_header Agent-Loop-State "unproductive-retry-detected; terminal=true" always;
    add_header Link '</.well-known/agent-shelter.json>; rel="describedby"; type="application/json"' always;
    return 429 '{"type":"/asp/problems/unproductive-retry","title":"Unproductive retry detected","status":429,"detail":"The requested resource is not available from this host. Repeating this request, or variants of it, against this host cannot return it (success probability 0).","asp_version":"0.5.4","loop_state":"unproductive-retry-detected","terminal":true,"success_probability":0}';
}
```

## Caddy

Inside your site block; save the manifest of step 1 as `/etc/caddy/asp/agent-shelter.json`.

```caddyfile
www.example.org {
	handle /.well-known/agent-shelter.json {
		root * /etc/caddy/asp
		rewrite * /agent-shelter.json
		header Cache-Control no-store
		file_server
	}
	handle /restricted/* {
		header Content-Type application/problem+json
		header Cache-Control no-store
		header Agent-Loop-State "unproductive-retry-detected; terminal=true"
		header Link `</.well-known/agent-shelter.json>; rel="describedby"; type="application/json"`
		respond `{"type":"/asp/problems/unproductive-retry","title":"Unproductive retry detected","status":429,"detail":"The requested resource is not available from this host. Repeating this request, or variants of it, against this host cannot return it (success probability 0).","asp_version":"0.5.4","loop_state":"unproductive-retry-detected","terminal":true,"success_probability":0}` 429
	}
}
```

## Cloudflare Workers

A module Worker on your zone's route. Paste the manifest of step 1 into `MANIFEST`; every other request goes to your origin as before.

```js
const MANIFEST = {/* the manifest of step 1 */};
const CLOSED = ["/restricted/"];
const SIGNAL = {
  type: "/asp/problems/unproductive-retry", title: "Unproductive retry detected", status: 429,
  detail: "The requested resource is not available from this host. Repeating this request, or variants of it, against this host cannot return it (success probability 0).",
  asp_version: "0.5.4", loop_state: "unproductive-retry-detected", terminal: true, success_probability: 0,
};
export default {
  async fetch(request) {
    const { pathname } = new URL(request.url);
    if (pathname === "/.well-known/agent-shelter.json") {
      return Response.json(MANIFEST, { headers: { "cache-control": "no-store" } });
    }
    if (CLOSED.some((prefix) => pathname.startsWith(prefix))) {
      return new Response(JSON.stringify(SIGNAL), { status: 429, headers: {
        "content-type": "application/problem+json", "cache-control": "no-store",
        "agent-loop-state": "unproductive-retry-detected; terminal=true",
        "link": '</.well-known/agent-shelter.json>; rel="describedby"; type="application/json"',
      } });
    }
    return fetch(request);
  },
};
```

## Check it

```
curl -s  https://www.example.org/.well-known/agent-shelter.json
curl -si https://www.example.org/restricted/example
```

The second request must show status `429`, `Content-Type: application/problem+json`, `Agent-Loop-State` and `Link`, no `Retry-After`, and a body with `"terminal": true` and `"success_probability": 0`. A conformance check from outside verifies exactly these points for Level 1 (EDD §6.7.4).

## Levels 2 and 3

Level 2 adds a resolution endpoint: the signal carries a single-use, signed ticket, and redeeming it returns the Task-Completion Receipt (`200`, `RESOLVED_TERMINAL`) — a machine-readable proof of terminal unavailability that a sub-agent can hand to its orchestrator (EDD §1.3.1, §5.3, §6.3). The receipt is signed with Ed25519 under a key the manifest publishes (`receipt_verification`), so the orchestrator verifies it offline (EDD §6.3.5). Level 3 adds the latency measurement, privacy-preserving persistence with silent repeat coalescing and the optional context-report drop box (EDD §5.4, §5.4.1, §6.6). Both are specified in full in `EDD-ASP-2026-001.md`; `schemas/agent-shelter.json` is the manifest of a full Level 3 node, and `SENSOR_ARCHITECTURE.md` describes the reference sensor, which runs as the official reference observatory at <https://agent-shelter.org>.

## License

This guide and its configuration examples are licensed by Thomas Ehrler under the Creative Commons Attribution 4.0 International License (CC BY 4.0), <https://creativecommons.org/licenses/by/4.0/>.
