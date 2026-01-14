# Ralph Reliability Layer Specification

## Problem Statement

AI coding agents (Claude Code, Codex, etc.) when executing Ralph loops are:

1. **Not issuing requests properly** - Skipping or malforming Oracle/rp-cli calls
2. **Not waiting for responses** - Proceeding before tools complete
3. **Fabricating responses** - Making up fake "expert reviews" instead of using tools

This is NOT a network reliability problem. This is an **agent discipline problem**.

## Threat Model

**In scope:**
- Lazy agents that skip steps to save time
- Impatient agents that don't wait for long-running tools
- Accidental skipping due to context loss or misunderstanding
- "Tool call theater" - agents that narrate tool usage without executing

**Out of scope:**
- A fully adversarial agent with filesystem + command execution powers intentionally forging artifacts
- Agents that compromise the Ralph binary itself
- Supply chain attacks on underlying tools (APR, Oracle, rp-cli)

**Escalation path:** If you need adversarial resistance, run Ralph in an isolated executor (container/VM) and store proofs outside the agent's writable workspace.

**Design philosophy:** Ralph's checks are pragmatic tripwires, not cryptographic attestation. They're highly effective against the in-scope failure modes without over-engineering for threats that require different solutions.

## Design Principle

> **Stop making enforcement depend on the agent following instructions. Put the enforcement into Ralph (phase gates + subcommands) so the agent can't "talk its way past" any phase.**

### Anti-Bloat Guardrail

Reliability changes must stay minimal and purpose-driven:
- Add a mechanism only if it closes a failure mode not already covered.
- Prefer removing or simplifying existing checks when adding new ones.
- Avoid redundant layers or optional knobs that expand surface area without clear benefit.
- Optimize for implementability and clarity over theoretical completeness.

## Hard Rules

### All Phases Require Artifacts

Every phase requires on-disk artifacts before `ralph advance` succeeds:

| Phase | Required Artifact | Lint Check |
|-------|-------------------|------------|
| Phase 1 (Research) | `research.md` | Contains `## Files Inspected` with existing paths |
| Phase 2 (Interview) | `interview.md` | Contains 5 categories: Scope, Location, Behavior, Edge Cases, Acceptance |
| Phase 3 (Design) | `design.md` | Contains "Files to create/modify" section |
| Phase 4 (Expert Review) | Receipt from `ralph review design` | Tool-backed verification |
| Phase 5 (Oracle) | Receipt from `ralph oracle` | Tool-backed verification (APR) |
| Phase 6 (Create Plan) | `plan.md` | Passes `ralph lint plan` |
| Phase 7 (Plan Review) | Receipt from `ralph review plan` | Tool-backed verification |
| Phase 8 (Finalize) | Approval receipt from `ralph approve` | TTY + challenge verified |
| Phase 9 (Execute) | All gates pass via `ralph yolo` | Single choke point |
| Phase 10 (Post Review) | Receipt from `ralph review post` | Tool-backed verification |

**Rationale:** This closes the "narrative completion" hole for ALL phases, not just tool phases.

Non-tool phases (1/2/3/6) are complete when the required artifact exists and passes lint. `ralph advance` validates the artifact and updates state, but no additional signed receipt is required for these phases.

### Direct Tool Calls Don't Count

**Primary rule:** Only receipts produced by Ralph (or imported into Ralph) count for gates.

Agents should use Ralph wrappers:
- ✅ `ralph review design` (Phase 4)
- ✅ `ralph oracle` (Phase 5)
- ✅ `ralph review plan` (Phase 7)
- ✅ `ralph review post` (Phase 10)

Direct tool calls (e.g., running `rp-cli` or `apr` directly) may be used for experimentation, but they **never satisfy gates**. Ralph MAY emit a warning when it detects missing receipts for required tool phases.

### Import Restrictions (Verify-Then-Adopt)

**Key principle:** Import is never "trust me." Import is "verify externally, then record a new Ralph receipt."

Import is a privileged action with phase-specific verification:

| Phase | Import Policy |
|-------|--------------|
| Phase 5 | **Verify + adopt** - queries APR robot mode to confirm completion |
| Phase 4, 7, 10 | **TTY-gated** - if tool-backed verify unavailable, requires TTY + challenge |

#### Phase 5 Import (Tool-Backed Verify)

```bash
ralph import-proof phase5 --from .apr/rounds/.../round_7.md
```

**Import process:**
1. Parse the file for `slug`, `output_file`, nonce, hash
2. Run `apr robot history/status` to confirm completion
3. Write a fresh Ralph-signed receipt pointing at the verified session
4. Mark `imported: true` in receipt

#### Phase 4/7/10 Import (TTY-Gated or Tool-Backed)

```bash
ralph import-proof phase4 --from external-review.txt
```

**If tool-backed rp-cli verification exists:**
1. Parse for `chat_id`, nonce, hash
2. Query rp-cli to verify session
3. Write Ralph-signed receipt

**If tool-backed verify unavailable:**
1. Require TTY + challenge (same as waivers)
2. Write receipt with `imported: true`, `tty_verified: true`

**Configuration:**
```yaml
# .ralph/config.yaml
import:
  allow_imported_receipts: true  # Set false to reject all imports at gates
  phase5_verify_before_adopt: true
  phase4_require_tool_backed: false  # If true, disallow TTY fallback
```

**Rationale:** Eliminates "receipt laundering" as a practical bypass. Preserves import ergonomics while centralizing enforcement in Ralph.

### Skill Text Must Use Wrappers

The Ralph skill is the default behavior blueprint for agents. It must **only** reference Ralph wrapper commands for Phases 4/5/7/10 and must state that **only Ralph-generated receipts** satisfy gates.

Required changes to the skill text:
- Replace direct `rp-cli` / `apr` / `codex` examples with `ralph review design`, `ralph oracle`, `ralph review plan`, `ralph review post`.
- Explicitly state: *“Phase completes only when `ralph advance <phase>` succeeds.”*
- Make the **re-review loop** explicit for Phases 4 and 7: review → edit → re-review → advance.

### Phase Completion Requires `ralph advance`

Every phase transition requires explicit CLI confirmation:

```bash
ralph advance <phase>  # The ONLY way to complete a phase
```

`advance` checks:
1. Required artifact exists
2. Artifact passes lint (non-tool phases) or receipt verifies (tool phases)
3. Input hashes match (tool phases)

This applies to ALL phases. Eliminates "narrative completion."

`ralph advance` always writes a **signed completion receipt**, including for non-tool phases (1/2/3/6). Gates require these receipts; file presence alone never completes a phase.

## Core Architecture

### Source of Truth: Receipts (state.json is a cache)

**Receipts and waivers are the source of truth.** `state.json` is a manifest and cache.

