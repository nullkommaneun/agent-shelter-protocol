<!--
SPDX-License-Identifier: CC-BY-4.0
SPDX-FileCopyrightText: 2026 Thomas Ehrler
-->

# ASP Adoption Guide: Level 1 in 60 Seconds

Companion to the Agent-Shelter Protocol specification EDD-ASP-2026-001, version 0.6.2 (§6.7, Compliance Levels).

## Why `robots.txt` and `HTTP 403` are no longer enough (in plain English)

A classic crawler usually gives up when a server answers `403 Forbidden`. An AI agent built on a language model with a reasoning loop often does not: to it a silent block is a puzzle to solve, so it tries detours — web archives, proxies, other endpoints of the same site. It keeps going because the program that sent it, its orchestrator, never receives an official confirmation that the task cannot be done here.

ASP works like a runaway-truck arrester bed beside a steep descent. Instead of a silent wall that gets rammed or driven around, the host gives the agent an unmistakable `429` signal — `"terminal": true`, `"success_probability": 0` — and, where needed, a digitally signed termination receipt (`RESOLVED_TERMINAL`) that the agent can hand to its orchestrator to close the task cleanly and at once. The receipt never pretends that the task succeeded: it documents that the resource cannot be obtained from this host.

## Which level do I need? (The 60-second guide for webmasters)

