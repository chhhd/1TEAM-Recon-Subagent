---
name: recon-endpoint-mapping
description: Procedure for mapping a web target's attack surface — crawling, form/link extraction, parameter classification, and producing a RequestSeed inventory plus a human-readable Attack Surface table. Use before any injection or access-control testing so downstream agents work from one shared inventory instead of rediscovering endpoints themselves.
---

# Endpoint Mapping Procedure

## 1. Run the harness crawler first
`dast-harness/dast_harness/agent_kit/recon.py` already crawls, parses `<a>`/`<form>`, and normalizes everything into `RequestSeed`s. This is the baseline — don't hand-roll a crawler.

```bash
PYTHONPATH=dast-harness python3 -m dast_harness.agent_kit.recon <base-url>
```

## 2. Cross-check against source (when available)
Grep the app's route definitions (`@app.get`, `@app.post`, `app.route`, framework-equivalent) and diff against what the crawler found. Anything in source but not crawled is either:
- **Unlinked but reachable** — no navigation link points to it, but it responds. This is worth flagging on its own (unlinked admin/upload endpoints are a common real finding, not just a coverage note).
- **Method-gated** — e.g. only responds to POST, so a GET-only crawl won't surface it as "working," only as a 405. Note the seed anyway; the relevant agent will hit it correctly.

## 3. Classify every parameter by location — then by what it *represents*
Location tells each agent *how* to manipulate the parameter; it does not by itself decide *who* should test it. Don't conflate the two — an early version of this skill routed purely on location and missed `?id=`-style object references because they weren't in `path`. Two passes:

**Pass A — location determines the manipulation technique:**

| `location` | What it means | Manipulation |
|---|---|---|
| `query` | value in the URL query string | value substitution |
| `body` | value in the POST body (form or JSON) | value substitution |
| `path` | value embedded in the URL path itself | segment swap |
| `header` | value read from a request header | value/role substitution |
| `cookie` | value read from a cookie | usually session — note but don't auto-route |

**Pass B — semantics determine the candidate class, independent of location:**

| Parameter looks like | Candidate class | Why location doesn't matter here |
|---|---|---|
| An id/reference to a specific object, account, or order (`id`, `user_id`, `order_no`, a UUID) | access-control-agent | Whether it's `/orders/{id}` or `?id=2`, the question is identical: does the server check the requester owns it? |
| A free-text value that flows into a search/filter/lookup (`q`, `search`, `query`, `name`) | injection-agent | The question is whether the value reaches an interpreter, which is orthogonal to where it's carried |
| A role/permission-flavored value (`role`, `is_admin`, a header the server might trust) | access-control-agent | Vertical privilege escalation candidate regardless of transport |

A parameter can land in both tables (e.g. a path param can also be injectable if reflected into a query, or a query `id` param is squarely access-control even though Pass A says "value substitution" like an injection candidate). When in doubt, list it under both candidate classes in the Attack Surface table and let the orchestrator decide — don't silently pick one, and don't let Pass A's technique classification override Pass B's semantic one.

## 4. Record auth requirement per endpoint
Send one unauthenticated probe per endpoint (via the harness client, `actor="anon"`) and note the status:
- 401/403 → auth required, and enforced
- 200 with no session → open by design, or missing auth (can't tell which from recon alone — flag for the downstream agent to determine, don't guess)

## 5. Note observed status/content-type
This becomes the "baseline" the injection/access-control agents will replay and compare against later — capture it accurately, don't approximate.

## 6. Produce the Attack Surface table
See the exact format in `recon-agent`'s agent definition — keep the columns identical across runs so diffs between recon passes are mechanical (used for the "no new seeds → stop" signal in `CLAUDE.md`).

## Common mistakes
- Reshaping `RequestSeed` into a custom format before handing off — downstream agents expect the contract shape verbatim.
- Guessing at vulnerability classes ("this looks like SQLi") — recon inventories and hints, it does not conclude. Leave the verdict to the specialist agent.
- Re-crawling a target from scratch every time instead of diffing against the last known seed set.