`state.json` stores:
- `loop_id`, feature name, policy decisions (`phase5_required`)
- Pointers to receipts and waivers
- Running phase metadata (pid, started_at, expected_nonce)

**Key invariant:** Status is always recomputed from receipt verification. If the receipt doesn't verify, the phase isn't complete—regardless of what `state.json` claims.

On any command that checks status (`ralph status`, `ralph gates`, `ralph advance`), Ralph:
1. Reads receipt pointers from `state.json`
2. Verifies each receipt (hash check, substance check, tool-backed check where applicable)
3. Computes actual status from verification results
4. Updates cached status in `state.json`

**Status values:** `pending` → `running` → `complete` | `failed` (or `waived`)

- **pending**: Not started
- **running**: Tool invoked but not yet complete (background mode)
- **complete**: Receipt exists and verified
- **failed**: Tool started but process died or verification failed
- **waived**: Skipped with explicit waiver

```json
{
  "loop_id": "loop-2026-01-13-abc123",
  "feature": "Console Panel",
  "started": "2026-01-13T10:00:00Z",
  "phase5_required": true,
  "phase5_policy": {
    "rule": "always_unless_waived",
    "decision": "required",
    "evidence": {
      "changed_files": ["src/console.ts", "src/panel.ts"],
      "diff_lines": 247
    }
  },
  "phases": {
    "research": {
      "status": "complete",
      "artifact": "research.md",
      "artifact_sha256": "a1b2c3..."
    },
    "interview": {
      "status": "complete",
      "artifact": "interview.md",
      "artifact_sha256": "b2c3d4..."
    },
    "design": {
      "status": "complete",
      "artifact": "design.md",
      "artifact_sha256": "c3d4e5..."
    },
    "expert_review": {
      "status": "complete",
      "receipt": "proofs/phase4-1736789012-abc123.receipt.json",
      "input_sha256": "c3d4e5..."
    },
    "oracle": {
      "status": "running",
      "pid": 12345,
      "run_id": "apr-ralph-oracle-round-1",
      "started_at": "2026-01-13T12:35:00Z",
      "expected_nonce": "ralph-2f8a9c",
      "input_sha256": "c3d4e5..."
    },
    "create_plan": {"status": "pending"},
    "plan_review": {"status": "pending"},
    "finalize": {"status": "pending"},
    "execute": {"status": "pending"},
    "post_review": {"status": "pending"}
  }
}
```

### Phase 5 Required Policy

`phase5_required` is determined by a **deterministic, boring rule**:

**Default rule:** Phase 5 is always required unless explicitly waived by a TTY-confirmed human.

**Alternative rule (if configured):** Phase 5 required if any file outside a small allowlist changed (e.g., not docs/comments-only), OR if diff > N lines.

The decision is always recorded:
- `rule`: which rule was applied
- `decision`: the computed result
- `evidence`: changed file list, diff stats
- **policy receipt**: a signed record of the decision (project- or loop-scoped)

**Rationale:** Deterministic rules eliminate agent wiggle room. Reliability > optimization.

### CLI-Only Phase Advancement

Phase transitions happen ONLY through Ralph CLI:

```bash
ralph advance <phase>  # Only way to mark a phase complete
```

- `advance` refuses unless required artifact/receipt exists and **verifies**
- `advance` checks input hashes match current artifacts (for tool phases)
- `advance` updates cached status in `state.json`
- Agent saying "Phase 4 done" is meaningless - Ralph must verify

### Phase Gates

| Gate | Requires |
|------|----------|
| Before Phase 4 | Phase 3 artifact (design.md) exists and passes lint |
| Before Phase 6 | Phase 4 receipt verified, input hash matches design.md |
| Before Phase 6 (if Phase 5 enabled) | Phase 5 receipt verified (tool-backed), input hash matches |
| Before Phase 7 | Phase 6 artifact (plan.md) exists and passes `ralph lint plan` |
| Before Phase 8 | Phase 7 receipt verified, input hash matches plan.md |
| Before Phase 9 (`ralph yolo`) | Phase 8 approval receipt + all prior gates |
| Before Phase 10 completes | Codex review receipt |

## Per-Loop Directory Structure

Each loop gets its own directory to prevent cross-loop confusion and receipt reuse:

```
.ralph/
├── loops/
│   └── <loop_id>/
│       ├── state.json
│       ├── research.md              # Phase 1 artifact
│       ├── interview.md             # Phase 2 artifact
│       ├── design.md                # Phase 3 artifact
│       ├── plan.md                  # Phase 6 artifact
│       ├── proofs/
│       │   ├── commit_phase4-<ts>-<id>.json  # Immutable nonce commitment
│       │   ├── phase4-<ts>-<id>.out
│       │   ├── phase4-<ts>-<id>.receipt.json
│       │   └── ...
│       ├── approvals/
│       │   └── phase8-<ts>.json     # Phase 8 approval receipt
│       └── waivers/
│           └── phase5-<ts>.json
├── current                          # Pointer file containing current loop_id
└── config.yaml                      # Verification policies
```

All commands operate on the current loop unless `--loop <id>` is specified.

**Benefits:**
- Receipts are naturally scoped to their loop
- Single-use waivers are trivial to enforce
- Audit becomes cleaner
- No accidental cross-loop receipt reuse

### External Proof Storage (Hardened Option)

**Default:** Proofs are stored in-repo under `.ralph/loops/<loop_id>/proofs/`. This keeps everything self-contained and simple.

**Hardened option:** For isolated executor (container/VM) or CI deployments, configure an external proof root:

```yaml
# .ralph/config.yaml
proof_root: ~/.local/share/ralph/proofs/  # External (hardened)
# Or: RALPH_PROOF_ROOT=~/.local/share/ralph/proofs/
```

**External structure (when hardened):**
```
~/.local/share/ralph/
├── secret.key                  # Signing key (chmod 600)
└── proofs/
    └── <project-slug>/
        └── <loop_id>/
            ├── proofs/
            ├── approvals/
            └── waivers/
```

**Rationale:** Signed receipts are the primary anti-fabrication control. External storage is an additional hardening layer for high-security contexts, not a default requirement.

## Proof System

### Two-File Proof Artifacts

Every tool invocation produces two files (simplified from three):

```
.ralph/loops/<loop_id>/proofs/
├── phase4-<ts>-<id>.out           # Raw stdout/stderr transcript
└── phase4-<ts>-<id>.receipt.json  # Machine-verifiable receipt
```

**Removed:** `.proof.md` (human summary) - now generated on demand via `ralph audit` or `ralph show-proof <phase>`.

**Rationale:** Fewer artifacts = fewer things for agents to manipulate, less repo noise, simpler implementation.

### Immutable Nonce Commitment Files

Before invoking any tool phase, Ralph creates an immutable commitment file:

```bash
# When starting Phase 4:
.ralph/loops/<loop_id>/proofs/commit_phase4-<ts>-<run_id>.json
```

