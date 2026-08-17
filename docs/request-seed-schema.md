# Standard Output Schema — Recon → Downstream Agents

Recon's job is to produce **one shared inventory** that injection-agent and
access-control-agent both consume without needing to re-discover anything
themselves. This document is the schema contract for that handoff. It is not
invented for this repo — it mirrors, field-for-field,
`dast-harness/dast_harness/agent_kit/contract.py` (the team's shared harness),
so recon's output is directly usable by an agent implementing that contract,
not just by another Claude Code subagent reading prose.

## 1. `RequestSeed` — the primary handoff artifact

One seed = one **real, replayable request**, not just an endpoint shape.
Downstream agents replay it and change exactly one thing.

| Field | Type | Meaning |
|---|---|---|
| `method` | `str` | HTTP method, e.g. `"GET"` |
| `url` | `str` | Absolute URL — sendable as-is |
| `params` | `tuple[RequestParameter, ...]` | See §2 below |
| `body_content_type` | `str` | e.g. `"application/x-www-form-urlencoded"` for form POSTs |
| `auth_required` | `bool \| None` | `None` = not yet determined; recon should resolve this via one unauthenticated probe per endpoint |
| `observed_status` | `int \| None` | Status recon actually saw; `None` means recon never sent this request (e.g. a POST-only route found via source, not crawled) |
| `observed_content_type` | `str` | Response `Content-Type` recon observed |
| `source` | `str` | One of `"seed"` / `"link"` / `"form"` / `"robots.txt"` / `"guess"` — where the seed came from |
| `template` (property) | `str` | `/api/orders/{id}` — path params folded back to placeholders, so seeds for the same endpoint group together instead of one-per-id |

## 2. `RequestParameter` — one parameter on a seed

| Field | Type | Meaning |
|---|---|---|
| `name` | `str` | Parameter name |
| `location` | `str` | One of `query` / `body` / `path` / `header` / `cookie` — **closed vocabulary, do not invent new values** |
| `value` | `str` | The *observed* value — this becomes the injection/IDOR baseline. Recording the real value, not a placeholder, is what lets downstream agents diff against a known-good baseline |
| `type` | `str` | One of `string` / `int` / `float` / `bool` / `json` |
| `json_path` | `str` | Only meaningful when `location == "body"` and the body is JSON, e.g. `"$.user.id"` |

**Why all four fields matter, not just `name`:** location decides which agent
technique applies (segment-swap vs value-swap); value is the baseline a
downstream agent's "attack" request gets compared against; type prevents a
numeric field being fed a string payload and the resulting type error being
misread as a SQL error; json_path is the only way to locate a field inside a
JSON body since name alone doesn't give a path.

## 3. Classification is separate from the schema — don't bake it in

`RequestSeed`/`RequestParameter` carry no "likely vulnerability class" field,
and recon should not invent one on the struct. Classification (injection vs
access-control candidate) is a **routing hint** layered on top, in the
human-readable Attack Surface table (§4) — not part of the machine-consumed
seed shape. Keeping these separate means the seed inventory stays reusable
even if the routing rules change later.

## 4. `ReconResult` — what `run()` actually returns

Recon's agent-level return value (`dast-harness` shape) is `ReconResult`,
which extends the common `AgentResult` (`agent`, `findings`, `coverage`,
`completion`) with one recon-specific field:

```python
request_seeds: list[RequestSeed]
```

This is **the A → B, C interface** — injection-agent and access-control-agent
take `request_seeds` as their input, filtered by `location` and semantic
class (see the recon-endpoint-mapping skill's Pass A / Pass B classification).

## 5. Attack Surface table — the human-readable companion

Alongside the raw `request_seeds`, recon also hands back a compact table for
the orchestrator and any human reviewer:

```
## Attack Surface

| Method | Path | Params (name:location:type) | Auth observed | Likely class |
|---|---|---|---|---|
| GET | /search | q:query:string | open | injection |
| GET | /user | id:query:int | required (401 without) | access-control |
| GET | /admin | — | none observed | access-control |
| POST | /upload | file:body:binary | required | other (no dedicated agent yet) |
```

Same information as the seeds, reshaped for readability — the `Likely class`
column is the routing hint from §3, kept out of the schema itself.

## 6. Validation

`dast-harness/dast_harness/agent_kit/contract.py`'s `validate_result()` /
`_seed_errors()` enforce this shape mechanically — a seed with a bad
`location`, a non-string `value`, a path param whose value isn't actually a
segment of the URL, or a duplicate `(name, location)` pair fails validation
before it ever reaches a downstream agent. Recon should run against a target
and let `Agent.finish()` (which calls `validate_result()`) catch shape
mistakes rather than trusting hand-written output.