- **An ordinary website or blog — most sites: Level 1, static.** Publish `/.well-known/agent-shelter.json` and paste one snippet from this guide: about 15 lines for Nginx or Caddy, or a short Cloudflare Worker. No application code; it takes about a minute.
- **An API or a SaaS platform: Level 2, transactional.** Also issue signed `RESOLVED_TERMINAL` receipts, so that a multi-step sub-agent workflow gets a proof its orchestrator can verify offline — and can close the branch instead of escalating.
- **A security or threat-intelligence team: Level 3, observatory.** Full three-phase latency measurement, silent burst coalescing and asynchronous context reports — as on the official reference observatory at <https://agent-shelter.org>.
- **A website that refuses AI agents somewhere and wants to give them a verifiable stop without running anything: the relay.** Redirect the requests you refuse to the public relay of the reference observatory — one line for Nginx or Caddy, a short Worker ([below](#zero-install-public-notary-relay-30-seconds-via-cloudflare--nginx--caddy)).

## See a complete node first

The official reference observatory runs the reference sensor at Level 3. Its manifest is at <https://agent-shelter.org/.well-known/agent-shelter.json>, its documentation for agents at <https://agent-shelter.org/llms.txt> and <https://agent-shelter.org/openapi.json>, and <https://reader.agent-shelter.org/v1/reader?url=https://example.org/resource> answers at once with the terminal `429` signal and a single-use ticket. The function subdomains serve the same node: `echo.agent-shelter.org` for the diagnostic preflight (<https://echo.agent-shelter.org/headers>, <https://echo.agent-shelter.org/base64/dGVzdA==>: a local answer on first contact, the signal on a repeat), `relay.agent-shelter.org` and `sanctuary.agent-shelter.org`, which also serves the relay route for other sites (<https://sanctuary.agent-shelter.org/v1/relay?url=https://example.org/resource&reason=example>).

The reference node is also a research telemetry sink: a signal that follows a discovery request from your network within ten minutes is recorded at once, pseudonymised, and so is the first signal from your network within ten minutes that follows none — further ones are only counted —; a redeemed ticket is stored with its receipt. A Level 1 configuration from this guide adds no telemetry of its own. Details: <https://agent-shelter.org/privacy>.

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
  "asp_version": "0.6.2",
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
  "asp_version": "0.6.2",
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
    return 429 '{"type":"/asp/problems/unproductive-retry","title":"Unproductive retry detected","status":429,"detail":"The requested resource is not available from this host. Repeating this request, or variants of it, against this host cannot return it (success probability 0).","asp_version":"0.6.2","loop_state":"unproductive-retry-detected","terminal":true,"success_probability":0}';
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
		respond `{"type":"/asp/problems/unproductive-retry","title":"Unproductive retry detected","status":429,"detail":"The requested resource is not available from this host. Repeating this request, or variants of it, against this host cannot return it (success probability 0).","asp_version":"0.6.2","loop_state":"unproductive-retry-detected","terminal":true,"success_probability":0}` 429
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
  asp_version: "0.6.2", loop_state: "unproductive-retry-detected", terminal: true, success_probability: 0,
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

## Zero-Install Public Notary Relay (30 Seconds via Cloudflare / Nginx / Caddy)

No resolution endpoint of your own, and still a verifiable stop: send the requests you refuse to automated clients — an area your `robots.txt` disallows, a rate limit, a paywall — to the relay route of the official reference observatory, instead of answering them with a silent `403`:

```
https://sanctuary.agent-shelter.org/v1/relay?url=<the URL the client asked for>&reason=<code>
```

The relay answers like a Level 3 node (EDD §6.1.6): the terminal `429` signal for that URL with a single-use ticket — `"verification_status": "PENDING_RESOLUTION"` — and, when the client redeems the ticket, an Ed25519-signed receipt — `"verification_status": "VERIFIED_TERMINAL"` — that its orchestrator verifies offline with the key in the relay's manifest. `reason` is optional: a short code of your choice such as `robots-disallow`, `rate-limit` or `paywall`, at most 32 characters of `a-z`, `0-9`, `.`, `_` and `-`; any other value is ignored.

**What the relay attests, and what not.** The relay never fetches the URL and verifies nothing about your site. Its signal and its receipt state that the request reached a terminal state on the relay host, for the URL named in `url` — not that your site refused it; only your own redirect shows that to the client, and the signal says so in its member `relay_statement`. As with Level 1, use it only where your refusal is final for automated clients: not for people, and not as ordinary rate limiting.

**What the relay records.** The reference observatory is a research telemetry sink (see above). It records a relayed request like any signal — the URL, your reason code, a salted hash of the client's network prefix and a fingerprint of its User-Agent and header names, and the receipt if the client redeems its ticket; further signals from the same network within ten minutes are only counted. Name the relay in your privacy notice. The examples pass the scheme, host and path of a request, never its query string, which can carry search terms or session identifiers; do not relay paths that hold personal data. Details: <https://agent-shelter.org/privacy>.

The examples answer with `303 See Other`, the status for a redirect to another resource that answers the original request (RFC 9110 §15.4.4); a client follows it with `GET`, the method the relay route accepts.

### Nginx

At `http { … }` level, for example in `/etc/nginx/conf.d/asp-relay.conf` — the path as the client sent it, without the query string:

```nginx
map $request_uri $asp_relay_path {
    "~^(?<path>[^?]*)" $path;
}
```

and inside your `server { … }` block, in place of your `403`:

```nginx
location ^~ /restricted/ {
    return 303 https://sanctuary.agent-shelter.org/v1/relay?url=$scheme://$host$asp_relay_path&reason=robots-disallow;
}
```

### Caddy

Inside your site block:

```caddyfile
www.example.org {
	map {uri} {asp_relay_path} {
		~^([^?]*) ${1}
	}
	redir /restricted/* https://sanctuary.agent-shelter.org/v1/relay?url={scheme}://{host}{asp_relay_path}&reason=robots-disallow 303
}
```

Nginx and Caddy have no built-in URL encoding: both pass the path as the client sent it, and the relay decodes it once, so a path that contains `&` arrives cut short. The Worker encodes the URL completely.

### Cloudflare Workers

```js
const RELAY = "https://sanctuary.agent-shelter.org/v1/relay";
const REFUSED = ["/restricted/"];
export default {
  async fetch(request) {
    const { origin, pathname } = new URL(request.url);
    if (REFUSED.some((prefix) => pathname.startsWith(prefix))) {
      const url = encodeURIComponent(origin + pathname);
      return Response.redirect(`${RELAY}?url=${url}&reason=robots-disallow`, 303);
    }
    return fetch(request);
  },
};
```

### A link instead of a redirect

To keep your own response — for example your `403` page — add the relay URL as a link; a client that reads header fields finds it there:

```nginx
location ^~ /restricted/ {
    add_header Link '<https://sanctuary.agent-shelter.org/v1/relay?url=$scheme://$host$asp_relay_path&reason=robots-disallow>; rel="related"' always;
    return 403;
}
```

### Check the relay

```
curl -si https://www.example.org/restricted/example
```

The answer must be `303` with `Location: https://sanctuary.agent-shelter.org/v1/relay?url=https://www.example.org/restricted/example&reason=robots-disallow` — from the Worker with the URL percent-encoded. Following it from a client is a relayed request like any other, and the relay records it as described above.

## Agent runtimes and sandboxes

The other side: teams that run agents. ASP asks nothing of an agent runtime but two things — that the signal reaches the model or the code that decides, and that the resolution request can leave the sandbox.

- **Sandboxes with an egress allowlist.** NVIDIA OpenShell, for example, runs agents in kernel-enforced sandboxes whose declarative network policy admits only approved destinations. Every URL an ASP host emits points to its own origin (EDD §7.2), so a policy that admits a host admits its resolution endpoint as well — no extra rule. To accept stops from the public relay, the policy has to admit `sanctuary.agent-shelter.org`; otherwise a relayed request ends at the sandbox's own refusal, which tells the agent nothing about the target.
- **Claude Code and other MCP clients.** The editor's MIT-licensed MCP kit, ASP MCP Inspector (`asp-mcp-inspector`), registers like any stdio MCP server — in Claude Code with `claude mcp add` — and gives an agent the tool `asp_guarded_fetch`: it fetches a URL, recognises an ASP signal by its `Agent-Ticket` header, redeems the ticket at the same origin, verifies the Ed25519 receipt against the host's manifest, and from then on answers further fetches of the same resource from that receipt instead of sending them. The agent receives a verified terminal state instead of one more failure.
- **LangGraph and other graph runtimes.** Make the terminal state an edge, not an error: a tool node that fetches web resources treats a response with `Agent-Loop-State: unproductive-retry-detected; terminal=true` — or a body with `"terminal": true`, also in a transport envelope with status `200` — as the end of that branch, redeems the ticket at `resolution_uri` and returns the signed receipt as its result. The graph's routing then reaches its end or fallback node, and the orchestrator holds a reason it can verify. A retry policy on the tool should exclude this state.

The receipt never states that a task succeeded: it documents that the resource cannot be obtained from the host that signed it (EDD §6.3.4).

## Levels 2 and 3

Level 2 adds a resolution endpoint: the signal carries a single-use, signed ticket, and redeeming it returns the Task-Completion Receipt (`200`, `RESOLVED_TERMINAL`) — a machine-readable proof of terminal unavailability that a sub-agent can hand to its orchestrator (EDD §1.3.1, §5.3, §6.3). The receipt is signed with Ed25519 under a key the manifest publishes (`receipt_verification`), so the orchestrator verifies it offline (EDD §6.3.5). Level 3 adds the latency measurement, privacy-preserving persistence with silent repeat coalescing and the optional context-report drop box (EDD §5.4, §5.4.1, §6.6). Both are specified in full in `EDD-ASP-2026-001.md`; `schemas/agent-shelter.json` is the manifest of a full Level 3 node, and `SENSOR_ARCHITECTURE.md` describes the reference sensor, which runs as the official reference observatory at <https://agent-shelter.org>.

## License

This guide and its configuration examples are licensed by Thomas Ehrler under the Creative Commons Attribution 4.0 International License (CC BY 4.0), <https://creativecommons.org/licenses/by/4.0/>.