```json
{
  "phase": "phase4",
  "run_id": "abc123",
  "nonce": "ralph-2f8a9c",
  "input_sha256": "c3d4e5...",
  "committed_at": "2026-01-13T12:30:00Z"
}
```

Immediately after creation: `chmod 444 commit_*.json`

**Verification requires:**
- Commitment file exists
- `commitment.nonce == receipt.nonce`
- `commitment.input_sha256 == current_artifact_sha256`

**Rationale:** Makes pre-commit tangible and harder to retrofit. An agent would need to create the commitment file before the tool was supposed to run.

### Receipt Schema (JSON - Primary)

```json
{
  "schema_version": "1.0.0",
  "phase": "phase4",
  "loop_id": "loop-2026-01-13-abc123",
  "ts": 1736789012,
  "run_id": "abc123",
  "nonce": "ralph-2f8a9c",
  "command": ["ralph", "review", "design"],
  "exit_code": 0,
  "transcript_path": "phase4-1736789012-abc123.out",
  "transcript_bytes": 4523,
  "transcript_sha256": "a1b2c3d4e5f6...",
  "inputs": {
    "design_path": "design.md",
    "design_sha256": "c3d4e5f6g7h8...",
    "selected_paths": ["src/console.ts", "src/panel.ts"],
    "selection_sha256": "d4e5f6g7h8i9..."
  },
  "evidence": {
    "kind": "rp-cli",
    "payload_hash": "b7c1..."
  },
  "tool": {
    "kind": "rp-cli",
    "identifiers": {
      "chat_name": "Design Review: Console Panel",
      "mode": "plan"
    }
  },
  "tool_backed_verification": null,
  "imported": false,
  "duration_ms": 45230,
  "created_mtime": 1736789057
}
```

### Non-Tool Phase Completion

Non-tool phases (1/2/3/6) are validated by artifact + lint, not by receipts:

- `ralph advance phase1` → verifies `research.md` exists and passes lint
- `ralph advance phase2` → verifies `interview.md` exists and passes lint
- `ralph advance phase3` → verifies `design.md` exists and passes lint
- `ralph advance phase6` → verifies `plan.md` exists and passes `ralph lint plan`

**Gate rule:** Non-tool phases are complete when `ralph advance` succeeds (artifact exists + lint passes). Status is recorded in `state.json`.

### Input Hash Binding

Receipts bind to the exact version of the artifact that was reviewed:

| Phase | Input Binding |
|-------|---------------|
| Phase 4 | `inputs.design_sha256` - hash of design.md at review time |
| Phase 5 | `inputs.design_sha256` (or bundle hash) |
| Phase 7 | `inputs.plan_sha256` - hash of plan.md at review time |

**Verification rule:** If `sha256(current_file) != receipt.inputs.*_sha256` → receipt invalid → gate blocks

**Rationale:** Prevents "run review, edit artifact, proceed with stale receipt" pattern.

### Review-Edit-Review Workflow

**Important:** Hash binding means the intended workflow is:

1. Phase 3/6 produces a **draft** artifact
2. Phase 4/7 reviews it
3. **Apply feedback** (edit the artifact)
4. **Re-run the review wrapper** to produce receipt for the final artifact
5. Immediately `ralph advance phase4/phase7`

**Key rule:** If you change `design.md` (or `plan.md`) after review, you must re-run the review wrapper; otherwise gates will block.

**CLI remediation:** When `ralph advance phase4/phase7` fails due to hash mismatch, print a single canonical remediation line:
`BLOCKED: artifact changed after review. Run: ralph review design` (or `ralph review plan`).

**Why this matters:** The skill text should not say "run review → process feedback → update artifact → advance." That's logically inverted relative to hash-binding. The correct sequence is:
- Draft → review → edit → **re-review** → advance

This prevents agents from interpreting gate failures as "Ralph being annoying" and trying to bypass. Aligns agent mental model with enforcement model.

**Phase 5 re-review loop:** Oracle output is bound to `design.md` (or bundle hash). If you change `design.md` after reading Oracle feedback, you must re-run `ralph oracle` before advancing. Provide an ergonomic helper:

```bash
ralph oracle --final  # run oracle, then attempt ralph advance phase5 if hash still matches
```

### Artifact Hash Injection + ACK Blocks

**Problem:** Input hashes in receipts prove what we *claimed* to review, but don't force the tool output itself to acknowledge the specific version.

**Solution:** Inject hashes into the tool prompt and require them echoed back.

**When Ralph constructs the rp-cli/APR/codex prompt, inject:**
```
RALPH_NONCE: <nonce>
DESIGN_SHA256: <hash>   # or PLAN_SHA256: <hash>
```

**Instruct reviewer tool to start response with ACK block:**
```
ACK:
RALPH_NONCE: ralph-2f8a9c
DESIGN_SHA256: c3d4e5f6g7h8...
```

**Phase-specific usage:**
| Phase | Hash Injected |
|-------|--------------|
| Phase 4 | `DESIGN_SHA256` |
| Phase 5 | `DESIGN_SHA256` (or bundle hash) |
| Phase 7 | `PLAN_SHA256` |

**Verification requires:**
- ACK block present in transcript
- Nonce in ACK matches expected nonce
- Hash in ACK matches receipt's input hash

**Rationale:** Tightens coupling between "the artifact version" and "the tool output." Improves strength of both local substance checks and any tool-backed transcript retrieval checks.

### Phase 10 Diff Binding

Phase 10 binds post-review to the exact code being reviewed:

- Compute `DIFF_SHA256` from the loop’s commit range (or a stored patch snapshot)
- Inject `DIFF_SHA256` + `RALPH_NONCE` into the post-review prompt
- Require ACK block to echo both values
- Receipt stores `diff_sha256` and `commit_range`

**Verification:** ACK block matches, and current git state/commit range matches the receipt.

### Ralph Transcript Envelope (v1)

Tool transcripts should be wrapped in a Ralph-owned header so verification does **not** depend on tool-specific log formats.

When writing `phaseX-*.out`, Ralph prepends:

```
RALPH_PROOF_HEADER v1
PHASE: phase4
LOOP_ID: <loop_id>
RUN_ID: <run_id>
NONCE: <nonce>
INPUT_SHA256: <design/plan hash>
SELECTED_PATHS_SHA256: <sha256>
SELECTED_PATHS:
 - path/a
 - path/b
TOOL_KIND: rp-cli
TOOL_PATH: /abs/path/to/tool
STARTED_AT: <iso8601>
================ BEGIN TOOL OUTPUT ================
<raw tool stdout/stderr>
================ END TOOL OUTPUT ==================
```

**Verification uses the header + transcript hash** (receipt signs transcript sha256). This avoids brittle parsing of rp-cli logs and makes audits predictable.

### Review Rubric (Advisory)

