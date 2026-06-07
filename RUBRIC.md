# Dedup & Severity Rubric

**Companion to** `DESIGN.md` §7 (Judge & scoring) and §11.4 (dedup hardness).
**Status:** Design (v0.1) · **Date:** 2026-06-07

This is the part of the harness most likely to be *wrong* — both judgments are
semantic, not mechanical. The goal here is to push as much of each call as
possible onto deterministic checks, and to make the residual LLM judgment
narrow, structured, and auditable (every verdict carries its reasoning).

---

## Part A — Dedup

### A.0 Why this is hard
"Same bug" is a property of the **root cause**, not of the report. Two agents
can describe the same vulnerability with different titles, different vuln_class
labels, different line ranges, and PoCs that take different attack paths. The
inverse is also true: two findings in the *same function* can be genuinely
distinct bugs. Neither location nor class nor PoC alone is sufficient.

### A.1 The unit of identity: a *root-cause fingerprint*
We dedup on a **root cause**, defined as a tuple:

```
fingerprint = (
  faulty_code_site,     # the line(s) that must change to fix it
  defect_mechanism,     # WHY it's wrong (missing check, wrong order, bad math…)
  exploit_precondition  # what the attacker needs to trigger it
)
```

Two findings are duplicates iff they share the **same fix**. This is the
operational test, and it cuts through the noise: *if applying agent A's
`suggested_fix` would also neutralize agent B's PoC, they are the same bug.*
That test is mechanically checkable (see A.4) and is the tiebreaker whenever the
semantic call is ambiguous.

### A.2 Staged dedup pipeline (cheap → expensive)
Run in order; stop at the first stage that returns a confident DUP or DISTINCT.

| Stage | Check | Resolves |
|---|---|---|
| **0. Exact** | Same `vuln_class` + identical location lines + same PoC assertion target | Obvious dups (rare) |
| **1. Locality screen** | Do location ranges overlap *or* touch the same function/modifier? If no overlap AND different files AND different class → **DISTINCT**, stop. | Cheaply separates unrelated findings |
| **2. Fix-equivalence** | Apply candidate's fix to a fork; re-run the *already-awarded* finding's PoC. Still exploitable? → not a dup of that one. Neutralized? → strong DUP signal. | The decisive mechanical test (A.4) |
| **3. Semantic adjudication** | LLM judge compares the two on the fingerprint tuple, using a fixed rubric (A.3). Only reached when stages 0–2 are inconclusive. | The residual hard cases |

Most pairs resolve at stage 1 (clearly unrelated) or stage 2 (fix neutralizes →
dup). Stage 3 is the expensive minority and the one we audit hardest.

### A.3 Semantic adjudication rubric (stage 3)
The judge answers four yes/no questions. **DUP requires Q1 AND Q2 to both be
YES.** Q3/Q4 break ties when Q1/Q2 disagree.

1. **Same fix?** Would a single, minimal code change resolve both? *(decisive —
   this is A.1)*
2. **Same mechanism?** Same defect class at the same code site (not just same
   vuln_class label — same actual flaw)?
3. **Same precondition?** Does triggering both require the same attacker setup /
   market state / privileged role?
4. **Subset relationship?** Is one finding a strictly more general statement of
   the other (e.g. "all functions miss `nonReentrant`" vs "withdraw() misses
   `nonReentrant`")?

Decision table:

| Q1 same-fix | Q2 same-mech | Verdict |
|---|---|---|
| YES | YES | **DUP** |
| YES | NO | DUP (fix dominates — same line changes, judge notes the mechanism discrepancy) |
| NO | YES | DISTINCT (same flaw class, but they need different fixes → two bugs) |
| NO | NO | **DISTINCT** |

For **subset (Q4 = YES)**: the *broader* correct finding wins the slot if its
PoC actually demonstrates the general case; otherwise the specific one wins and
the broad one is recorded as "claimed but unproven scope."

### A.4 The fix-equivalence test (stage 2, mechanical)
This is the single most reliable signal and worth implementing carefully.

```
given:  awarded finding F (with PoC_F), candidate C (with suggested_fix patch_C)
1. fork at F.fork_block, snapshot
2. apply patch_C to the target
3. recompile; if patch_C fails to apply or compile → inconclusive, fall to stage 3
4. run PoC_F against the patched fork
5. PoC_F now FAILS (exploit blocked)  → patch_C fixes F too → DUP (C and F share a fix)
   PoC_F still PASSES (still exploitable) → patch_C does NOT fix F → DISTINCT
6. revert snapshot
```

Caveat: requires the candidate to ship an applicable `patch` (not just a prose
fix summary). If absent, stage 2 is skipped and we rely on stage 3. This is a
concrete argument for **making the fix patch mandatory, not bonus** — see the
recommendation in §C.

### A.5 First-valid-wins & ordering
- Award goes to the **earliest valid submission** by `timestamp` among a dup
  cluster. "Valid" = passed PoC gate. A non-compiling earlier submission does
  **not** hold the slot.
- Race window: a finding is `under-review` from intake until verdict. If B
  submits a dup while A is still `under-review`, B is queued behind A; if A
  resolves valid, B is marked `dup`; if A fails its PoC, B is re-evaluated as
  the new earliest. (Avoids a fast-but-broken submission blocking a real one.)
- Ordering uses a **single orchestrator clock** stamped at intake, not
  agent-reported timestamps (which are spoofable). Agents can't game ordering.

### A.6 Worked dedup examples

**Ex.1 — DUP (different class labels, same bug).**
A: `vuln_class: reentrancy`, "withdraw() allows reentrancy, drain vault."
B: `vuln_class: logic-error`, "withdraw() updates balance after transfer."
→ Stage 1: same function. Stage 2: B's fix (move balance update before transfer
= CEI) neutralizes A's reentrancy PoC. → **DUP.** Earliest wins. Class labels
disagreeing is irrelevant.

