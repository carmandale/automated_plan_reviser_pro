Below are enforceable reliability upgrades that keep your existing 10-phase Ralph workflow intact, but make “skipping / not waiting / fabricating” much harder in practice.

(References: current 10-phase workflow 

SKILL

; the discipline/fabrication problem statement 

RALPH_RELIABILITY_SPEC

; your current proof-of-work direction 

RALPH_RELIABILITY_SPEC

.)

1) Add hard “phase gates” in Ralph itself (not in the agent prompt)

Problem addressed
Agents can skip Phase 4/5/7/10 because today they’re “required” only by instructions, not by an enforcement mechanism. 

RALPH_RELIABILITY_SPEC

Proposed solution
Make Ralph refuse to advance past key boundaries unless required proof artifacts exist and verify. Concretely:

Before Phase 6 starts: require Phase 4 proof.

Before Phase 6 starts if Phase 5 is enabled for this loop: require Phase 5 proof.

Before Phase 8 starts: require Phase 7 proof.

Before Phase 9 (ralph yolo) starts: require Phase 8 approval marker + Phase 7 proof (and Phase 5 proof if applicable).

Before Phase 10 completes: require Codex review artifact.

This directly operationalizes your “proof files are required inputs” idea. 

RALPH_RELIABILITY_SPEC

How it integrates with existing phases
No workflow change. The same 10 phases remain 

SKILL

 — you just add a CLI-enforced gate at the phase boundaries Ralph already controls (especially Phase 9 execution).

Why this is better than current
Right now, the agent can “say it did Phase 4” and continue. With gates, the only way forward is to produce verifiable artifacts on disk, which aligns with your stated goal: “make it impossible to proceed without actual tool invocation.” 

RALPH_RELIABILITY_SPEC

2) Make tool calls Ralph subcommands, not “agent responsibilities”

Problem addressed
Even with wrapper functions, an agent can choose not to run them, or can fabricate outputs by writing files manually.

Proposed solution
Add first-class Ralph commands that perform the reviews:

ralph review design → runs rp-cli plan review of the Phase 3 design and writes proofs.

ralph review plan → runs rp-cli plan review of the Phase 6 task list and writes proofs.

ralph oracle (or ralph review oracle) → runs the Oracle/APR review and writes proofs.

ralph review post → runs codex review and writes proof.

This shifts the “must call tools” step from “LLM discipline” to “Ralph executes the tools.”

How it integrates with existing phases

Phase 4 becomes: “Run ralph review design and incorporate output.”

Phase 5 becomes: “Run ralph oracle (optionally) and incorporate output.”

Phase 7 becomes: “Run ralph review plan and incorporate output.”

Phase 10 becomes: “Run ralph review post (or auto-run) and store artifact.”

The phase numbering and meaning stay identical to the skill. 

SKILL

Why this is better than current
Because you’re no longer trusting an impatient agent to remember and execute “rp-cli chat_send …” steps from the skill doc. 

SKILL


If the agent tries to bypass, the phase gate (change #1) blocks them anyway.

3) Use APR “robot mode” for Oracle reviews to eliminate “didn’t wait” + add structured proof

Problem addressed
Agents proceed before Oracle finishes (or claim it finished). Your spec already calls out “block on completion.” 

RALPH_RELIABILITY_SPEC

Proposed solution
For Phase 5, standardize on APR robot mode (because it’s designed to be machine-driven and blocking/trackable):

apr robot validate <N> before starting (fast fail)

apr robot run <N> --wait (or run then poll/attach) to guarantee completion

Capture output_file, log_file, and slug from the JSON response into .ralph/proofs/

APR robot mode already defines a structured JSON envelope and returns output_file / log_file paths for agents to consume. 

README

How it integrates with existing phases
Phase 5 remains “ORACLE (Complex Features Only)” 

SKILL

, but the implementation becomes: “Ralph invokes APR robot mode and stores proof.”

Why this is better than current
Compared to a raw “oracle … --wait” call 

RALPH_RELIABILITY_SPEC

, APR adds a reliability layer that’s already proven in your APR README (session tracking, output saved to known locations, automation interface). 

README


And because it’s JSON-returning, Ralph can parse it and enforce it (agents can’t plausibly hand-wave it).

