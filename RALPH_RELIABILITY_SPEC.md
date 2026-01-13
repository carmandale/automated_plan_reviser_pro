# Ralph Reliability Layer Specification

## Problem Statement

AI coding agents (Claude Code, Codex, etc.) when executing Ralph loops are:

1. **Not issuing requests properly** - Skipping or malforming Oracle/rp-cli calls
2. **Not waiting for responses** - Proceeding before tools complete
3. **Fabricating responses** - Making up fake "expert reviews" instead of using tools

This is NOT a network reliability problem. This is an **agent discipline problem**.

## Design Principle

> **Stop making enforcement depend on the agent following instructions. Put the enforcement into Ralph (phase gates + subcommands) so the agent can't "talk its way past" Phase 4/5/7/10.**

## Hard Rules

### Never Call Tools Directly

**Agents must NEVER call these tools directly:**
- ❌ `rp-cli` 
- ❌ `apr` / `oracle`
- ❌ `codex`

**Only use Ralph wrappers:**
- ✅ `ralph review design` (Phase 4)
- ✅ `ralph oracle` (Phase 5)
- ✅ `ralph review plan` (Phase 7)
- ✅ `ralph review post` (Phase 10)

**Enforcement:** If Ralph detects direct tool usage without corresponding receipts, it prints a loud warning. Best-effort detection via comparing proof artifacts to expected receipts.

**Rationale:** If you didn't run the wrapper, you can't advance. This makes "I did it" narratively meaningless.

### Phase Completion Requires `ralph advance`

Every phase transition requires explicit CLI confirmation:

```bash
ralph advance <phase>  # The ONLY way to complete a phase
```

Add to each phase in the skill: *"Complete this phase by running `ralph advance <phase>`"*

This applies to ALL phases, not just gated ones. Eliminates "narrative completion."

## Core Architecture

### Source of Truth: state.json

`state.json` is the **single source of truth** for phase completion. Not the agent's narrative, not markdown parsing.

**Status values:** `pending` → `running` → `complete` (or `waived`)

- **pending**: Not started
- **running**: Tool invoked but not yet complete (background mode)
- **complete**: Receipt exists and verified
- **waived**: Skipped with explicit waiver

**Key invariant:** `running ≠ complete`. A phase in "running" state cannot advance until the tool finishes and produces a verified receipt.

```json
{
  "feature": "Console Panel",
  "started": "2026-01-13T10:00:00Z",
  "phase5_required": true,
  "phases": {
    "research": {"status": "complete"},
    "interview": {"status": "complete"},
    "design": {"status": "complete"},
    "expert_review": {
      "status": "complete",
      "receipt": ".ralph/proofs/phase4-1736789012-abc123.receipt.json"
    },
    "oracle": {
      "status": "running",
      "run_id": "apr-ralph-oracle-round-1",
      "started_at": "2026-01-13T12:35:00Z"
    },
    "create_plan": {"status": "pending"},
    "plan_review": {"status": "pending"},
    "finalize": {"status": "pending"},
    "execute": {"status": "pending"},
    "post_review": {"status": "pending"}
  }
}
```

### CLI-Only Phase Advancement

Phase transitions happen ONLY through Ralph CLI:

```bash
ralph advance <phase>  # Only way to mark a phase complete
```

- `advance` refuses unless required proof/waiver exists and verifies
- `advance` writes to state.json
- Agent saying "Phase 4 done" is meaningless - Ralph must confirm

### Phase Gates

| Gate | Requires |
|------|----------|
| Before Phase 6 | Phase 4 receipt verified |
| Before Phase 6 (if Phase 5 enabled) | Phase 5 receipt verified |
| Before Phase 8 | Phase 7 receipt verified |
| Before Phase 9 (`ralph yolo`) | Phase 8 approval + Phase 7 receipt (+ Phase 5 if applicable) |
| Before Phase 10 completes | Codex review receipt |

## Proof System

### Three-File Proof Artifacts

Every tool invocation produces three files:

```
.ralph/proofs/
├── phase4-<ts>-<id>.out           # Raw stdout/stderr transcript
├── phase4-<ts>-<id>.receipt.json  # Authoritative machine-verifiable receipt
└── phase4-<ts>-<id>.proof.md      # Human-readable summary (derived)
```

### Receipt Schema (JSON - Primary)

