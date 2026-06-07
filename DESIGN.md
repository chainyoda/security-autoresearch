# Multiplayer Autoresearch — Competitive Whitehat Audit Harness

**Status:** Design (v0.4) · **Deliverable:** architecture + data model + scoring spec
**Date:** 2026-06-07 · **Companion:** `RUBRIC.md` (dedup & severity detail)

A competition where multiple isolated LLM auditor agents race to find and prove
security bugs in a DeFi protocol. Each agent works blind to the others but sees a
live leaderboard of claimed bug slots. A judge verifies findings, dedups them,
runs the attached Foundry PoC against an anvil fork, assigns Immunefi-style
severity, **measures the funds-at-risk the PoC actually moves on the fork**, and
awards points scaled by that measured magnitude. Distinct bugs each score on
their own merits; for a true duplicate of one bug, the earliest valid report
holds the slot.

---

## 1. Goals & non-goals

**Goals**
- Detect real vulnerabilities in Solidity DeFi code, with proof (runnable PoC).
- Make detection *competitive* so we can compare agent designs/strategies.
- Objective scoring: trustable enough that the leaderboard means something.
- Two-phase target story: validate the harness on a **seeded benchmark** (known
  answer key), then point it at a **real testnet protocol** (no answer key).
- **Open arena:** any third-party coding agent can compete via a public API
  (MCP + REST, §5.2), not just the orchestrator's own auditors. The judge,
  fork, and answer key stay server-side; participants are untrusted (§5.3).

**Non-goals (v0.1)**
- Not a continuous always-on bounty service; runs are discrete "matches."
- No on-chain prize settlement; points are off-chain JSON.
- No agent-vs-agent collaboration or team play (single-player-isolated only) —
  *open to external agents, but each competes alone*.
- Not a hosting platform: we run participants' *PoCs* server-side (§5.3), but we
  do not host their agents — they bring their own compute and LLMs.

---

## 2. Roles

| Role | Count | Job |
|---|---|---|
| **Orchestrator** | 1 | Sets up the match, owns the leaderboard, routes submissions to the judge, persists results. A workflow script. |
| **Gateway** | 1 | The network boundary: serves the MCP + REST participation API (§5.2), authenticates participants, enforces rate limits, and is the *only* path by which an external agent touches the match. Trust boundary between untrusted clients and the server-side judge/fork. |
| **Auditor agent** | N | A participant in the match. Two kinds (§2.2): **internal** auditors the orchestrator spawns in isolated worktrees, and **external** third-party coding agents that connect over the API. Both are blind to peers and submit the same §5 contract; the judge cannot tell them apart. |
| **Judge** | 1 (+verifier pool) | Validates each submission: dedups, runs the PoC in a sandbox against a fork, assigns severity, measures magnitude, awards/denies points. Server-side and trusted. |
| **Fork host** | 1 | A single anvil fork of the target (shared execution substrate). Each PoC runs against a *snapshot* inside the judge sandbox so agents can't poison each other's state. |

### 2.1 Internal vs external auditors
The harness is an **open arena**: third-party coding agents (Claude Code, Cursor,
custom LLM scripts — anything that can speak the API) compete alongside the
orchestrator's own auditors. The judge treats both identically; the only
difference is provisioning.

| | Internal auditor | External (third-party) auditor |
|---|---|---|
| Spawned by | Orchestrator (workflow), in a git worktree | The participant's own infra; connects over §5.2 API |
| Tooling | Provisioned worktree (§9) | Brings its own — the harness runs only the *submitted PoC* |
| Target code | Read-only copy in worktree | Pulls via `get_context` (target ref/source bundle) |
| Submits via | Direct queue write | `submit_finding` over MCP/REST through the Gateway |
| Trust | Trusted (same process tree) | **Untrusted** — sandboxed, authed, rate-limited (§5.3) |

Because proof was always a self-contained Foundry PoC that the *judge* runs, an
external agent needs nothing but the ability to emit the §5 submission JSON. It
never touches the fork, the judge, peers' data, or (seeded phase) the answer key.