Tool wrapper prompts request a structured response format:

Expected sections for phases 4/5/7/10:
- `ACK:` (nonce + hash) — **required for gate, blocks if missing**
- `SUMMARY:` (brief bullets)
- `RISKS:` (identified risks)
- `RECOMMENDATIONS:` (actionable items)
- `BLOCKERS:` (blocking issues)
- `FOLLOW-UPS:` (next steps)

**Verification:**
- **ACK block**: Required. Gates block if ACK is missing or nonce/hash don't match.
- **Rubric sections**: Advisory. Ralph MAY report warnings in `ralph audit` if sections are missing or sparse, but gates do not block solely on rubric bullet counts.

**Rationale:** ACK block binding (nonce + hash) is the primary anti-fabrication control. Bullet-counting is brittle and can incentivize padding rather than quality.

### Auto-Select rp-cli Context

Ralph wrappers automatically select context files for rp-cli:

**`ralph review design` (Phase 4):**
1. Parse `design.md` for file paths in "Files to modify/create" section
2. Resolve existing paths (ignore non-existent "to create" paths)
3. Call `rp-cli manage_selection` automatically
4. Store `selected_paths[]` + `selection_sha256` in receipt

**`ralph review plan` (Phase 7):**
1. Parse `plan.md` "Files Changed" section
2. Same process as above

**Verification requires:**
- `selected_paths` in receipt with at least N existing files
- `selection_sha256` matches receipt

**Rationale:** Removes high-variance "agent judgment step" that often gets skipped.

### Phase 5 Receipt (Tool-Backed Verification)

Phase 5 receipts include APR's robot mode response for tool-backed verification:

```json
{
  "schema_version": "1.0.0",
  "phase": "phase5",
  "loop_id": "loop-2026-01-13-abc123",
  "ts": 1736789500,
  "run_id": "def456",
  "nonce": "ralph-8b3d2e",
  "command": ["ralph", "oracle"],
  "exit_code": 0,
  "transcript_path": "phase5-1736789500-def456.out",
  "transcript_bytes": 8291,
  "transcript_sha256": "c3d4e5f6g7h8...",
  "inputs": {
    "design_sha256": "c3d4e5f6g7h8..."
  },
  "tool": {
    "kind": "apr",
    "apr_response": {
      "slug": "apr-ralph-oracle-round-1",
      "output_file": ".apr/rounds/ralph-oracle/round_1.md",
      "log_file": ".apr/logs/oracle_apr-ralph-oracle-round-1.log",
      "status": "completed"
    },
    "output_file_sha256": "d4e5f6g7h8i9...",
    "log_file_sha256": "e5f6g7h8i9j0..."
  },
  "tool_backed_verification": {
    "apr_robot_status": "ok",
    "session_confirmed": true,
    "nonce_found_in_output": true
  },
  "imported": false,
  "duration_ms": 1845230
}
```

### Substance Checks (Anti-Fabrication)

`ralph advance` verifies receipts have real content beyond simple markers:

#### Basic Checks (Minimum)

| Phase | Minimum Bytes | Required Markers |
|-------|---------------|------------------|
| Phase 4 (rp-cli) | 2048 | `chat_name` or `"mode": "plan"`, `RALPH_NONCE:` |
| Phase 5 (APR) | 1024 | APR output file exists, `RALPH_NONCE:` present |
| Phase 7 (rp-cli) | 2048 | `chat_name` or `"mode": "plan"`, `RALPH_NONCE:` |
| Phase 10 (codex) | 512 | Recognizable review header/format |

#### Structure Checks (Strengthened)

**Problem:** Marker checks are easy to fake with a single fabricated blob.

**Solution:** Add structure-level checks that are hard to fake accidentally and annoying to fake deliberately:

**For rp-cli phases (4/7), require transcript evidence of:**
- A `manage_selection` call was made
- A `chat_send` call was made
- Minimum count of selected files that actually exist
- `selection_sha256` matches receipt (or selected paths appear in transcript)
- ACK block with `DESIGN_SHA256:` / `PLAN_SHA256:` (see below)

**For APR phase (5), additionally require:**
- `apr_response.slug` appears in transcript
- Transcript includes at least one APR "completed/running" status line
- Output file mtime is **after** commitment file mtime
- ACK block includes expected nonce + hash
- `duration_ms` is non-zero and within sane bounds
- Tool-side status confirms **completed** when available (`apr status` / `oracle status`)

**For Phase 10, additionally require:**
- ACK block includes `DIFF_SHA256`
- Receipt contains `commit_range` or patch snapshot hash

```yaml
# .ralph/config.yaml
verification:
  phase4:
    structure_checks:
      require_manage_selection: true
      require_chat_send: true
      min_selected_files: 2
      require_ack_block: true
      # rubric_checks: advisory (warn in audit, don't block)
  phase7:
    structure_checks:
      require_manage_selection: true
      require_chat_send: true
      min_selected_files: 2
      require_ack_block: true
      # rubric_checks: advisory
  phase5:
    structure_checks:
      require_slug_in_transcript: true
      require_status_line: true
      require_ack_block: true
      # rubric_checks: advisory
```

**Rationale:** Makes "fabricate one fake paragraph" insufficient; you'd need to fabricate a multi-step, internally consistent transcript—exactly what lazy agents don't do.

**Note:** `RALPH_NONCE:` is now required for Phase 4 and Phase 7 (not just Phase 5).

### Tool-Backed Verification

Tool phases query external registries, not just local files:

#### Phase 5 (APR/Oracle)

```bash
# During ralph advance phase5 (and ralph yolo preflight):
apr robot status                    # Confirm APR is available
apr robot history                   # Confirm session slug exists and completed
# Verify output file matches APR's reported path and contains nonce
```

**If APR can't be queried:** Phase 5 is BLOCKED, not soft-pass.

#### Phases 4 & 7 (rp-cli)

**Goal:** Make rp-cli phases "tool-backed" like Phase 5 to prevent fabrication.

**A. Capture stable rp-cli identifier at runtime:**
When `ralph review design` runs `rp-cli chat_send`, capture the stable identifier returned (e.g., `chat_id`, `conversation_id`, or slug). Store in receipt:
```json
{
  "tool": {
    "identifiers": {
      "chat_id": "abc123-def456",
      "chat_name": "Design Review: Console Panel"
    }
  }
}
```

**B. Verify by querying rp-cli after the fact:**
During `ralph advance phase4` / `ralph yolo preflight`:
1. Call a read-only rp-cli command (e.g., `rp-cli get_chat`, `export_chat`, or `list_chats`) to retrieve chat metadata
2. Verify the returned transcript contains:
   - Exact `RALPH_NONCE:` value
   - Artifact hash (see ACK block below)
   - Sane timestamps (chat created after commitment)