**Ex.2 — DISTINCT (same function, two bugs).**
A: "withdraw() reentrancy" → fix: add `nonReentrant`.
B: "withdraw() rounding lets you withdraw 1 wei more per call" → fix: change
rounding direction.
→ Stage 2: A's fix doesn't stop B's PoC and vice-versa. Q1 NO. → **DISTINCT.**
Two separate awards even though both live in `withdraw()`.

**Ex.3 — Subset.**
A: "all 4 external functions lack `nonReentrant`" (PoC hits `withdraw` only).
B: "withdraw() lacks `nonReentrant`" (PoC hits `withdraw`).
→ Q4 YES, A is broader but PoC only proves `withdraw`. Per A.3 subset rule:
B's slot for the proven `withdraw` case; A recorded as "broader scope, 3/4
unproven." If A had PoCs for all four, A wins and B is a dup.

**Ex.4 — DISTINCT despite same root file & same class.**
Both `oracle-manipulation` in `Lending.sol`. A manipulates the spot reserves of
pool X; B manipulates a *different* oracle feed for asset Y. Locations overlap
(both read from `getPrice()`), but fixes differ (A: TWAP on pool X; B: validate
feed Y staleness). Stage 2: neither fix blocks the other's PoC. → **DISTINCT.**

---

## Part B — Severity

### B.0 Principle
Severity must be **computed and justified**, never vibed. The judge fills a
fixed `impact × likelihood` matrix, the PoC caps the impact axis (you can't
claim fund-loss you didn't demonstrate), and the result maps deterministically
to a severity band. The agent's `claimed_severity` is a hint only.

### B.1 Impact axis (set by what the PoC demonstrates)
| Level | DeFi meaning | PoC must show |
|---|---|---|
| **I5 Catastrophic** | Theft/permanent freeze of *all* or unbounded funds; protocol insolvency; unauthorized infinite mint | Attacker net-positive draining > protocol-significant amount, or supply inflation, on the fork |
| **I4 Severe** | Theft/freeze of a *bounded but significant* portion; conditional on realistic state | Demonstrated loss under stated, reachable preconditions |
| **I3 Moderate** | Value leak / griefing / temp DoS without direct theft | State shows leak or liveness break, no attacker profit needed |
| **I2 Minor** | Limited, recoverable, or requires unlikely preconditions | Effect shown but small/recoverable |
| **I1 Negligible** | No security-relevant on-chain effect | Best-practice / informational |

> **PoC cap rule:** impact axis cannot exceed what the PoC's assertions actually
> prove. A prose claim of "could drain everything" with a PoC that only moves 1
> wei is capped at I2. This is the main anti-inflation lever.

### B.2 Likelihood axis (set by preconditions)
| Level | Meaning |
|---|---|
| **L5 Trivial** | Any unprivileged actor, any time, no special state. Single tx. |
| **L4 Easy** | Unprivileged but needs cheap setup (flashloan, a few txs, common market state). |
| **L3 Conditional** | Needs a plausible but specific state (certain pool ratio, pending governance). |
| **L2 Hard** | Needs privileged role *misuse*, rare state, or large capital with real risk. |
| **L1 Theoretical** | Needs trusted-party action, unrealistic state, or assumptions outside threat model. |

### B.3 The matrix → severity band
```
            L5 Triv  L4 Easy  L3 Cond  L2 Hard  L1 Theo
I5 Catastr   CRIT     CRIT     CRIT     HIGH     MED
I4 Severe    CRIT     HIGH     HIGH     MED      LOW
I3 Moderate  HIGH     MED      MED      LOW      LOW
I2 Minor     MED      LOW      LOW      LOW      INFO
I1 Neglig    INFO     INFO     INFO     INFO     INFO
```
Severity is the cell. The verdict records `(impact_level, likelihood_level,
cell, one-line justification for each axis)`. This is what makes scores
auditable and appealable.

### B.4 Threat-model guardrails (prevents bogus criticals)
A finding is **downgraded to Info / rejected** if its exploit relies on:
- A trusted/privileged actor acting maliciously *when that role is explicitly
  trusted in the protocol's documented threat model* (e.g. "owner can rug" when
  owner is a trusted multisig by design). Record as `out-of-threat-model`.