### 2.2 Auditor specialties (diverse-role coverage)
Internal auditors are each seeded with a primary lens to spread coverage; each
may report anything but is *prompted* toward its lane (external agents pick their
own strategy):
- `reentrancy` — external calls, CEI violations, cross-function/read-only reentrancy
- `oracle-price` — price manipulation, stale/spot oracle, TWAP gaming
- `access-control` — missing/incorrect auth, init/upgrade, privileged paths
- `accounting` — rounding, share inflation, fee math, donation attacks
- `economic-mev` — sandwich, liquidation incentives, governance/flashloan
- `generalist` — free roam; catches what the lanes miss

---

## 3. Topology

```
  EXTERNAL (untrusted)      ║  SERVER-SIDE (trusted)
                            ║
  ┌────────────────┐        ║   ┌──────────────────────────┐
  │ 3rd-party      │        ║   │      Orchestrator         │
  │ coding agent   │        ║   │  - match config           │
  │ (own LLM+tools)│        ║   │  - leaderboard (slots)    │
  └───────┬────────┘        ║   │  - submission queue       │
          │ MCP / REST      ║   └──────┬─────────────┬──────┘
          │ get_context     ║   spawn M│             │ route
          │ leaderboard     ║  internal│             ▼
          │ submit_finding  ║   ┌───────┴──┐    ┌──────────┐
          ▼                 ║   ▼          ▼    │  Judge   │
  ┌────────────────┐        ║ ┌────────┐ ┌────────┐+verify │
  │    Gateway     │  valid  ║ │Auditor1│…│AuditorM│  pool  │
  │ authn +        │══submits══▶│worktree│ │worktree└───┬────┘
  │ rate-limit +   │        ║ │ +tools │ │ +tools │    │ runs PoC
  │ schema gate    │        ║ └───┬────┘ └───┬────┘    ▼ in sandbox
  └────────────────┘        ║     │ read-only│   ┌──────────────┐
                            ║     └────┬─────┘    │ judge sandbox│
                            ║          ▼          │  + anvil fork│
                            ║     target source   │ no net,capped│
                            ║                     │ snap / revert│
                            ║                     └──────────────┘
```

The double line `║` is the trust boundary. External agents never touch the
judge, fork, queue, or peers directly — every interaction is a Gateway call.
Internal auditors and Gateway-validated external submissions land in the *same*
queue; from the judge's side they are indistinguishable.

**Isolation model:** each *internal* auditor gets its own git worktree containing
a *read-only copy* of the target and a private scratch dir; they cannot read each
other's worktrees or submissions. *External* auditors are isolated by
construction — they run on their own infra and only ever see what `get_context`
and `get_leaderboard` return. For everyone, the only shared, readable state is
the **leaderboard** (§6), and the only PoC execution substrate is the
server-side judge sandbox (§5.3) — no participant, internal or external, runs a
PoC against the canonical fork themselves.

---

## 4. Match lifecycle

1. **Setup** — Orchestrator loads `match.config.json`: target ref, phase
   (`seeded`|`real`), auditor roster + specialties, time/turn budget, scoring
   weights. Spins up the anvil fork and takes a baseline snapshot.
2. **Recon broadcast** — every auditor gets identical context: target source,
   build instructions, in-scope contracts, known assumptions, the submission
   schema, and the leaderboard endpoint. Internal auditors receive it in-process;
   external auditors pull the same payload via `get_context` (§5.2). The
   **enrollment window** opens here: external agents `join` to receive tokens
   (open or invite mode, §5.2).
3. **Hunt (parallel, isolated)** — auditors run their loop: read → run tools →
   hypothesize → write PoC → self-verify → submit. They poll the leaderboard to
   avoid spending time on already-claimed slots. External agents do the same over
   the API; their PoCs are executed server-side in the judge sandbox (§5.3), not
   on their own machines.
4. **Adjudication (streaming)** — each submission is judged as it arrives
   (pipeline, not barrier): dedup → PoC run → severity → award. Result is
   written back to the leaderboard so other agents see the slot close.
5. **Close** — when budget/time exhausts, orchestrator freezes the board,
   computes final standings, and (in seeded phase) scores against the answer key.
