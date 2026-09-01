# Experimental Findings

> ℹ️ **This document is not actively maintained.** It captures a snapshot of early findings. Current discussion and decisions are tracked in the [meeting notes discussions](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/categories/meeting-notes-skills-over-mcp-wg), [Discord #skills-over-mcp-wg](https://discord.com/channels/1358869848138059966/1464745826629976084), and on the [SEP-2640 PR](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2640).

> **Contributing findings?** See [#50](https://github.com/modelcontextprotocol/experimental-ext-skills/issues/50) for the contribution template proposal.

## VS Code: MCP-served skills as agent skills (Issue #66)

**Implementation:** [tobi-oye/vscode#1](https://github.com/tobi-oye/vscode/pull/1) — on a [microsoft/vscode](https://github.com/microsoft/vscode) fork (personal exploration, not submitted upstream)

Added `skill://` discovery to VS Code and verified it against the [Hugging Face MCP server](https://github.com/huggingface/hf-mcp-server) (`https://huggingface.co/mcp`). VS Code discovered 8 skills and offered them to the model with no manual attachment.

**Findings:**

- **VS Code already had the whole progressive-disclosure loop; it just had no MCP source.** `findAgentSkills()` → `name`/`description` into a `<skills>` block → a model-facing `skill` tool. Same shape as [skillsdotnet](https://github.com/PederHP/skillsdotnet)'s `SkillCatalog`. Only the MCP origin was missing.
- **The loading half needed no code.** `mcp-resource://` is already registered with VS Code's `IFileService`, so mapping `skill://…/SKILL.md` onto it is enough — the existing skill tool reads it and each read becomes a `resources/read`. "Treat filesystem and MCP skills identically" falls out for free.
- **The index wire format had already moved past the checked-in draft.** The live server serves `{url, digest, frontmatter:{name, description}}`; the draft here specifies top-level `name`/`description` and a required `type: "skill-md"`. A parser written to the draft matches **zero** entries on a real server. Accepting both shapes is three lines.
- **Per-context-computation discovery is an accidental DoS.** `findAgentSkills()` runs on every context rebuild; discovery is two round trips per server. A naive version issued 20 index reads for one chat turn. Same shape as the incident behind hf-mcp-server's client denylist ([#164](https://github.com/huggingface/hf-mcp-server/pull/164), ~100k req/min). Cache the *promise*, keyed on connection state — not just server id, or a lookup made while a server was stopped pins an empty result.
- **Cross-server name collisions are unspecified.** Skill names are a flat namespace. The SEP ties the final URI segment to the frontmatter `name` but says nothing about two servers both serving `deploy`. Local-over-MCP is defensible; MCP-vs-MCP degenerates to discovery order. **Worth an explicit note in the SEP.**
- **Provenance reaches the UI but not the model.** The `<skills>` block emits only `<name>`, `<description>`, `<file>` — and `<file>` encodes an opaque server id. Loading is never ambiguous, but a model reasoning across servers has no legible origin signal.
- **Archives cost 17 of 25 skills.** Archive-only entries have no `url` to a `SKILL.md`, so a resource-reading host skips them. Archives were removed from SEP-2640 (decision log, 2026-07-16); until servers follow, such hosts see a fraction of what is offered.
- **Adding a skill source touched four type surfaces** that must agree (storage enum, a parallel source union, an ext-host DTO, a proposed API type). Missing two produced a runtime throw that broke chat while `tsc` stayed green.
- **Identifying skills by URL alone was enough ([#54](https://github.com/modelcontextprotocol/experimental-ext-skills/issues/54)).** `skill://` prefix plus `/SKILL.md` suffix, no `_meta` read — a two-line filter with nothing to negotiate. Because the last path segment must match the skill name, the name is readable off the URL, so a picker fills without fetching every `SKILL.md`. The tradeoff is that a URL carries no structured metadata; tags, versions or provenance would still need `_meta`.
- **The index ships a `digest` that this client ignores.** The live server sends `sha256:` per entry; nothing here verifies it, so a corrupted or swapped skill loads silently. fast-agent does check it. Worth settling whether verification is the host's job — the draft currently leans on it being "the transport's concern over an authenticated MCP connection".

**Verification:** `tsc` clean; 10 unit tests (incl. `skill://` → `mcp-resource://` round trip and index parsing against verbatim live output); 112 existing promptSyntax tests pass; discovery confirmed against the live server.

**Note:** none of the three defects above were caught by type-checking or unit tests — all surfaced only from running against a real server.

**Open:**

- **Model invocation not demonstrated.** Discovery and contribution are confirmed; no run yet shows the model choosing to load an MCP-served skill. Confounded by source builds being unable to reach the Copilot service (OSS `product.json` ships no OAuth client config) and by a small auto-routed model. Consistent with the adherence problems recorded elsewhere on this page.
- Resource templates parsed but not materialized (need the completion API).
- No `resources/subscribe`, so mid-session skill updates are missed.

## VS Code: SEP-2640 v1 detection over `skills/list` (Issue #66, follow-up)

**Implementation:** [tobi-oye/vscode#1](https://github.com/tobi-oye/vscode/pull/1) (detection) and [#3](https://github.com/tobi-oye/vscode/pull/3) (manifest verification) — same fork as the entry above, re-run against v1 rather than the pre-v1 draft.

**Server:** [olaservo/skills-over-mcp-demo](https://huggingface.co/spaces/olaservo/skills-over-mcp-demo) on a Hugging Face Space, `@olaservo/ext-skills` 0.13.0, tracking SEP-2640 at [`753b9f2`](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2640/commits/753b9f2). Streamable HTTP, negotiating `2025-11-25` (VS Code's `LATEST_PROTOCOL_VERSION`).

The earlier entry predates `skills/list`. This one exercises the v1 surface end to end, with one query chosen so no server-side tool could answer it.

**Walkthrough — "what does 5d10dh1 mean?"**

Detection, once per session:

```
06:48:27  [editor -> server] {"method":"skills/list"}
06:48:28  [mcp-skills] "ola-skills" served 3 skill(s): tabletop-dice, mcp-glossary, release-notes-writer
```

`secret-menu` is correctly absent — it is served but unlisted. The model received `name` and `description` only; no skill body was fetched at connect, at listing, or at contribution time.

Retrieval, two hops, only once the model chose to load:

```
10:03:34  resources/read skill://dice-roller/tabletop-dice/SKILL.md
10:03:38  resources/read skill://dice-roller/tabletop-dice/references/dice-notation.md
```

The first read returns a body containing *"see `references/dice-notation.md` for the full grammar"*; the model followed that pointer to the second. The answer — `5d10` rolls five ten-sided dice, `dh1` drops the highest one — comes from the reference file's Keep/Drop table, not from the `SKILL.md`. Both files were verified against the `{uri, digest, size}` manifest carried in the `skills/list` entry from 06:48.

**Findings:**

- **Detection and retrieval separate cleanly in a real host.** The `skills/list` entry carried everything needed to verify content that had not been fetched yet, and nothing was fetched until the model loaded the skill. The on-demand retrieval requirement ([`72cc599`](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2640/commits/72cc599)) is implementable without a separate prefetch path — the host simply never has one.
- **Two-hop reference resolution needed no protocol support.** The relative path inside `SKILL.md` resolved against the skill's `mcp-resource://` base through the existing file service. A skill's supporting files cost nothing extra to reach.
- **A server-side tool out-competes the skill it is meant to pair with.** The same deployment ships a `roll_dice` tool whose description overlaps `tabletop-dice` almost verbatim. Asked to "roll 2d6+3", the model called the tool and never loaded the skill — with three skills in context and a `BLOCKING REQUIREMENT: … load the relevant skill(s) … as your first action` instruction present and ignored. The tool exists because tool-centric hosts need a `tools/list` surface, and the server's README states it "pairs with the `tabletop-dice` skill without substituting for it"; in this host it substitutes. Only a query the tool provably could not serve (explaining notation rather than executing it) routed through the skill. Related to the *Skill Reliability and Adherence* section below, but with a sharper cause: this is not decay or inattention, it is a matching tool winning against a document.
- **A directory URI has no read path.** After the two file reads the host issued `resources/read` for `skill://dice-roller`, the parent directory, and got `-32602 Not a skill file`. The server advertises `directoryRead: true`; the host does not implement `resources/directory/read`, so a directory request has nowhere correct to go. Harmless here, but it is a wasted round trip and a user-visible error on an unluckier turn.
- **No read cache: the same supporting file was fetched six times.** `references/dice-notation.md` was read three times inside the turn that answered the question and three more in later turns. The spec pairs on-demand retrieval with a SHOULD to cache what is retrieved and revalidate against the entry digest. Implementing the first half without the second converts a prefetch problem into a refetch problem, and this ran against a free Hugging Face Space.
- **A listing was cached for 27 hours across four connections, with no staleness bound in sight.** Exactly one `skills/list` was issued, at 06:48 on 08-30. Discovery reported three skills again at 10:01 and 10:03 on 08-31, over four separate connections, without another wire call. The cause is a cache keyed on the server's *connection state string*: `Stopped → Running` reproduces the key the entry already had, so reconnecting never invalidates it. Two things make this more than a freshness bug. First, verification binds fetched bytes to digests from that cached manifest, so a server redeploying between sessions would have fresh content checked against a stale manifest — and at that point a legitimate update is indistinguishable from tampering. Second, the server sends SEP-2549 `ttlMs`/`cacheScope`, but scopes them to 2026-07-28+ connections; this host negotiates `2025-11-25` and so receives **no** caching guidance at all (confirmed: zero occurrences in the session log). The SEP places no upper bound on how long a host may retain a listing on an older revision, so a host that caches indefinitely is not violating anything. **Worth an explicit note in the SEP:** re-listing on reconnect is the behaviour a server would expect, and nothing currently asks for it.
- **`skills/get` remained unexercised.** It is reachable only for a skill absent from the listing, and this host has no path that produces such a URI — the server's `instructions` field points at `skill://secret-menu/SKILL.md`, but nothing mines instructions for skill URIs. A host that implements only `skills/list` never calls `skills/get`, and so never discovers that it works.

**Verification:** the demo's own SEP-2640 conformance suite (`smoke-http.ts`) passes against the live deployment, including digest- and size-verified reads. Independently confirmed on the wire: `resultType` is absent from every result — `skills/list`, `skills/get`, `tools/list`, `tools/call`, `resources/read` — while negotiating `2026-07-28`, where the base schema states servers "MUST include this field". Not skills-specific; it applies to every server on the v2 TypeScript SDK, and the same schema instructs clients to treat an absent value as `"complete"`, so nothing breaks today.

## McpGraph: Skills in MCP Server Repo

**Repo:** [TeamSparkAI/mcpGraph](https://github.com/TeamSparkAI/mcpGraph)
**Skill:** [mcpgraphtoolkit/SKILL.md](https://github.com/TeamSparkAI/mcpGraph/blob/main/skills/mcpgraphtoolkit/SKILL.md) (875+ lines)

Bob Dickinson built a standalone SKILL.md file that lives in the same repo as the MCP server, but they weren't formally connected. The skill instructs agents on building directed graphs of MCP nodes to orchestrate tool calls.

**Findings:**

- Claude ignored the SKILL.md initially, even when the skill and server had similar descriptions
- Claude would fail at using the server tools a couple times, then read the skill and succeed
- Expected Claude to start with the skill ("I know how to do X") before the server ("I do X"), but it didn't

**Resolution:** Added a server instruction telling the agent to read the SKILL.md before using the tool. That one change caused Claude to reliably read the skill first.

**Remaining concerns:**

- This workaround works for 1:1 skill-to-server case, but doesn't solve discovery — users installing from a registry don't know to also install the skill
- Distinguishes between "skill required to make the server work at all" vs. "skill that orchestrates tools you could use without it" — potentially different solutions needed

## Skilljack MCP

**Repo:** [olaservo/skilljack-mcp](https://github.com/olaservo/skilljack-mcp)

Loads skills into tool descriptions. Uses dynamic tool updates to keep the skills manifest current.

Example eval approach and observations here: https://github.com/olaservo/skilljack-mcp/blob/main/evals/README.md

## FastMCP 3.0 Skills Support

**URL:** [gofastmcp.com/servers/providers/skills](https://gofastmcp.com/servers/providers/skills)

FastMCP added skills support in version 3.0. Worth examining for alignment with other approaches.

**Update model comparison (Feb 26 office hours):**

- FastMCP supports more of a "pull" model for updating resources that have changed
- The skills-as-resources implementation in this repo ([PR #16](https://github.com/modelcontextprotocol/experimental-ext-skills/pull/16)) watches for changes and allows clients to subscribe to resources via `resources/subscribe` and `resources/updated` notifications — more of a "push" model
- Both models are worth evaluating; the right choice is likely use-case specific

**Related:** [jlowin/fastmcp#2694](https://github.com/jlowin/fastmcp/issues/2694)

## PydanticAI Skills Support

**PR:** [pydantic/pydantic-ai#3780](https://github.com/pydantic/pydantic-ai/pull/3780)

Introduces support for agent skills with a tools-based approach.

## NimbleBrain: skill:// Resource Consolidation

[Mat Goldsborough](https://github.com/mgoldsborough) (NimbleBrain) had previously maintained separate components for MCP server code, a skills monorepo, and registry metadata with `server.json`. After community discussion, he consolidated into single atomic repos per server with skills exposed as `skill://` resources directly on the server.

**Findings:**

- Collapsing three separate artifacts into one repo simplified build, versioning, and deployment — skills are colocated with the tools they describe and shipped atomically
- `skill://` resources enable ephemeral/installless availability: skill context is present while the server is installed and disappears when it disconnects, with no git cloning or file system access required on the client side
- Quick tests showed same or better results compared to the previous approach of injecting skills upstream before the LLM call
- Validates the skills-as-resources approach documented in [Approach 3](approaches.md#3-skills-as-tools-andor-resources)

**Reference implementations:** [mcp-ipinfo](https://github.com/NimbleBrainInc/mcp-ipinfo), [mcp-webfetch](https://github.com/NimbleBrainInc/mcp-webfetch), [mcp-pdfco](https://github.com/NimbleBrainInc/mcp-pdfco), [mcp-folk](https://github.com/NimbleBrainInc/mcp-folk), [mcp-brave-search](https://github.com/NimbleBrainInc/mcp-brave-search)

**Community input:**

> "Skills living as skill:// resources on the server itself was the natural endpoint of that consolidation. The skill context is colocated with the tools it describes, versioned together, shipped together." — [Mat Goldsborough](https://github.com/mgoldsborough) (NimbleBrain), via Discord

## Skill Reliability and Adherence

Multiple community members have independently reported that models do not reliably load or follow skill instructions, even when skills are preloaded in context. This is a cross-cutting behavioral problem, not specific to any single implementation approach.

**Findings:**

- Models appear to frequently ignore available skills, requiring hooks or repeated prompting to trigger skill loading
- Skill adherence appears to be "time-decaying" similar to other model instructions — models follow instructions initially but lose adherence as the context window grows and compaction occurs
- Behavior is model-specific: weaker models show lower success rates with lazy-loaded skills
- One effective workaround observed by Kryspin: wrapping skills in a subagent whose name or description mentions the skill topic
- Community desire for "skill autoloads" and "dynamic memory autoloads" as design patterns

**Community input:**

> "Even Opus 4.6 needs to be constantly bugged to load skills when they're preloaded in the context already. I actually have a hook that reminds it to load skills and it still just doesn't a lot of the time." — Luca (AWS), via Discord

> "I also have this problem with skills: they're useful… when used. Which isn't nearly often enough." — Jeremiah (FastMCP), via Discord

> "Skills are ephemeral and/or time decaying — it clicks once and then give it some time and they lose the plot." — Kryspin (qcompute), via Discord

> "I've seen lazy load skills with various degrees of success, actually looks like it might be model specific… [best pattern is] putting them in with a subagent that similarly named or mentions the topic in their description." — Kryspin (qcompute), via Discord

**See also:** [#37](https://github.com/modelcontextprotocol/experimental-ext-skills/issues/37) — Compare skill delivery mechanisms: file-based vs MCP-based