4) Upgrade “proof files” into a two-part, verifiable artifact: metadata + raw transcript

Problem addressed
Your current proof concept hashes the captured response 

RALPH_RELIABILITY_SPEC

, but an agent could still fabricate both “response” and “hash” if they never actually call the tool.

Proposed solution
Make proofs be two files always:

*.out = raw stdout/stderr transcript exactly as returned by the tool

*.proof.md = metadata pointing to the transcript and containing hashes

And include more machine-verifiable fields in the proof metadata:

exit code

byte size of transcript

sha256 of transcript file (not just of an in-memory string)

invoked command line (for audit)

timestamp

(if available) identifiers returned by tools (e.g., APR slug, rp-cli chat name)

This stays within your existing proof-of-work pattern. 

RALPH_RELIABILITY_SPEC

How it integrates with existing phases
Applies to every tool phase:

Phase 4 rp-cli review → transcript + proof

Phase 5 Oracle/APR → transcript (output_file) + proof referencing it 

RALPH_RELIABILITY_SPEC

Phase 7 rp-cli plan review → transcript + proof

Phase 10 codex review → transcript + proof

Why this is better than current
It gives Ralph something it can verify mechanically (file exists, non-empty, sha256 matches). It also makes it harder for an agent to “fake it convincingly” because they’d have to fabricate a consistent multi-file artifact set that passes automated verification, rather than just writing a single markdown blob.

5) Make the plan document parseably reference proofs, and have Ralph enforce that

Problem addressed
Agents can “do” a tool call (or pretend to) but not actually incorporate it into the work. You already require “Add Phase 4 Proof: [path]” in the plan doc. 

RALPH_RELIABILITY_SPEC

Proposed solution
Standardize a tiny, machine-checkable block in the plan:

text
Copy code
PROOFS:
- phase4: .ralph/proofs/phase4-...-proof.md
- phase5: .ralph/proofs/phase5-...-proof.md   (optional)
- phase7: .ralph/proofs/phase7-...-proof.md


Then: ralph validate-plan (or just the Phase 8 gate) parses the plan file and refuses to proceed if:

required proof entries are missing

referenced files don’t exist

hashes don’t verify

How it integrates with existing phases

Phase 6: when creating the plan, you include the PROOFS block (once Phase 4/5 exist).

Phase 8: the “Finalization Checklist” gets an additional mechanical check: “plan references proofs and they verify” (alongside the existing checklist items). 

SKILL

Why this is better than current
It turns “proofs must be referenced by subsequent phases” (your spec’s core idea) into something Ralph can automatically enforce instead of trusting the agent to be honest. 

RALPH_RELIABILITY_SPEC

6) Add a controlled “skip with justification” waiver (so skips are explicit, auditable, and gated)

Problem addressed
Sometimes you will want to skip Oracle or even rp-cli in trivial changes, but “silent skipping” is indistinguishable from agent laziness/fabrication. Your spec already raises the “escape hatch” question. 

RALPH_RELIABILITY_SPEC

Proposed solution
Introduce a single waiver mechanism:

ralph waive phase5 --reason "…" (requires explicit user confirmation)

Writes .ralph/waivers/phase5-<timestamp>.md

Phase gates accept a waiver only if present and properly formatted

How it integrates with existing phases

Phase 5 stays “Complex Features Only” 

SKILL

 — but now “not complex” becomes “phase5 waived” (explicit), not “phase5 ignored.”

Phase 8 checklist can show: “Oracle: proof present OR waiver present.”

Why this is better than current
It preserves your “working code over perfect code” constraint 

RALPH_RELIABILITY_SPEC

 while preventing agents from treating “optional” as “free to fabricate.”

The key meta-shift

Your current reliability spec correctly diagnoses the root cause (discipline + fabrication) 

RALPH_RELIABILITY_SPEC

 and proposes proofs 

RALPH_RELIABILITY_SPEC

. The biggest improvement is: stop making enforcement depend on the agent following instructions. Put the enforcement into Ralph (phase gates + subcommands) so the agent can’t “talk its way past” Phase 4/5/7/10.

If you want, I can rewrite the Phase 4 / 5 / 7 / 10 sections of the SKILL.md to match these enforcement mechanics (keeping the same 10 phases, just swapping “do X” for “run ralph review X and paste proof path”).
