Below are concrete, enforceable improvements to your Ralph reliability spec, designed to snap into the existing 10-phase Ralph skill workflow while directly attacking the “skip / don’t wait / fabricate” failure modes.

I’m grounding this in: APR’s robot-mode + preflight model (machine-parseable, blocking, artifacts) 

README

, the current Ralph 10-phase workflow (esp. phases 4/5/7/10) 

SKILL

, and your reliability-layer spec’s state.json + phase-gates + receipts approach 

RALPH_RELIABILITY_SPEC

.

1) Make “tool calls” impossible except through Ralph wrappers

Problem
Agents can still “do the right thing” in narrative while silently skipping rp-cli / APR / codex calls (or fabricating results) because the skill currently describes direct tool usage in phases 4/5/7/10. 

SKILL

Proposed solution
Amend the skill + spec with a hard rule:

Never call rp-cli, apr, oracle, or codex directly.

Only call:

ralph review design (Phase 4)

ralph oracle (Phase 5)

ralph review plan (Phase 7)

ralph review post (Phase 10)

…and Ralph should print a loud warning if it detects the agent ran rp-cli/apr/codex directly (best-effort detection: shell history hooks optional; minimum viable: grep for new proof artifacts vs expected receipts).

Integration with existing phases
This is a direct swap of the “how” in phases 4/5/7/10 without changing the 10-phase structure. The phases remain the same; only the allowed mechanism changes. 

SKILL

 

RALPH_RELIABILITY_SPEC

Why better than current approach
Right now enforcement depends on “agent discipline.” Your own design principle says to move enforcement into Ralph so the agent can’t “talk its way past” gates. 

RALPH_RELIABILITY_SPEC

 This makes that real: if you didn’t run the wrapper, you literally can’t advance.

2) Split “phase work” from “phase completion” everywhere (mandatory ralph advance)

Problem
Agents can claim phase completion without verifiable evidence. Even with receipts, if the system doesn’t require a distinct “advance” action, agents may drift into the next phase in practice.

Proposed solution
Make this an explicit contract:

Any phase can have “in_progress” work, but completion requires ralph advance <phase>.

advance must refuse unless:

required receipt exists and verifies (or a waiver exists), and

phase prerequisites are satisfied.

This is already sketched in your spec; the improvement is to make it universal and to align it explicitly with the 10-phase skill’s transitions (especially 3→4, 5→6, 6→7, 7→8, 8→9, 9→10). 

RALPH_RELIABILITY_SPEC

Integration with existing phases
Add one “Phase Completion” line to each phase description in the skill:

“Complete this phase by running ralph advance <phase>.”

No structural change to the phases—just a clean, enforceable checkpoint.

Why better than current approach
It eliminates “narrative completion.” If the agent doesn’t have receipts/waivers, it cannot advance the state machine. State.json stays the source of truth (as intended). 

RALPH_RELIABILITY_SPEC

3) Tighten receipts to prove “non-fabrication”: require output content constraints

Problem
A receipt can exist even if the underlying transcript is empty / trivial / unrelated (e.g., tool crashed early but still produced a file). Agents can also fabricate human-readable proof.md if it’s not strictly derived.

Proposed solution
Extend advance verification rules to include minimum “substance” checks:

Transcript must be non-empty and above a minimum byte threshold (e.g., > 2KB for rp-cli reviews; configurable).

For rp-cli phases (4,7), transcript must contain at least one expected marker:

the requested chat_name or mode (“plan”) you already include in receipt identifiers. 

RALPH_RELIABILITY_SPEC

For APR (phase 5), require:

apr robot validate success + apr robot run --wait success (already implied by your design) 

RALPH_RELIABILITY_SPEC

and that the referenced APR output file exists and is non-empty (APR saves outputs to files by design). 

README

For codex post-review, require the review output includes a recognizable header or format marker.

These are pragmatic “tripwires”—not perfect cryptography, but they kill the most common fake/empty proof patterns.

Integration with existing phases
Lives entirely inside ralph verify / ralph advance gate checks. The agent doesn’t need to do anything extra.

Why better than current approach
Right now, receipts are “machine-verifiable” mainly via hashes, but you don’t require that the underlying content demonstrates a real review occurred. 

RALPH_RELIABILITY_SPEC

 These checks directly target fabrication and “tool call theater.”

4) Make background execution safe by construction: “started ≠ done” in state.json

Problem
Agents are impatient: they’ll kick off something async/background and continue. Your spec offers --background + wait, but you need to make “not waiting” impossible

RALPH_RELIABILITY_SPEC