```json
{
  "schema_version": "1.0.0",
  "phase": "phase4",
  "ts": 1736789012,
  "run_id": "abc123",
  "command": ["ralph", "review", "design"],
  "exit_code": 0,
  "transcript_path": "phase4-1736789012-abc123.out",
  "transcript_bytes": 4523,
  "transcript_sha256": "a1b2c3d4e5f6...",
  "tool": {
    "kind": "rp-cli",
    "identifiers": {
      "chat_name": "Design Review: Console Panel",
      "mode": "plan"
    }
  },
  "duration_ms": 45230,
  "created_mtime": 1736789057
}
```

### Substance Checks (Anti-Fabrication)

`ralph advance` verifies receipts have real content, not empty/trivial transcripts:

| Phase | Minimum Bytes | Required Markers |
|-------|---------------|------------------|
| Phase 4 (rp-cli) | 2048 | `chat_name` or `"mode": "plan"` in output |
| Phase 5 (APR) | 1024 | APR output file exists and non-empty |
| Phase 7 (rp-cli) | 2048 | `chat_name` or `"mode": "plan"` in output |
| Phase 10 (codex) | 512 | Recognizable review header/format |

**Rationale:** Pragmatic tripwires that kill common "tool call theater" patterns. Not cryptographic proof, but sufficient friction against fabrication.

### Filesystem Hardening

After Ralph creates proof artifacts:
```bash
chmod 444 *.receipt.json *.out *.proof.md
```
Raises friction for agent editing proofs. Not impossible to bypass, but stops common failure modes.

### Auto-Registration

When Ralph runs a tool wrapper, it:
1. Generates proof artifacts (out + receipt.json + proof.md)
2. Updates state.json with receipt path
3. Makes files read-only

No agent action required - proofs are automatically registered.

## Tool Wrapper Commands

### Blocking by Default

All tool wrappers block until completion:

| Command | Phase | Behavior |
|---------|-------|----------|
| `ralph review design` | 4 | Runs rp-cli, blocks, writes proof |
| `ralph oracle` | 5 | Runs APR robot mode, blocks, writes proof |
| `ralph review plan` | 7 | Runs rp-cli, blocks, writes proof |
| `ralph review post` | 10 | Runs codex review, blocks, writes proof |

### Background Mode (Explicit)

If background mode is allowed:
```bash
ralph oracle --background  # Returns run_id, sets status="running"
ralph wait <run_id>        # Blocks until complete, writes proof, sets status="complete"
```

**State transitions:**
1. `--background` → status becomes `"running"` with `run_id` and `started_at`
2. `ralph wait` → blocks until tool completes, then status becomes `"complete"` with `receipt`
3. `ralph advance` → refuses if status is `"running"` (must wait first)

**Key invariant:** A tool call isn't "done" until the receipt exists. "Started" ≠ "done".

### APR Robot Mode for Oracle

Phase 5 uses APR's proven reliability layer:

```bash
# Ralph internally runs:
apr robot validate 1 -w ralph-oracle  # Fast fail
apr robot run 1 -w ralph-oracle --wait  # Blocking execution

# Captures from APR response:
# - slug for session tracking
# - output_file path
# - log_file path
```

## Waiver System

### Scoped, Single-Use Waivers

Waivers are tied to specific loop + phase + run_id:

```bash
ralph waive phase5 --reason "Single-line typo fix, no architecture impact"
```