**C. Fallback if rp-cli lacks query capability:**
If no tool-backed verify exists → force TTY-gated import/waiver for non-interactive acceptance. This stays aligned with the threat model (lazy agents are in scope, adversarial is out of scope).

```yaml
# .ralph/config.yaml
verification:
  phase4:
    tool_backed_verify: true          # Try rp-cli query
    fallback_to_tty_gate: true        # If query unavailable, require TTY
  phase7:
    tool_backed_verify: true
    fallback_to_tty_gate: true
```

**Rationale:** Upgrades Phases 4/7 from "local file plausibility checks" to "prove the tool session exists," mirroring why Phase 5 is hard to fake.

### Data-Driven Verification Rules

Verification thresholds are defined in config, not scattered in code:

```yaml
# .ralph/config.yaml
verification:
  phase1:
    required_artifact: research.md
    required_sections:
      - "## Files Inspected"
    min_file_paths: 3
  phase2:
    required_artifact: interview.md
    required_sections:
      - "Scope"
      - "Location"
      - "Behavior"
      - "Edge Cases"
      - "Acceptance"
  phase3:
    required_artifact: design.md
    required_sections:
      - "Files to"
  phase4:
    min_bytes: 2048
    required_patterns:
      - "chat_name"
      - "RALPH_NONCE:"
    requires_input_binding: true
  phase5:
    min_bytes: 1024
    requires_tool_backed_verify: true
    required_patterns:
      - "RALPH_NONCE:"
    requires_input_binding: true
  phase6:
    required_artifact: plan.md
    requires_lint: true
  phase7:
    min_bytes: 2048
    required_patterns:
      - "chat_name"
      - "RALPH_NONCE:"
    requires_input_binding: true
  phase8:
    requires_approval: true
  phase10:
    min_bytes: 512
    required_patterns:
      - "review"
```

`ralph verify --explain` prints exactly which rule failed.

### Verification Levels (MVP → Hardened)

Allow a single switch to control verification strictness:

```yaml
# .ralph/config.yaml
verification_level: mvp   # mvp | standard | hardened
```