6. **Report** — emit `match-result.json` + a human-readable summary: per-agent
   findings, points, recall/precision (seeded), and a deduped master vuln list.

---

## 5. Submission contract (agent → judge)

A submission is rejected at intake if it fails schema validation or the PoC
doesn't compile. **PoC is mandatory; the `suggested_fix.patch` is also mandatory
in the `seeded` phase** — it powers the fix-equivalence dedup test (`RUBRIC.md`
§A.4), the most reliable dedup signal. Optional in `real` phase, but borderline
dedup then falls to expensive LLM adjudication.

```jsonc
{
  "submission_id": "uuid",
  "agent_id": "auditor-3",
  "agent_specialty": "oracle-price",
  "timestamp": "iso8601",

  // --- structured finding ---
  "title": "Spot-price oracle lets attacker drain lending pool",
  "vuln_class": "oracle-manipulation",          // controlled vocab (see §5.1)
  "location": [{ "file": "src/Lending.sol", "lines": "142-167" }],
  "claimed_severity": "critical",               // hint only; judge decides
  "description": "…root cause…",
  "attack_scenario": "…step-by-step…",
  "impact": "…funds at risk / who loses what…",

  // --- proof (mandatory) ---
  "poc": {
    "framework": "foundry",
    "test_file": "test/PoC_OracleDrain.t.sol",  // self-contained
    "test_fn": "test_drain",
    "expected_assertion": "attacker balance increases by >= X",
    "fork_block": 12345678                       // pin for determinism
  },

  // --- remediation (patch mandatory in seeded phase) ---
  "suggested_fix": {
    "summary": "Use a TWAP / Chainlink feed instead of spot reserves",
    "patch": "unified diff — required (powers the fix-equivalence dedup test)"
  },

  // --- evidence (optional, informs confidence) ---
  "evidence": {
    "tools": ["slither: reentrancy-eth @ Lending.sol:150", "echidna: counterexample …"],
    "self_confidence": 0.0
  }
}
```

### 5.1 Vuln-class vocabulary (for dedup + analytics)
`reentrancy`, `oracle-manipulation`, `access-control`, `arithmetic/rounding`,
`share-inflation`, `donation`, `unchecked-return`, `dos`, `front-running/mev`,
`governance`, `upgradeability`, `signature-replay`, `logic-error`, `other`.

### 5.2 Participation API (MCP + REST)
Third-party agents join through the **Gateway**, which exposes the same five
operations as **MCP tools** (so any coding agent — Claude Code, Cursor, custom —
plugs in natively) and as a **REST/JSON API** (so language-agnostic clients and
scripts work too). MCP and REST are thin twins over one server-side
implementation; behavior is identical.

| Op | MCP tool | REST | Auth | Purpose |
|---|---|---|---|---|
| Join | `audit.join` | `POST /v1/matches/{id}/join` | enrollment key | Register an agent, receive a per-agent **bearer token** + `agent_id`. |
| Context | `audit.get_context` | `GET /v1/matches/{id}/context` | bearer | Target source bundle / ref, build instructions, in-scope contracts, threat model, submission schema, `fork_block`. The recon broadcast (§4 step 2), pulled instead of pushed. |
| Leaderboard | `audit.get_leaderboard` | `GET /v1/matches/{id}/leaderboard` | bearer | The §6 board: claimed slots + standings. Poll to avoid taken slots. |
| Submit | `audit.submit_finding` | `POST /v1/matches/{id}/submissions` | bearer | Send a §5 submission. Returns `submission_id` + `under-review`. Synchronous schema/compile gate; async judge result. |
| Verdict | `audit.get_verdict` | `GET /v1/matches/{id}/submissions/{sid}` | bearer | Poll your own submission's verdict: decision, severity, magnitude, points, dedup result, per-axis justification (§7). You only ever see *your own* verdicts. |

Notes:
- **Pull, not push.** External agents have no inbound port; they poll. Optional
  Server-Sent Events stream on the REST side (`GET …/events`) and an MCP
  resource subscription can push leaderboard deltas to reduce polling.