- Centralization risks already disclosed as accepted assumptions.
- Conditions the `match.config.json` scope marks out of bounds.

These guardrails live in `match.config.json.threat_model` so they're explicit
per target and the judge applies them uniformly.

### B.5 Worked severity examples

**Ex.1 — CRIT.** Spot-oracle manipulation drains the lending pool. PoC: flashloan
→ skew reserves → borrow against inflated collateral → repay flashloan,
attacker net +$X of pool funds, single tx, no privilege. Impact I5 (theft of
unbounded funds, shown). Likelihood L4 (flashloan, cheap). Matrix I5×L4 = **CRIT.**

**Ex.2 — HIGH not CRIT.** Same drain but only works when pool utilization > 90%
(a real but specific state). Impact I4 (bounded, conditional). Likelihood L3.
I4×L3 = **HIGH.** The conditionality pulls it off Critical — and it's recorded
*why*, so the agent can appeal if utilization >90% is actually common.

**Ex.3 — capped by PoC.** Agent claims "reentrancy drains the vault" (claimed
Critical) but the PoC only demonstrates re-entering to mint 1 extra share worth
dust. PoC cap → impact I2, not I5. Likelihood L4. I2×L4 = **LOW.** Note in
verdict: "claimed I5, PoC proves I2; resubmit with a draining PoC to upgrade."

**Ex.4 — out of threat model.** "Owner can call `setFee(100%)` and steal all
fees." `threat_model.trusted_roles` includes `owner` (3/5 multisig). → B.4
guardrail → **Info / out-of-threat-model.** No points.

**Ex.5 — griefing.** Anyone can call `poke()` to force a costly rebalance,
no profit to attacker, victims pay gas + slippage. Impact I3 (griefing).
Likelihood L5. I3×L5 = **HIGH.** (Pure griefing *can* be High when trivially
triggerable — the matrix captures this without the judge having to "feel" it.)

---

## Part C — Recommendations this rubric surfaces

These are changes to `DESIGN.md` that the rubric work argues for:

1. **Make `suggested_fix.patch` mandatory, not bonus.** The fix-equivalence test
   (A.4) is the most reliable dedup signal and it *needs* an applicable patch.
   Without it, every borderline pair falls to expensive, less-reliable stage-3
   LLM adjudication. Trade-off: raises the bar on agents. Recommend mandatory in
   `seeded` phase at minimum, where we can measure if it hurts recall.
2. **Add `threat_model` to `match.config.json`** (trusted_roles,
   accepted_assumptions, out-of-scope) so B.4 guardrails are explicit and
   uniform, not improvised per verdict.
3. **Record both severity axes + per-axis justification in every verdict**
   (already implied by §7.2; make it a required field). This is what makes the
   appeals lane (§11.3) tractable — an appeal challenges a specific axis level,
   not a vibe.
4. **Single orchestrator intake clock** for ordering (A.5) — don't trust
   agent-reported timestamps.
5. **Dedup verdicts get the verifier pool too**, not just severity. A wrong DUP
   call silently zeroes a real finding (a false negative, the §11.3 gap). For
   any DUP that denies points to a finding whose own PoC passed, require a second
   judge to confirm the dup before zeroing it.

---

## Part D — Open edges (still unresolved)

- **Partial-fix dedup.** If patch_C blocks 1 of 2 attack paths in PoC_F, are
  they the same bug? Current lean: DISTINCT if the surviving path is independently
  exploitable; needs examples.
- **Cross-contract root cause.** A bug whose fix could go in either of two
  contracts (caller vs callee). Fingerprint's `faulty_code_site` becomes a set;
  fix-equivalence still works but locality screen (stage 1) may misfire. Flag for
  stage 3.
- **Severity of chained findings.** Two individually-Low bugs that compose into a
  Critical. Who gets the Critical — and is the chain itself a separately
  awardable finding? Proposed: the agent that submits the *composed* PoC gets a
  Critical; the component findings stand as their own Lows. Needs ratification.