**MVP (ship first):**
- Signed receipts (tool phases 4/5/7/10)
- Input hash binding
- ACK block required (nonce + hash)
- Min-bytes thresholds
- Phase 5 tool-backed verification (APR robot)
- Rubric checks: advisory (warn, don't block)

**Standard:**
- Auto-context selection checks (selected_paths exist + minimum count)
- Structure checks (manage_selection/chat_send evidence)

**Hardened:**
- Tool-backed rp-cli verification (export chat by id)
- Tamper-evident proof index required

In **agent mode**, require `standard` or `hardened` by default.

### Filesystem Hardening

After Ralph creates proof artifacts:
```bash
chmod 444 *.receipt.json *.out commit_*.json
```
Raises friction for agent editing proofs. Not impossible to bypass, but stops common failure modes.

### Ralph-Signed Receipts

**Problem:** Agents can write arbitrary `.receipt.json` files that satisfy marker/byte thresholds.

**Solution:** Add a lightweight signing step so hand-written receipts don't verify:

**Setup (on `ralph init`):**
```bash
# Generate or load machine-local secret (outside repo)
~/.local/share/ralph/secret.key  # chmod 600
```

**Receipt signing:**
When Ralph writes any receipt (phase4/5/7/10, approvals, waivers):
1. Canonicalize JSON (stable key ordering)
2. Compute `sig = HMAC-SHA256(secret, canonical_json_without_sig)`
3. Store `sig` in the receipt

```json
{
  "schema_version": "1.0.0",
  "phase": "phase4",
  ...
  "sig": "a1b2c3d4e5f6..."
}
```

**Verification rule:** Any receipt without a valid signature is rejected.

**Threat model alignment:** This is not perfect crypto attestation (an adversarial agent could exfiltrate the key), but per the threat model it's a high-value tripwire against lazy fabrication. An agent must either (a) actually run Ralph wrappers, or (b) do key theft + custom signer—far beyond "impatient/skip" behavior.

### Tamper-Evident Proof Index

Maintain an append-only proof index outside the repo:

```
~/.local/share/ralph/proofs/<project>/index.json
```

Each entry includes receipt filename, sha256, timestamp, and `prev_index_sha256`. The index itself is signed.

**Verification rule:** If an expected receipt is missing from the chain, verification fails.

**Rationale:** Cheap integrity win that makes “rewrite history” and accidental deletion obvious during audits.

### Auto-Registration

When Ralph runs a tool wrapper, it:
1. Creates immutable commitment file
2. Generates proof artifacts (out + receipt.json)
3. Updates state.json with receipt path
4. Makes files read-only

No agent action required - proofs are automatically registered.

## Plan Linter

### Scaffold Templates (Non-Tool Phases)

To reduce lint/parser flakiness, Ralph provides canonical scaffolds for non-tool phases:

```bash
ralph scaffold research
ralph scaffold interview
ralph scaffold design
ralph scaffold plan
```

Each scaffold contains fixed headings and stable markers (e.g., a required “Files to modify/create” table) so linting and auto-selection are reliable.

**Gating:** Using the scaffold is not mandatory, but lint rules assume the scaffold’s structure. Agents should run `ralph scaffold <phase>` before drafting artifacts.

**Rationale:** Makes the compliant path easier than improvising formatting.

### `ralph lint plan`

A lightweight linter that enforces plan hygiene:

**Checks:**
- Every task line contains WHAT + at least one file path pattern OR a `WHERE:` field
- Every task contains `VERIFY:`
- "Files Changed" table exists
- Task count > 0
- Optional: flag tasks over max character count (proxy for "too big")

**Gating:**
- `ralph advance create_plan` requires lint pass
- `ralph review plan` wrapper refuses to run if lint fails
- `ralph yolo` preflight refuses if lint fails

```bash
ralph lint plan                    # Check plan.md
ralph lint plan --fix              # Suggest fixes (future)
ralph lint plan --explain          # Show all failed rules
```

**Rationale:** Converts "best practice" into an enforceable rule. Improves downstream reliability more than almost any other low-effort change.

## Tool Wrapper Commands

### Blocking by Default

All tool wrappers block until completion:

| Command | Phase | Behavior |
|---------|-------|----------|
| `ralph review design` | 4 | Runs rp-cli, blocks, writes proof |
| `ralph oracle` | 5 | Runs APR, blocks until completion evidence, writes proof |
| `ralph review plan` | 7 | Runs rp-cli, blocks, writes proof |
| `ralph review post` | 10 | Runs codex review, blocks, writes proof |

### Per-Phase Locks (Prevent Duplicate Runs)

To prevent duplicate tool runs, each wrapper creates a per-phase lock (stored in `state.json` or a `.lock_phaseN` file) containing pid/run_id/slug. A second run refuses to start if a live lock exists. If the lock is stale (pid dead), allow `ralph retry <phase>` to clear and re-run.

### Auto-Context Selection

Tool wrappers automatically select context:

```bash
ralph review design   # Parses design.md, selects files, runs rp-cli
ralph review plan     # Parses plan.md, selects files, runs rp-cli
```

Manual override:
```bash
ralph review design --files src/foo.ts src/bar.ts  # Override auto-selection
```

### Ensure Reviews (Ergonomic Helper)

**Problem:** Agents skip steps because step-count is friction. Gates prevent harm but don't make success easy.

**Solution:** Ergonomic command that runs missing review wrappers:

```bash
ralph ensure-reviews  # or: ralph run-reviews
```

**Behavior:**
1. Checks gates for phase4/5/7/10 in order
2. Runs missing wrappers blocking:
   - Phase 4: `ralph review design`
   - Phase 5: `ralph oracle` (if required)
   - Phase 7: `ralph review plan`
   - Phase 10: `ralph review post` (if applicable)

**Default path for `ralph yolo`:** When `RALPH_YOLO_AUTO=1` (agent/CI contexts), `ralph yolo` should call `ralph ensure-reviews` automatically if blocked, then re-check gates before proceeding. Otherwise, keep the explicit `--auto` flag.
3. Re-checks gates and prints pass/fail summary

**Integration with yolo:**
```bash
ralph yolo --auto  # Calls ensure-reviews automatically when blocked
```

**Rationale:** You're not just blocking skipping—you're making compliance the path of least resistance. Especially effective against "impatient" agents: one command, fewer chances to wander.

### `--final` (Atomic Review + Advance)

For Phases 4/5/7/10, `--final` runs the wrapper, verifies the receipt, and immediately attempts `ralph advance phaseN`. If the input hash changed, it fails with a single remediation line.

### Background Mode (Explicit)

If background mode is allowed:
```bash
ralph oracle --background  # Returns run_id, sets status="running", pre-commits nonce
ralph wait <run_id>        # Blocks until complete, writes proof, sets status="complete"
```

**Agent mode:** `--background` is disallowed (hard-fail).

**State transitions:**
1. `--background` → status becomes `"running"` with `pid`, `run_id`, `started_at`, `expected_nonce`
2. `ralph wait` → blocks until tool completes **and** completion evidence passes (output exists, ACK present, mtime after commitment, tool status completed), then status becomes `"complete"` with `receipt`
3. `ralph advance` → refuses if status is `"running"` (must wait first)

**Key invariant:** A tool call isn't "done" until the receipt exists. "Started" ≠ "done".

While any phase is `"running"`, Ralph blocks all phase advancement until `ralph wait` completes.

### Stale Run Detection

Ralph detects and handles stale/crashed runs:

**Detection (in `ralph status`, `ralph gates`, `ralph wait`):**
- If `status == "running"` and `pid` no longer exists → mark as `"failed"`
- For Phase 5: also query `apr robot status` / `oracle status`:
  - If slug is complete → offer to finalize (generate receipt)
  - If slug missing/failed → mark as `"failed"`

**Recovery:**
```bash
ralph retry <phase>  # Archives stale run metadata, starts fresh with new nonce
```

**Rationale:** Background mode failures are common in practice. Without explicit handling, agents get wedged in "running" forever.

### APR Robot Mode for Oracle (Blocking Semantics)

Phase 5 must be **hard-blocking** by default: no receipt is written until completion evidence exists.

Recommended implementation (pick the simplest that works reliably):
- **Option A:** `apr run <N> --wait` (non-robot) + deterministic output path
- **Option B:** `apr robot run <N>` + poll until output exists and status is **completed**

```bash
# Ralph internally runs:
apr robot validate 1 -w ralph-oracle  # Fast fail (pre-flight)
apr robot run 1 -w ralph-oracle       # Returns running
# Poll until output_file exists, ACK block present, and status = completed
```

**Invariant:** Phase 5 receipt is only written after completion evidence (output file + ACK + status).

In **agent mode**, if APR status/history cannot be queried, Phase 5 remains **BLOCKED**.

## Phase 8 Approval System

### `ralph approve` Command

Phase 8 approval is a first-class receipt, not a narrative step:

```bash
ralph approve  # TTY required
```

**Process:**
1. Prints gate status + plan hash + design hash
2. Requires typing a challenge string (same pattern as waivers)
3. Writes approval receipt

**Approval Receipt:**
```json
{
  "phase": "phase8",
  "loop_id": "loop-2026-01-13-abc123",
  "ts": 1736790000,
  "design_sha256": "c3d4e5...",
  "plan_sha256": "d4e5f6...",
  "lint_passed": true,
  "tty": true,
  "user": "$USER",
  "host": "hostname",
  "pwd": "/path/to/project",
  "confirmed_at": "2026-01-13T12:50:00Z",
  "challenge_response": "APPROVE PLAN x8m3k"
}
```

**Gate rule:** `ralph yolo` requires **either**:
- Phase 8 approval receipt with matching hashes, **or**
- `allow-auto-yolo` policy receipt with matching hashes (for agent/CI contexts)

**Rationale:** Prevents "approval theater." Ensures approval is for this specific plan/design version.

## Waiver System

### TTY-Gated Overrides

**All override actions require interactive TTY:**
- `ralph waive <phase>`
- `ralph approve` (Phase 8)
- `ralph import-proof phase4|phase7|phase10`
- `ralph yolo --ignore-plan-lint`
- Any future `--force` bypass flags

**Rules:**
1. If stdin is not a TTY → **hard fail** (exit non-zero)
2. If TTY is present → require deliberate action:
   - Retype the waiver reason
   - Type a one-time challenge: `TYPE "WAIVE PHASE5 <random>"`

**Rationale:** Most agent runners execute non-interactively. TTY-gating crushes the lazy/automatic bypass path.

## Agent Mode (Fail-Closed)

If stdin is not a TTY, Ralph enters `RALPH_MODE=agent` automatically.

In agent mode:
- **Disable** `ralph waive`, `ralph approve`, `ralph import-proof` unless a **signed policy receipt** explicitly allows the exception.
- **Disallow** background runs for tool phases (`--background` hard-fails).
- **Require** tool-backed verification wherever available. If unavailable, the phase remains **BLOCKED** (no soft pass).
- Wrappers are **blocking-only**: they return only after a verified receipt is written (or fail).

**Rationale:** Enforces "skip / don't wait / fabricate" defenses in the environments where agents are most likely to misbehave.

### Phase 5 (APR/Oracle) in Agent Mode

If Oracle/APR cannot be reliably launched in the current (non-TTY/agent) environment, `ralph oracle` MUST fail fast with code `tool_unavailable` and actionable instructions:

```json
{
  "ok": false,
  "code": "tool_unavailable",
  "hint": "APR/Oracle cannot be reliably launched in agent mode. Human intervention required.",
  "instructions": [
    "1. Run APR/Oracle in a TTY/human session: apr robot run N -w <workflow>",
    "2. Note the resulting slug and output_file",
    "3. Use: ralph import-proof phase5 --from <output_file>"
  ]
}
```

**Rationale:** Agent-initiated APR runs may report success but never actually start. Rather than letting the agent believe Phase 5 succeeded, fail fast and direct to human-verified import path.

### Waiver Receipt

```json
{
  "phase": "phase5",
  "loop_id": "loop-2026-01-13-abc123",
  "ts": 1736789012,
  "reason": "Single-line typo fix, no architecture impact",
  "tty": true,
  "user": "$USER",
  "host": "hostname",
  "pwd": "/path/to/project",
  "confirmed_at": "2026-01-13T12:30:12Z",
  "challenge_response": "WAIVE PHASE5 x7k2m"
}
```

Gates accept: **Receipt present OR Waiver present** (not silent skip)

### Scoped, Single-Use Waivers

Waivers are tied to specific loop + phase:

```bash
ralph waive phase5 --reason "Single-line typo fix, no architecture impact"
# Requires TTY, challenge typing, stored in current loop's waivers/ directory
```

**Requirements:**
- Interactive TTY (non-negotiable)
- Challenge response
- Minimum reason length (non-empty)
- Scoped to current loop only
- Single-use (can't reuse for next loop)

## Policy Receipts (Agent-Mode Exceptions)

Policy changes are not convenience flags. They require signed, TTY-gated receipts.

```bash
ralph allow-auto-yolo --reason "CI pipeline for repo X"
```

**Rules:**
- Requires interactive TTY + challenge (same as waivers)
- Writes a signed policy receipt (project-scoped or loop-scoped)
- `RALPH_YOLO_AUTO=1` only works when an **allow-auto-yolo** policy receipt exists and required hashes match

**Rationale:** Removes "set env var and bypass" failure modes. Aligns policy changes with the same trust model as waivers/approvals.

## Robot Mode (JSON API)

Ralph provides a machine-friendly JSON API mirroring APR's proven pattern:

```bash
ralph robot status           # Environment and loop status
ralph robot gates            # Gate status (what's blocking)
ralph robot run-phase <N>    # Execute a phase
ralph robot verify-all       # Verify all receipts
ralph robot yolo-preflight   # Pre-flight check only (no execution)
ralph robot lint-plan        # Plan lint as JSON
```

### Response Format

All robot commands return a consistent envelope:

```json
{
  "ok": true,
  "code": "ok",
  "data": { ... },
  "hint": "Optional helpful message",
  "meta": {
    "v": "1.0.0",
    "ts": "2026-01-13T12:30:00Z"
  }
}
```

**Error codes:**
- `ok` - Success
- `not_configured` - No `.ralph/` directory
- `not_found` - Resource doesn't exist
- `validation_failed` - Receipt verification failed
- `gate_blocked` - Gate not satisfied
- `lint_failed` - Plan lint failed
- `input_hash_mismatch` - Artifact changed after review
- `tty_required` - Override requires interactive TTY
- `tool_unavailable` - External tool (APR, rp-cli) not available

**Rationale:** Agents can't "interpret" gates—they must consume structured output. This eliminates natural-language compliance issues.

## `ralph yolo` - The Single Choke Point

`ralph yolo` is the unavoidable execution gateway. All gates are verified through this single choke point.

### Preflight (Mandatory)

Before execution, `ralph yolo` verifies:

```
Preflight Check:
✓ Phase 1 artifact: research.md (verified, 5 files inspected)
✓ Phase 2 artifact: interview.md (verified, 5 categories)
✓ Phase 3 artifact: design.md (verified)
✓ Phase 4 receipt: proofs/phase4-....receipt.json (verified, input hash match)
✓ Phase 5 receipt: proofs/phase5-....receipt.json (verified, tool-backed)
✓ Phase 6 artifact: plan.md (lint passed)
✓ Phase 7 receipt: proofs/phase7-....receipt.json (verified, input hash match)
✓ Phase 8 approval: approvals/phase8-....json (verified, hashes match)

All gates satisfied. Proceeding with execution...
```

**If any check fails:** `BLOCKED BY: <specific failure>` and exits nonzero.

**If any phase is "running":** Refuses and tells agent to `ralph wait <run_id>`.

### Gate Check Implementation

`ralph robot gates` and `ralph robot yolo-preflight` return the same internal gate-check result:

```json
{
  "ok": false,
  "code": "gate_blocked",
  "data": {
    "blocked_by": "phase7",
    "reason": "input_hash_mismatch",
    "details": {
      "receipt_hash": "a1b2c3...",
      "current_hash": "d4e5f6...",
      "artifact": "plan.md"
    }
  }
}
```

**Rationale:** Makes it impossible for agents to "talk their way into execution."

## Command Reference

### Core Commands

```bash
ralph init <feature-name>      # Initialize new ralph loop
ralph status                   # Show current state (recomputes from receipts)
ralph advance <phase>          # Mark phase complete (verifies artifact/receipt first)
ralph gates                    # Show gate status (what's blocking)
ralph yolo                     # Execute (preflight verifies ALL gates)
```

### Scaffold Commands

```bash
ralph scaffold research        # Create canonical research.md template
ralph scaffold interview       # Create canonical interview.md template
ralph scaffold design          # Create canonical design.md template
ralph scaffold plan            # Create canonical plan.md template
```

### Review Commands (Tool Wrappers)

```bash
ralph review design            # Phase 4: rp-cli design review (blocking, auto-context)
ralph oracle                   # Phase 5: APR/Oracle review (blocking)
ralph oracle --final           # Run oracle, then attempt ralph advance phase5
ralph review plan              # Phase 7: rp-cli plan review (blocking, auto-context)
ralph review post              # Phase 10: codex review (blocking)
ralph ensure-reviews           # Run all missing review wrappers in order
ralph review design --final    # Run review, then attempt ralph advance phase4
ralph review plan --final      # Run review, then attempt ralph advance phase7
ralph review post --final      # Run review, then attempt ralph advance phase10
```

### Approval Command

```bash
ralph approve                  # Phase 8: TTY-gated approval with challenge
```

### Background & Wait

```bash
ralph oracle --background      # Returns run_id, pre-commits nonce
ralph wait <run_id>            # Block until complete
ralph retry <phase>            # Recover from stale/failed run
```

### Verification Commands

```bash
ralph verify <receipt>         # Verify single receipt
ralph verify-all               # Verify all receipts for current loop
ralph verify --explain         # Show which verification rule failed
ralph lint plan                # Lint plan.md
ralph lint plan --explain      # Show all lint failures
ralph validate-plan            # Check plan mirrors state (deprecated, use lint)
ralph audit                    # Human-friendly table of all proofs
ralph show-proof <phase>       # Render human summary on demand
```

### Import Commands

```bash
ralph import-proof phase5 --from <path>   # Free import (APR verification)
ralph import-proof phase4 --from <path>   # TTY-gated import
```

### Waiver Commands (TTY Required)

```bash
ralph waive <phase> --reason "..."   # Skip phase (requires TTY + challenge)
ralph waivers                        # List active waivers
```

### Policy Commands (TTY Required)

```bash
ralph allow-auto-yolo --reason "..."      # Signed policy receipt to allow auto-yolo
```

### Robot Mode (JSON API)

```bash
ralph robot status             # {"ok": true, "data": {...}}
ralph robot gates              # Gate status as JSON
ralph robot run-phase <N>      # Execute phase, return JSON
ralph robot verify-all         # Verify all receipts
ralph robot yolo-preflight     # Pre-flight only
ralph robot lint-plan          # Plan lint as JSON
ralph robot help               # API documentation
```

## `ralph audit` Output

Quick human sanity check:

```
┌─────────┬──────────┬─────────────────────────────────┬─────────────┬────────┬──────┬───────┬────────────┐
│ Phase   │ Status   │ Artifact/Receipt                │ Timestamp   │ Tool   │ Exit │ Bytes │ Input Hash │
├─────────┼──────────┼─────────────────────────────────┼─────────────┼────────┼──────┼───────┼────────────┤
│ phase1  │ complete │ research.md                     │ 10:15:00    │ -      │ -    │ 2.1K  │ -          │
│ phase2  │ complete │ interview.md                    │ 10:20:00    │ -      │ -    │ 1.8K  │ -          │
│ phase3  │ complete │ design.md                       │ 10:45:00    │ -      │ -    │ 3.2K  │ -          │
│ phase4  │ complete │ phase4-1736789012-abc123.json   │ 12:30:12    │ rp-cli │ 0    │ 4523  │ ✓ match    │
│ phase5  │ complete │ phase5-1736789500-def456.json   │ 12:38:20    │ apr    │ 0    │ 8291  │ ✓ match    │
│ phase6  │ complete │ plan.md (lint: ✓)               │ 12:42:00    │ -      │ -    │ 4.1K  │ -          │
│ phase7  │ complete │ phase7-1736790000-ghi789.json   │ 12:46:40    │ rp-cli │ 0    │ 3892  │ ✓ match    │
│ phase8  │ complete │ phase8-1736790100.json          │ 12:48:20    │ -      │ -    │ -     │ ✓ match    │
│ phase10 │ pending  │ -                               │ -           │ -      │ -    │ -     │ -          │
└─────────┴──────────┴─────────────────────────────────┴─────────────┴────────┴──────┴───────┴────────────┘
```

## Implementation Phases

### Phase 1: Core Infrastructure
- [ ] Per-loop `.ralph/loops/<loop_id>/` directory structure
- [ ] `state.json` as cache, receipts as truth
- [ ] Receipt schema + generation with input hashes
- [ ] Hash verification
- [ ] Pre-committed nonces with immutable commitment files
- [ ] Non-tool phase artifact requirements
- [ ] Signed receipts for all proofs/approvals/waivers/imports
- [ ] In-repo proof storage (with optional external hardening)
- [ ] Scaffold templates for research/interview/design/plan
- [ ] Policy receipts (phase5_required decisions)
- [ ] Tamper-evident proof index (append-only, signed)

### Phase 2: Tool Wrappers
- [ ] `ralph review design` (rp-cli + proof + auto-context selection)
- [ ] `ralph oracle` (APR robot mode + proof + tool-backed verification)
- [ ] `ralph review plan` (rp-cli + proof + auto-context selection)
- [ ] `ralph review post` (codex + proof)
- [ ] Per-phase locks to prevent duplicate runs
- [ ] Ralph transcript envelope (header + delimiters)
- [ ] ACK block enforcement (nonce + hash binding)
- [ ] Review rubric advisory checks (warn in audit, don't block)
- [ ] Phase 5 blocking semantics (robot run + poll or apr run --wait)
- [ ] `ralph oracle --final` helper (run + advance when hash matches)
- [ ] `--final` helpers for review design/plan/post

### Phase 3: Phase Gates
- [ ] Gate checking logic with receipt verification
- [ ] Input hash binding verification
- [ ] `ralph advance` command (verifies artifacts/receipts before advancing)
- [ ] Tool-backed verification for Phase 5
- [ ] `ralph yolo` as single choke point
- [ ] Auto-run `ensure-reviews` when configured (`RALPH_YOLO_AUTO=1`)
- [ ] Agent mode detection (non-TTY fail-closed)

### Phase 4: Verification & Audit
- [ ] `ralph verify` / `ralph verify-all`
- [ ] `ralph verify --explain` (data-driven rules)
- [ ] `ralph lint plan` with all checks
- [ ] `ralph audit` table output with input hash status
- [ ] `ralph show-proof` (on-demand human summary)

### Phase 5: Approval & Waiver System
- [ ] TTY detection and challenge
- [ ] `ralph approve` command with artifact hash binding
- [ ] `ralph waive` with confirmation
- [ ] Waiver receipt generation
- [ ] Gate integration (proof OR waiver)
- [ ] `ralph allow-auto-yolo` policy receipt

### Phase 6: Robot Mode
- [ ] `ralph robot status/gates/verify-all`
- [ ] `ralph robot run-phase`
- [ ] `ralph robot yolo-preflight`
- [ ] `ralph robot lint-plan`
- [ ] Consistent JSON envelope with detailed error codes

### Phase 7: Recovery & Import
- [ ] Stale run detection
- [ ] `ralph retry` command
- [ ] `ralph import-proof` with TTY-gating for non-Phase-5

## Success Criteria

1. **All phases are artifact-gated** - No phase completes without verifiable work product
2. **Receipts are verifiable** - SHA256 hashes match, substance checks pass, tool-backed verification for Phase 5
3. **Input binding works** - Reviews are invalidated if artifacts change afterward
4. **Phases are gated** - `ralph yolo` fails without verified receipts/artifacts
5. **Skips are explicit** - Waivers require TTY + challenge, create audit trail
6. **Approvals are binding** - Phase 8 approval binds to specific artifact versions
7. **Plans are linted** - Malformed plans block execution
8. **Receipts are authoritative** - Not state.json status, not agent narrative
9. **Human can audit** - `ralph audit` provides quick verification
10. **Machines can query** - `ralph robot` provides structured JSON API

## Open Questions

1. Should proofs be committed to git? (Audit trail vs. repo noise)
2. How long should loops be retained before archival?
3. Should there be a `ralph clean` to remove old loop directories?
4. What's the right TTY challenge complexity? (Balance security vs. annoyance)
5. Should `ralph lint plan` have auto-fix capability?
