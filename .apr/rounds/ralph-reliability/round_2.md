Below are concrete, enforceable upgrades to your reliability layer spec that snap into the existing Ralph 10-phase workflow, while directly attacking “skip / don’t wait / fabricate.” I’m grounding this in: APR’s existing reliability/robot patterns 

README

, the current 10-phase Ralph skill contract 

SKILL

, and your current reliability-layer spec 

RALPH_RELIABILITY_SPEC

.

1) Make phase progression CLI-only (no “narrated” completion)

Problem addressed
Agents can claim “Phase 4 done” (or “Oracle review done”) without actually running anything, then continue.

Proposed solution
Introduce a single authority for phase transitions:

ralph advance <phase> (or ralph next as the only progression primitive)

advance refuses unless the required proof/waiver exists and verifies

advance writes state.json and is the only way to mark a phase complete

This makes “I did it” meaningless unless Ralph says so.

Integrates with existing phases

Phase 4/5/7/10: completion status is set only by the corresponding ralph review ... / ralph oracle wrapper finishing successfully.

Phase 6/8/9: blocked unless state.json says prior gates satisfied.

Why better than current approach
Your spec already gates before Phase 6/8/9, but agents can still “act like” they’re in later phases conversationally. CLI-only phase advancement removes ambiguity: the agent can’t proceed because the tool that changes phase state refuses.

2) Auto-generate and auto-register proofs (don’t make the agent paste paths)

Problem addressed
Even with proof files, an agent can “forget” to add PROOFS, point to the wrong proof, or fabricate the PROOFS block.

Proposed solution
When Ralph runs a tool wrapper, it should:

Generate the proof artifacts

Update state.json with the proof path(s)

Optionally patch the plan’s ## PROOFS block automatically (idempotent)

So the canonical truth is state.json; the plan’s PROOFS block becomes a human-readable mirror.

Integrates with existing phases

Phase 4/5/7/10 wrappers: write proof + register it

Phase 8 gate: validate against state.json (plan parsing becomes secondary)

Why better than current approach
Your spec currently relies on parsing a plan PROOFS block as part of validation. Automating the insertion/registration removes a common failure mode and further reduces agent degrees of freedom.

3) Standardize proofs as a single “receipt” schema (md is nice, JSON is enforceable)

Problem addressed
Markdown proofs are human-auditable, but machine validation tends to be brittle if fields drift.

Proposed solution
Keep your “two-file” approach, but make the proof metadata a JSON receipt as the primary verifiable unit, with a Markdown summary as a convenience:

bash
Copy code
.ralph/proofs/
  phase4-<ts>-<id>.out
  phase4-<ts>-<id>.receipt.json   # authoritative
  phase4-<ts>-<id>.proof.md       # derived display


Receipt fields (minimal, enforceable):

phase, ts, run_id

command: argv array

exit_code

transcript_path, transcript_bytes, transcript_sha256

tool: { kind: "rp-cli" | "apr" | "codex", identifiers: {...} }

duration_ms

Integrates with existing phases

Phase gates + ralph verify: validate the JSON receipt and transcript hash quickly.

Why better than current approach
Your spec already uses hashes (good). Making JSON the primary format makes validation trivial and robust, while still allowing Markdown for humans.

4) Make “wait” a first-class concept with blocking defaults

Problem addressed
Agents fire-and-forget tools, then keep going (especially Oracle/APR sessions).

Proposed solution

Tool wrapper commands should be blocking by default.

If you allow background mode, it must return a run_id/slug, and the only way forward is:

ralph wait <run_id> (blocks until complete and writes proof)

gates require the completion receipt, not merely “started”

This borrows directly from APR’s “robot run / status / attach” discipline 

README

.

Integrates with existing phases

Phase 5 (ralph oracle): default --wait behavior; background requires explicit ralph wait.

Phase 4/7/10: same pattern (even if they’re fast).

Why better than current approach
It eliminates the “didn’t wait” class entirely. A tool call isn’t “done” until the proof receipt exists.

5) Strengthen rp-cli proofs with tool identifiers you can actually check

Problem addressed
A fabricated transcript can look plausible. You want a “receipt” that’s hard to fake without running rp-cli.

Proposed solution
When ralph review design/plan runs rp-cli, capture and store identifiers that come from rp-cli itself (whatever is realistically available without overengineering), e.g.:

chat name (you already propose)

model/mode used (plan mode)

any returned chat/session id if rp-cli emits one

timestamped start/end markers produced by Ralph itself

Even if rp-cli doesn’t provide a strong ID, you can at least require:

non-empty transcript

transcript contains expected rp-cli output markers

exit code == 0

receipt hashes match

Integrates with existing phases

Phase 4/7 gates: verify tool.kind == rp-cli + marker checks.

Why better than current approach
Your spec mentions “Tool Identifiers” but doesn’t define how Ralph obtains them. Tightening that makes proof verification less hand-wavy while still “working code over perfect code.”

6) Make bypassing costly with lightweight filesystem hardening

Problem addressed
Even if an agent can fabricate files, you want to raise friction so bypass is the rare edge case, not the default.

Proposed solution
After Ralph creates a receipt+transcript, immediately:

chmod 444 the receipt/proof md

chmod 444 the transcript

optionally mark them immutable where available (chflags uchg on macOS) if it doesn’t become a support nightmare

Also record created_mtime in the receipt and verify it hasn’t changed.

Integrates with existing phases
All proof-producing phases (4/5/7/10).

Why better than current approach
This doesn’t make fabrication “impossible,” but it reliably stops the most common “agent edits the proof file” failure mode with almost no complexity.

7) Turn waivers into explicit, scoped “override receipts” (and keep them rare)

Problem addressed
Waivers can become the new bypass: “just waive everything.”

Proposed solution
Upgrade waiver rules:

Waiver is scoped to a specific loop + specific phase + single-use (run_id)

Require a minimum reason length

Store waiver both as a file and in state.json

Gates accept waiver only if confirmed_by == user and confirmed_at exists (as you already intend), but also require reason non-empty and scoped

Integrates with existing phases
Applies to any gated phase; especially Phase 5.

Why better than current approach
Keeps the “escape hatch” without turning it into the default path. Still simple.

8) Remove plan-parsing as a critical dependency (use plan parsing as a lint)

Problem addressed
Parsing markdown to enforce correctness is fragile, and agents can accidentally break formatting.

Proposed solution

Treat ## PROOFS parsing as lint, not the source of truth.

Primary enforcement uses state.json + verified receipts.

ralph validate-plan can still ensure the plan mirrors reality (nice UX), but gates should not depend on markdown parsing.

Integrates with existing phases
Phase 8 “finalize” check becomes: validate state + (optionally) validate plan mirrors state.

Why better than current approach
You keep the human-friendly plan artifact, but the enforcement backbone becomes a stable data model.

9) Make ralph yolo consume the verified state, not the agent’s narrative

Problem addressed
Agents can still “start execution” in their own head without meeting prerequisites.

Proposed solution
Have ralph yolo do a single preflight that prints exactly what it verified:

Phase 4 receipt OK

Phase 5 receipt OK or waiver OK (if enabled)

Phase 7 receipt OK

Phase 8 approved (user confirmation artifact)

Then proceed

If any missing: it prints “BLOCKED BY GATE X” and exits nonzero.

Integrates with existing phases
Phase 9 is your hard wall.

Why better than current approach
Your spec already says “blocked by phase gates” — this just makes it explicit that execution reads state+receipts, not the plan text.

10) Add one command: ralph audit (quick human sanity check)

Problem addressed
Humans need a fast “trust but verify” view, without opening multiple files.

Proposed solution
ralph audit prints a compact table:

Phase -> status -> receipt path -> timestamp -> tool -> exit code -> transcript bytes

Optionally include “open transcript” convenience.

Integrates with existing phases
Supports all gated phases, especially before running ralph yolo.

Why better than current approach
It turns your proofs from “security mechanism” into “good UX,” which increases the chance people actually use them.

One meta recommendation (ties it all together)

Your spec’s core insight is already correct: enforce in Ralph, not in prompts 

RALPH_RELIABILITY_SPEC

. The highest ROI changes above are the ones that reduce agent choice:

CLI-only phase advancement

wrapper commands that block + emit receipts

state.json as source of truth

automatic proof registration

Those four eliminate the common failure modes without overengineering, and they integrate cleanly into your existing 10 phases 

SKILL

 while leveraging APR’s already-proven “robot / wait / artifacts” approach 

README

.
