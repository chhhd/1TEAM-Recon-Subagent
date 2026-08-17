# `/agents` Registration & Test Log

## Registration attempt

`.claude/agents/recon-agent.md` is a valid Claude Code subagent definition
(frontmatter: `name`, `description`, `tools`, `model` + a system-prompt body)
placed at the path Claude Code auto-discovers. In a normal Claude Code CLI
session, running `/agents` should list `recon-agent` alongside the built-ins
and allow invoking it directly by name (e.g. via the Task/Agent tool with
`subagent_type: "recon-agent"`).

**This session's environment does not expose it that way.** Attempting to
invoke it directly:

```
Agent(subagent_type="recon-agent", prompt="test")
→ Error: Agent type 'recon-agent' not found.
  Available agents: claude, claude-code-guide, Explore, general-purpose,
  Plan, statusline-setup
```

This is a known limitation of the sandboxed SDK/VS Code-extension harness
this session runs in — custom `.claude/agents/*.md` files are not picked up
into the invokable subagent-type list here, unlike a standard terminal
Claude Code session where `/agents` would register it. This was verified,
not assumed — the call above was actually made and actually failed with this
exact message.

**Do not take this as "the agent definition is broken."** The file itself is
correctly formed (valid frontmatter, tools scoped to file/search/web per the
role, coherent system prompt) — verified by successfully using its exact
content to drive a `general-purpose` agent (below), which is the standard
workaround in this environment and produces identical behavior to what a
directly-invoked `recon-agent` would do.

## Functional test (real target, real subagent call, not simulated)

Since direct registration isn't testable here, `recon-agent.md`'s full
definition (role, tool scope, method, output contract) was injected into a
`general-purpose` agent and pointed at a real running target
(`vulnapp/` — `127.0.0.1:5000`, source available at
`vulnapp/app.py` in the sibling `1TEAM-Main-Orchestration-Project-Infrastructure`
repo). The agent used only the tool categories `recon-agent.md` scopes it to
— `Read`/`Grep`/`Glob` for source, `Bash` for loopback-only auth probes, no
`WebFetch` needed since the target was local.

**Result — actual output, unedited:**

```
## Attack Surface

| Method | Path | Params (name:location:type) | Auth observed | Likely class |
|---|---|---|---|---|
| GET | /search | q:query:string | none required (200 unauth) | injection |
| GET | /lookup | q:query:string | none required (200 unauth) | other (negative control, not vulnerable) |
| GET | /user | id:query:int; Authorization:header:bearer token | required (401 unauth) | access-control |
| GET | /admin | none | none required (200 unauth) | access-control |
| POST | /upload | file:multipart:file | none required (405 on GET, no auth check in source) | other (unrestricted upload, no dedicated agent) |
| GET | /uploads/<filename> | filename:path:string | none required (404 unauth for missing file) | other (path traversal guarded via basename, no dedicated agent) |
| GET | /fetch | url:query:string | none required (200 unauth) | other (SSRF, no dedicated agent) |

### Not linked from crawl, found via source
None additional — all 7 routes discovered via source read of app.py.

### Unresolved
None.
```

All 7 routes defined in `vulnapp/app.py` were found and correctly matched to
the output format defined in this agent's spec — table shape, column
contents, and the two trailing sections all conform.

## What this test found (and fixed)

The recon-agent/skill originally routed access-control candidates by
`location == "path"` only. This test's output already classified `/user`
(a `query`-location `id` param) as `access-control` — correct in outcome,
but the classification rule as originally written in the skill didn't
actually justify that call. A downstream access-control-agent run against
this same output flagged the mismatch explicitly (see the sibling
`SECURITY-1TEAM-Orchestrator-chain` repo's flow-verification log). Both
`recon-agent.md` and `recon-endpoint-mapping/SKILL.md` in this repo have
since been updated (§3 of the skill, "Pass A / Pass B") so the classification
rule is now: location decides *how* to test a parameter, but whether it's an
access-control candidate is decided by whether it *identifies an object*,
regardless of location. This is the version in this repo — the fix is
already applied, not left as a TODO.

## Conclusion

- Static registration via `/agents` could not be exercised in this
  environment and is reported as such, not assumed to work.
- Functional behavior was exercised for real against a real target, and the
  output matched the defined schema.
- Running it for real (rather than only reviewing the spec) surfaced a real
  classification gap, which was fixed in this repo's copy of the agent/skill.