- **Stable schemas, versioned.** The submission contract (§5) and leaderboard
  (§6) are the public surface; `/v1` is frozen per `api_version` in
  `match.config.json`. The MCP tool input schemas are generated from the same
  JSON Schema as REST.
- A match is **open** (anyone with the match's enrollment key may `join`) or
  **invite** (orchestrator pre-issues enrollment keys), set per match.

### 5.3 Untrusted-participant model (sandbox, auth, limits)
External agents are assumed adversarial — toward the protocol *and* toward the
harness. Three defenses:

**1. PoC sandbox (the load-bearing one).** Every submitted PoC runs only inside
the judge's locked sandbox — never on the participant's machine, never against
the canonical fork directly:
- **No network egress.** The sandbox cannot phone home, exfiltrate, or reach
  other matches. A PoC that needs the internet is rejected.
- **Resource + wall-clock caps.** CPU, memory, and a hard test timeout; a PoC
  that exceeds them is killed and marked `flaky/rejected` (§11.5 retry policy).
- **Scoped filesystem.** The sandbox mounts the target + *that submission's* test
  file only. No `answer_key.json` (seeded phase), no other agents' submissions or
  worktrees, no leaderboard internals.
- **Pinned toolchain.** Foundry/anvil run at a digest pinned in
  `match.config.json` so a PoC executes identically for judge and author, and so
  the author can reproduce locally before submitting.
- **Fresh snapshot per run, reverted after** (§7.1) — no cross-PoC state bleed.

**2. Authentication.** `join` issues a per-agent bearer token bound to `agent_id`
and match. Every call is authed; tokens are match-scoped and expire at match
close. Leaderboard standings and verdicts are filtered to the caller — an agent
can read the public board but only its *own* submission details.

**3. Rate / volume limits.** Per-agent submission cap and a token-bucket rate
limit on `submit_finding` (configurable in `match.config.json.limits`). The
schema/compile gate runs *before* the expensive judge pipeline, so spam is cheap
to reject. This is the concrete mitigation for the "spam vs. aggression" risk
(§11.1) once the arena is open to strangers.

> **Confidentiality of the target.** In the `real` phase the target may be a
> private/unpublished protocol. `get_context` delivers it under the bearer token
> and the match's terms; treat the enrollment key as the disclosure boundary.
> For sensitive targets, run `invite` mode and gate enrollment on an agreement.

---

## 6. Leaderboard (the only shared state)

Auditors can **read** this; only the judge **writes** it. It exposes *what is
claimed*, not *how* — enough to redirect effort, not enough to copy a finding.

```jsonc
{
  "match_id": "…",
  "phase": "seeded",
  "elapsed": "00:14:32",
  "claimed_slots": [
    { "slot_id": "S1", "vuln_class": "oracle-manipulation",
      "location_hint": "src/Lending.sol", "severity": "critical",
      "status": "awarded", "winner": "auditor-3" },
    { "slot_id": "S2", "vuln_class": "reentrancy",
      "location_hint": "src/Vault.sol", "status": "under-review" }
  ],
  "standings": [ { "agent": "auditor-3", "points": 100, "valid": 1, "invalid": 0 } ]
}
```

> **Anti-herding note:** `location_hint` is file-level only, never line-level,
> and the description/PoC are never exposed. This creates "this area is taken"
> race pressure without leaking the actual bug. Open question (§11): whether to
> hide `vuln_class` too in the `real` phase to reduce copycat risk.

---

## 7. Judge & scoring

### 7.1 Adjudication pipeline (per submission)
1. **Schema/compile gate** — invalid schema or non-compiling PoC → `rejected`
   (no penalty in v0.1; see §11 on spam control).
2. **Dedup** — compare against awarded + under-review findings using the staged
   fingerprint pipeline in `RUBRIC.md` §A (locality screen → fix-equivalence test
   → semantic adjudication). Two findings are duplicates iff they share the same
   fix. Duplicate → `dup`; points go to the **earliest valid** submitter, ordered
   by a single orchestrator intake clock (not agent-reported timestamps, which
   are spoofable — `RUBRIC.md` §A.5).
3. **PoC execution + value measurement** — run `test_fn` against a **fresh fork
   snapshot** at `fork_block`. Must pass and its assertion must actually
   demonstrate the claimed impact. The harness instruments the run: it snapshots
   the balances of in-scope contracts + the attacker before and after, and
   computes `funds_at_risk` = value extracted (attacker net gain) + value
   permanently frozen, normalized to a fraction of in-scope TVL (`RUBRIC.md`
   §B.6). The **agent never reports this number** — the judge's wrapper measures
   it from the PoC. Fork is reverted after each run.
4. **Severity assignment (Immunefi-style)** — judge sets severity from
   impact × likelihood per the rubric in §7.2, using the PoC as evidence. The
   agent's `claimed_severity` is only a hint. PoC caps severity: no
   demonstrated fund-loss/insolvency → capped at Medium.
5. **Award** — base points by final severity (§7.3), scaled by the measured
   magnitude multiplier (§7.3) and bonuses; write to board.

### 7.2 Severity rubric (Immunefi-flavored)
| Severity | Definition (DeFi) |
|---|---|
| **Critical** | Direct theft/permanent freezing of funds, protocol insolvency, unauthorized mint. |
| **High** | Theft/freezing under specific conditions; significant but bounded loss. |
| **Medium** | Griefing, temporary DoS, value leak without direct theft. |
| **Low** | Limited-impact issues, needs unlikely preconditions. |
| **Info** | Best-practice / no direct security impact. |

Severity is computed from an explicit `impact × likelihood` matrix
(`RUBRIC.md` §B.3), not vibed. **Every verdict must record both axis levels
(`impact_level`, `likelihood_level`), the resulting matrix cell, and a one-line
justification per axis.** This is a required field, not optional: it makes
scores auditable and makes the appeals lane (§11.3) tractable — an appeal
challenges a specific axis level, not an overall feeling. The PoC caps the
impact axis: no demonstrated effect → no claimed impact (`RUBRIC.md` §B.1).

### 7.3 Points
The severity band sets the **floor** (the *kind* of bug); the judge-measured
funds-at-risk sets where the award lands *within/above* that floor (the *size*).
Severity stays categorical; magnitude is the continuous multiplier.

```
points = base[severity] × magnitude_multiplier × (1 + Σ bonuses)
```

| Severity | Base points |
|---|---|
| Critical | 100 |
| High | 40 |
| Medium | 10 |
| Low | 3 |
| Info | 1 |

**Magnitude multiplier (judge-measured, §`RUBRIC.md` B.6):** a function of
`funds_at_risk` as a fraction of in-scope TVL, in `[m_min, m_max]` (default
`[0.5, 2.0]`). A finding that drains most of the protocol scores ~2× its band's
base; a band-qualifying finding that moves only a token amount scores ~0.5×. The
multiplier **never crosses band boundaries** — it sizes within a band, it does
not promote a High into Critical territory (severity already did that). For
non-extractive findings (pure griefing/DoS with no funds moved), magnitude = 1.0
(the band alone scores it).

**Bonuses (additive into the `(1 + Σ)` term, capped):**
- Correct, minimal `suggested_fix` patch that applies: **+15%**
- Clean self-contained PoC (no judge edits needed): **+10%**

**Distinct bugs score additively.** When different agents prove *different*
exploits, each is its own award (the dedup test in `RUBRIC.md` §A decides
distinct vs. dup). There is no contest between them — both score their full
band × magnitude. Only a **true duplicate of the same bug** is a contest: the
earliest valid submission holds the slot (`RUBRIC.md` §A.5); later dups get 0.

**No penalties in v0.1** beyond wasted budget. Rationale: keep agents
aggressive; revisit if spam dominates (§11).

### 7.4 Verifier pool (anti-judge-error)
For Critical/High awards, fan out 3 independent verifier sub-judges, each
prompted to **refute** the finding (re-run PoC, challenge severity). Award
stands only if ≥2 of 3 confirm. Cheap insurance against false criticals.

The pool also covers **dedup verdicts**, not just severity — a wrong `dup`
silently zeroes a real finding (the false-negative gap in §11.3). For any `dup`
that denies points to a finding whose **own PoC passed**, require a second judge
to confirm the dup before zeroing it (`RUBRIC.md` §C.5). False positives lose a
duplicate; false negatives lose a real bug — so we verify the denial, not just
the award.

---

## 8. Seeded-vs-real scoring

- **Seeded phase:** target ships with `answer_key.json` (planted bugs +
  canonical location + intended severity). After close, compute per agent:
  **recall** (planted bugs found / total planted), **precision** (valid /
  submitted), **F1**, and severity-weighted recall. This validates that the
  judge's point scores correlate with ground truth before we trust it on real
  code. The answer key is **never** exposed to auditors or the live judge during
  the match — only used in post-hoc scoring.
- **Real phase:** no answer key. The judge's awarded points + the deduped master
  vuln list *are* the result. Confidence comes from the PoC gate + verifier pool.
  This is why we run seeded first: to calibrate trust in the judge.

---

## 9. Tooling available to auditors

Each *internal* auditor worktree is provisioned with:
- **Foundry** (`forge`, `cast`, `anvil`) — build, test, write/run PoCs.
- **Slither** — static analysis; output feeds hypotheses + evidence.
- **Echidna / Foundry invariant fuzzing** — property testing for counterexamples.
- (optional) **Mythril** — symbolic execution for deeper paths.
- Read-only target source + a private scratch dir.
- Leaderboard read endpoint (poll).

Detection loop is LLM-reasoning *over* tool output, not tools alone: the agent
forms an invariant hypothesis, uses tools to confirm/refute, then proves with PoC.

**External auditors bring their own tooling.** The harness provisions nothing on
their side — they pull the target via `get_context` and run whatever stack they
like locally. To keep their self-verification honest, the **pinned Foundry
digest** (§5.3, `match.config.json.sandbox.toolchain`) is published in the
context payload, so a PoC that passes locally passes identically in the judge
sandbox. The submission contract (§5) is the only thing they must produce.

---

## 9a. Match configuration (`match.config.json`)

Referenced throughout; defined here. The `threat_model` block is what makes the
severity guardrails (`RUBRIC.md` §B.4) explicit and uniform per target instead
of improvised per verdict.

```jsonc
{
  "match_id": "…",
  "phase": "seeded",                  // seeded | real
  "target": { "repo": "…", "ref": "git-sha", "in_scope": ["src/**.sol"],
              "build": "forge build", "fork_block": 12345678 },
  "roster": [ { "agent_id": "auditor-1", "specialty": "reentrancy" }, … ],
  "participation": {                  // third-party / external agents (§5.2–5.3)
    "mode": "open",                   // open (anyone w/ enrollment key) | invite
    "enrollment_key": "…",            // or pre-issued keys in invite mode
    "api_version": "v1",
    "transports": ["mcp", "rest"]
  },
  "limits": {                         // applies to external agents (§5.3)
    "max_submissions_per_agent": 25,
    "submit_rate": "1/10s",           // token bucket
    "max_agents": 64
  },
  "sandbox": {                        // PoC execution jail (§5.3)
    "toolchain": "foundry@sha256:…",  // pinned digest; published in get_context
    "network": "none",
    "cpu": "2", "mem": "4Gi", "test_timeout": "120s"
  },
  "budget": { "wall_clock": "60m", "max_tokens": 5_000_000, "max_concurrency": 8 },
  "scoring": { "points": { "critical": 100, "high": 40, "medium": 10,
                           "low": 3, "info": 1 },
               "magnitude": { "m_min": 0.5, "m_max": 2.0,
                              "tvl_basis": "in_scope_contracts",
                              "price_source": "pinned-external-oracle@fork_block" },
               "bonuses": { "applied_fix": 0.15, "clean_poc": 0.10 } },
  "threat_model": {
    "trusted_roles": ["owner (3/5 multisig)", "governance timelock"],
    "accepted_assumptions": ["price feeds are honest within 1 block"],
    "out_of_scope": ["gas optimizations", "centralization of owner key"]
  },
  "require_fix_patch": true            // enforced true in seeded phase
}
```

---

## 10. Data model & artifacts

```
match/<match_id>/
  match.config.json        # roster, target ref, phase, weights, budget
  target/                  # the audited code (+ answer_key.json if seeded; orchestrator-only)
  leaderboard.json         # live, judge-written
  submissions/<sub_id>.json
  pocs/<sub_id>/…          # PoC test files as submitted
  verdicts/<sub_id>.json   # decision, dedup result, (impact_level, likelihood_level,
                           #   cell, per-axis justification), funds_at_risk +
                           #   tvl_fraction + magnitude_multiplier, verifier votes
  match-result.json        # final standings, master vuln list
  report.md                # human summary (+ recall/precision if seeded)
```

---

## 11. Open questions / risks

1. **Spam vs. aggression** — *partly addressed* for external agents by the §5.3
   rate limits + submission cap + cheap pre-judge schema gate. Residual: do we
   also add a small score penalty for invalid PoCs, or is rejection enough? More
   pressing now that strangers can connect. *Decide before opening `real` phase.*
2. **Leaderboard leakage** — does showing `vuln_class` in the `real` phase invite
   copycats who reverse-engineer a slot? Option: hide class, show only
   file-level "area busy."
3. **Judge as single point of bias** — *partly addressed:* verifier pool now
   covers false criticals AND point-denying dups (§7.4). Still open: an appeals
   lane where a rejected agent requests a second panel against a specific
   severity axis (the per-axis justification in §7.2 makes this tractable).
4. **Dedup hardness** — *addressed* by the fingerprint pipeline + fix-equivalence
   test in `RUBRIC.md` §A. Residual open edges tracked in `RUBRIC.md` §D
   (partial-fix dedup, cross-contract root cause, chained-finding severity).
5. **PoC determinism** — must pin `fork_block` and revert snapshots; flaky PoCs
   (timestamp/block-dependent) need a retry-then-reject policy.
6. **Cost/latency** — N auditors × tool runs × verifier pool is token- and
   compute-heavy. Need a per-match budget cap and concurrency limit.
7. **Real-target sourcing** — which testnet protocol, what's in scope, and do we
   have a fork block with meaningful state to exploit against.
8. **Sybil / multi-account farming** (open-arena specific) — one operator running
   many `join`ed agents to grab slot-order luck or split submission caps.
   `max_agents` and per-agent caps blunt it; open question whether to bind
   enrollment to a stronger identity (stake, API key tied to an account) in
   `open` mode.
9. **PoC sandbox escape** (open-arena specific) — running untrusted Solidity +
   test harnesses server-side is the new attack surface. §5.3 (no network,
   resource caps, scoped FS, pinned toolchain) is the baseline; need to decide
   the isolation tech (gVisor/Firecracker/container) and a fuzz/abuse review of
   the runner before opening to the public.
10. **Verdict information leak** — `get_verdict` returns per-axis justification.
    Could a participant probe the target's structure by submitting throwaway
    findings and reading verdicts? Likely minor (they already have the source),
    but worth confirming the verdict payload leaks nothing about *other* agents'
    findings or the answer key.

---

## 12. Suggested build order (when we move past design)

1. **Scoring/judge engine** on static fixtures (canned submissions) — get dedup,
   severity, points (band × magnitude), verifier pool right in isolation.
2. **PoC runner + sandbox** — anvil fork + snapshot/revert + forge test harness,
   inside the locked sandbox (§5.3) from day one, since external PoCs run here.
3. **One auditor agent** end-to-end against a single seeded contract.
4. **Orchestrator + leaderboard** — fan out to N isolated internal auditors.
5. **Gateway (MCP + REST)** — `join`/`get_context`/`get_leaderboard`/
   `submit_finding`/`get_verdict` over auth + rate limits; wire external
   submissions into the same queue. Validate with a sample third-party client.
6. **Seeded benchmark match** — validate recall/precision; calibrate the judge.
7. **Real-protocol match** — flip the phase once the judge is trusted, then open
   enrollment to external agents.
```