**Requirements:**
- Minimum reason length (non-empty)
- User confirmation required
- Scoped to current loop only
- Single-use (can't reuse for next loop)

### Waiver Receipt

```json
{
  "phase": "phase5",
  "ts": 1736789012,
  "run_id": "current-loop-id",
  "reason": "Single-line typo fix, no architecture impact",
  "confirmed_by": "user",
  "confirmed_at": "2026-01-13T12:30:12Z"
}
```

Gates accept: **Receipt present OR Waiver present** (not silent skip)

## Plan Document

### PROOFS Block (Lint, Not Enforcement)

The plan's `## PROOFS` section is a human-readable mirror, not source of truth:

```markdown
## PROOFS
- phase4: .ralph/proofs/phase4-1736789012-abc123.receipt.json
- phase5: .ralph/proofs/phase5-1736789500-def456.receipt.json
- phase7: .ralph/proofs/phase7-1736790000-ghi789.receipt.json
```

`ralph validate-plan` checks that plan mirrors state.json (nice UX), but gates enforce against state.json + receipts directly.

## Command Reference

### Core Commands

```bash
ralph init <feature-name>      # Initialize new ralph loop
ralph status                   # Show current state and next required action
ralph advance <phase>          # Mark phase complete (only way)
ralph gates                    # Show gate status (what's blocking)
ralph yolo                     # Execute (preflight verifies all gates)
```

### Review Commands (Tool Wrappers)

```bash
ralph review design            # Phase 4: rp-cli design review (blocking)
ralph oracle                   # Phase 5: APR/Oracle review (blocking)
ralph review plan              # Phase 7: rp-cli plan review (blocking)
ralph review post              # Phase 10: codex review (blocking)
```

### Background & Wait

```bash
ralph oracle --background      # Returns run_id
ralph wait <run_id>            # Block until complete
```

### Verification Commands

```bash
ralph verify <receipt>         # Verify single receipt (hash check)
ralph verify-all               # Verify all receipts for current loop
ralph validate-plan            # Check plan mirrors state (lint)
ralph audit                    # Human-friendly table of all proofs
```

### Waiver Commands

```bash
ralph waive <phase> --reason "..."   # Skip phase with justification
ralph waivers                        # List active waivers
```

## `ralph yolo` Preflight

Before execution, `ralph yolo` prints exactly what it verified:

```
Preflight Check:
✓ Phase 4 receipt: .ralph/proofs/phase4-....receipt.json (verified)
✓ Phase 5 receipt: .ralph/proofs/phase5-....receipt.json (verified)
✓ Phase 7 receipt: .ralph/proofs/phase7-....receipt.json (verified)
✓ Phase 8 approval: confirmed

All gates satisfied. Proceeding with execution...
```

If any missing: `BLOCKED BY GATE X` and exits nonzero.

## `ralph audit` Output

Quick human sanity check:

```
┌─────────┬──────────┬─────────────────────────────────┬─────────────┬────────┬──────┬───────┐
│ Phase   │ Status   │ Receipt                         │ Timestamp   │ Tool   │ Exit │ Bytes │
├─────────┼──────────┼─────────────────────────────────┼─────────────┼────────┼──────┼───────┤
│ phase4  │ complete │ phase4-1736789012-abc123.json   │ 12:30:12    │ rp-cli │ 0    │ 4523  │
│ phase5  │ complete │ phase5-1736789500-def456.json   │ 12:38:20    │ apr    │ 0    │ 8291  │
│ phase7  │ complete │ phase7-1736790000-ghi789.json   │ 12:46:40    │ rp-cli │ 0    │ 3892  │
│ phase10 │ pending  │ -                               │ -           │ -      │ -    │ -     │
└─────────┴──────────┴─────────────────────────────────┴─────────────┴────────┴──────┴───────┘
```

## File Structure

```
.ralph/
├── state.json                    # Source of truth
├── proofs/
│   ├── phase4-<ts>-<id>.out
│   ├── phase4-<ts>-<id>.receipt.json
│   ├── phase4-<ts>-<id>.proof.md
│   ├── phase5-<ts>-<id>.out
│   ├── phase5-<ts>-<id>.receipt.json
│   ├── phase5-<ts>-<id>.proof.md
│   └── ...
├── waivers/
│   └── phase5-<ts>.json
└── plans/
    └── current-plan.md
```

## Implementation Phases

### Phase 1: Core Infrastructure
- [ ] `.ralph/` directory structure
- [ ] `state.json` management
- [ ] Receipt schema + generation
- [ ] Hash verification

### Phase 2: Tool Wrappers
- [ ] `ralph review design` (rp-cli + proof)
- [ ] `ralph oracle` (APR robot mode + proof)
- [ ] `ralph review plan` (rp-cli + proof)
- [ ] `ralph review post` (codex + proof)

### Phase 3: Phase Gates
- [ ] Gate checking logic
- [ ] `ralph advance` command
- [ ] Integration with `ralph yolo`

### Phase 4: Verification & Audit
- [ ] `ralph verify` / `ralph verify-all`
- [ ] `ralph audit` table output
- [ ] `ralph validate-plan` (lint)

### Phase 5: Waiver System
- [ ] `ralph waive` with confirmation
- [ ] Waiver receipt generation
- [ ] Gate integration (proof OR waiver)

## Success Criteria

1. **Agents actually call tools** - Receipts exist with valid transcripts
2. **Receipts are verifiable** - SHA256 hashes match, files non-empty
3. **Phases are gated** - `ralph yolo` fails without required receipts
4. **Skips are explicit** - Waivers create audit trail
5. **state.json is authoritative** - Not agent narrative, not markdown parsing
6. **Human can audit** - `ralph audit` provides quick verification

## Open Questions

1. Should proofs be committed to git? (Audit trail vs. repo noise)
2. How long should proofs be retained?
3. Should there be a `ralph clean` to remove old proof artifacts?
4. Should `ralph audit` support `--json` for machine consumption?
