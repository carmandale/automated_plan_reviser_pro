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

> **Stop making enforcement depend on the agent following instructions. Put the enforcement into Ralph (phase gates + subcommands) so the agent can't "talk its way past" Phase 4/5/7/10.**

## Hard Rules

### Direct Tool Calls Don't Count Unless Imported

**Primary rule:** Only receipts produced by Ralph (or imported into Ralph) count for gates.

Agents should use Ralph wrappers:
- ✅ `ralph review design` (Phase 4)
- ✅ `ralph oracle` (Phase 5)
- ✅ `ralph review plan` (Phase 7)
- ✅ `ralph review post` (Phase 10)

**Shell enforcement (ergonomic guardrail):** Ralph prepends a `.ralph/bin/` directory to `PATH` with shim scripts named `rp-cli`, `apr`, and `codex` that warn and redirect to wrapper commands. This is a guardrail, not a security boundary.

**Import flow for external evidence:**
```bash
ralph import-proof phase5 --from .apr/rounds/.../round_7.md
```

Import performs:
1. Hashes the provided artifact
2. Tool-backed verification where possible (Phase 5 queries APR)
3. Writes a Ralph-style receipt marked `imported: true`

**Rationale:** This makes enforcement depend on Ralph's gate checks, not on brittle "don't do X" rules. If you didn't get a Ralph receipt (directly or via import), you can't advance.

### Phase Completion Requires `ralph advance`

Every phase transition requires explicit CLI confirmation:

```bash
ralph advance <phase>  # The ONLY way to complete a phase
```

Add to each phase in the skill: *"Complete this phase by running `ralph advance <phase>`"*

This applies to ALL phases, not just gated ones. Eliminates "narrative completion."

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
    "research": {"status": "complete"},
    "interview": {"status": "complete"},
    "design": {"status": "complete"},
    "expert_review": {
      "status": "complete",
      "receipt": "proofs/phase4-1736789012-abc123.receipt.json"
    },
    "oracle": {
      "status": "running",
      "pid": 12345,
      "run_id": "apr-ralph-oracle-round-1",
      "started_at": "2026-01-13T12:35:00Z",
      "expected_nonce": "ralph-2f8a9c"
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

**Rationale:** Deterministic rules eliminate agent wiggle room. Reliability > optimization.

### CLI-Only Phase Advancement

Phase transitions happen ONLY through Ralph CLI:

```bash
ralph advance <phase>  # Only way to mark a phase complete
```

- `advance` refuses unless required proof/waiver exists and **verifies**
- `advance` updates cached status in `state.json`
- Agent saying "Phase 4 done" is meaningless - Ralph must verify

### Phase Gates

| Gate | Requires |
|------|----------|
| Before Phase 6 | Phase 4 receipt verified |
| Before Phase 6 (if Phase 5 enabled) | Phase 5 receipt verified (tool-backed) |
| Before Phase 8 | Phase 7 receipt verified |
| Before Phase 9 (`ralph yolo`) | Phase 8 approval + Phase 7 receipt (+ Phase 5 if applicable) |
| Before Phase 10 completes | Codex review receipt |

## Per-Loop Directory Structure

Each loop gets its own directory to prevent cross-loop confusion and receipt reuse:

```
.ralph/
├── loops/
│   └── <loop_id>/
│       ├── state.json
│       ├── proofs/
│       │   ├── phase4-<ts>-<id>.out
│       │   ├── phase4-<ts>-<id>.receipt.json
│       │   └── ...
│       ├── waivers/
│       │   └── phase5-<ts>.json
│       └── plan.md
├── current                    # Pointer file containing current loop_id
├── bin/                       # PATH shims
└── config.yaml                # Verification policies
```

All commands operate on the current loop unless `--loop <id>` is specified.

**Benefits:**
- Receipts are naturally scoped to their loop
- Single-use waivers are trivial to enforce
- Audit becomes cleaner
- No accidental cross-loop receipt reuse

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

### Pre-Committed Nonce Challenge

Nonces are **pre-committed** before tool invocation to make fabrication harder:

1. When starting a phase, Ralph:
   - Generates nonce
   - Writes it to `state.json` as `expected_nonce`
   - Only then invokes the tool

2. Verification requires:
   - `receipt.nonce == state.expected_nonce`
   - Nonce appears in transcript/output
   - For Phase 5: nonce appears in APR output file

**Rationale:** This upgrades nonce from "self-consistency marker" to "workflow binding marker." An agent can't just fabricate a receipt—they'd also need to coordinate expected_nonce in state.json before the tool was supposed to run.

### Substance Checks (Anti-Fabrication)

`ralph advance` verifies receipts have real content:

| Phase | Minimum Bytes | Required Markers |
|-------|---------------|------------------|
| Phase 4 (rp-cli) | 2048 | `chat_name` or `"mode": "plan"` |
| Phase 5 (APR) | 1024 | APR output file exists, nonce present |
| Phase 7 (rp-cli) | 2048 | `chat_name` or `"mode": "plan"` |
| Phase 10 (codex) | 512 | Recognizable review header/format |

### Tool-Backed Verification (Phase 5)

Phase 5 verification queries APR's session registry, not just local files:

```bash
# During ralph advance phase5 (and ralph yolo preflight):
apr robot status                    # Confirm APR is available
apr robot history                   # Confirm session slug exists and completed
# Verify output file matches APR's reported path and contains nonce
```

**If APR can't be queried:** Phase 5 is BLOCKED, not soft-pass.

**Rationale:** Agents can fabricate local files, but they can't fabricate APR's internal session history.

### Data-Driven Verification Rules

Verification thresholds are defined in config, not scattered in code:

```yaml
# .ralph/config.yaml
verification:
  phase4:
    min_bytes: 2048
    required_patterns:
      - "chat_name"
      - '"mode": "plan"'
  phase5:
    min_bytes: 1024
    requires_tool_backed_verify: true
    required_patterns:
      - "RALPH_NONCE:"
  phase7:
    min_bytes: 2048
    required_patterns:
      - "chat_name"
      - '"mode": "plan"'
  phase10:
    min_bytes: 512
    required_patterns:
      - "review"
```

`ralph verify --explain` prints exactly which rule failed.

### Filesystem Hardening

After Ralph creates proof artifacts:
```bash
chmod 444 *.receipt.json *.out
```
Raises friction for agent editing proofs. Not impossible to bypass, but stops common failure modes.

### Auto-Registration

When Ralph runs a tool wrapper, it:
1. Generates proof artifacts (out + receipt.json)
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
ralph oracle --background  # Returns run_id, sets status="running", pre-commits nonce
ralph wait <run_id>        # Blocks until complete, writes proof, sets status="complete"
```

**State transitions:**
1. `--background` → status becomes `"running"` with `pid`, `run_id`, `started_at`, `expected_nonce`
2. `ralph wait` → blocks until tool completes, then status becomes `"complete"` with `receipt`
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

### APR Robot Mode for Oracle

Phase 5 uses APR's proven reliability layer:

```bash
# Ralph internally runs:
apr robot validate 1 -w ralph-oracle  # Fast fail (pre-flight)
apr robot run 1 -w ralph-oracle --wait  # Blocking execution

# Captures from APR response:
# - slug for session tracking
# - output_file path
# - log_file path
```

## Waiver System

### TTY-Gated Overrides

**All override actions require interactive TTY:**
- `ralph waive <phase>`
- `ralph yolo --ignore-plan-lint`
- Phase 8 approvals
- Any future `--force` bypass flags

**Rules:**
1. If stdin is not a TTY → **hard fail** (exit non-zero)
2. If TTY is present → require deliberate action:
   - Retype the waiver reason
   - Type a one-time challenge: `TYPE "WAIVE PHASE5 <random>"`

**Rationale:** Most agent runners execute non-interactively. TTY-gating crushes the lazy/automatic bypass path.

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

## Robot Mode (JSON API)

Ralph provides a machine-friendly JSON API mirroring APR's proven pattern:

```bash
ralph robot status           # Environment and loop status
ralph robot gates            # Gate status (what's blocking)
ralph robot run-phase <N>    # Execute a phase
ralph robot verify-all       # Verify all receipts
ralph robot yolo-preflight   # Pre-flight check only (no execution)
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
- `tty_required` - Override requires interactive TTY
- `tool_unavailable` - External tool (APR, rp-cli) not available

**Rationale:** Agents can't "interpret" gates—they must consume structured output. This eliminates natural-language compliance issues.

## Plan Document

### PROOFS Block (Lint, Not Enforcement)

The plan's `## PROOFS` section is a human-readable mirror, not source of truth:

```markdown
## PROOFS
- phase4: proofs/phase4-1736789012-abc123.receipt.json
- phase5: proofs/phase5-1736789500-def456.receipt.json
- phase7: proofs/phase7-1736790000-ghi789.receipt.json
```

`ralph validate-plan` checks that plan mirrors state.json (nice UX), but gates enforce against verified receipts directly.

## Command Reference

### Core Commands

```bash
ralph init <feature-name>      # Initialize new ralph loop
ralph status                   # Show current state (recomputes from receipts)
ralph advance <phase>          # Mark phase complete (verifies receipt first)
ralph run-phase <phase>        # Run tool phase + advance (phases 4/5/7/10)
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
ralph oracle --background      # Returns run_id, pre-commits nonce
ralph wait <run_id>            # Block until complete
ralph retry <phase>            # Recover from stale/failed run
```

### Verification Commands

```bash
ralph verify <receipt>         # Verify single receipt
ralph verify-all               # Verify all receipts for current loop
ralph verify --explain         # Show which verification rule failed
ralph validate-plan            # Check plan mirrors state (lint)
ralph audit                    # Human-friendly table of all proofs
ralph show-proof <phase>       # Render human summary on demand
```

### Import Commands

```bash
ralph import-proof <phase> --from <path>  # Import external evidence
```

### Waiver Commands (TTY Required)

```bash
ralph waive <phase> --reason "..."   # Skip phase (requires TTY + challenge)
ralph waivers                        # List active waivers
```

### Robot Mode (JSON API)

```bash
ralph robot status             # {"ok": true, "data": {...}}
ralph robot gates              # Gate status as JSON
ralph robot run-phase <N>      # Execute phase, return JSON
ralph robot verify-all         # Verify all receipts
ralph robot yolo-preflight     # Pre-flight only
ralph robot help               # API documentation
```

## `ralph yolo` Preflight

Before execution, `ralph yolo` prints exactly what it verified:

```
Preflight Check:
✓ Phase 4 receipt: proofs/phase4-....receipt.json (verified)
✓ Phase 5 receipt: proofs/phase5-....receipt.json (verified, tool-backed)
✓ Phase 7 receipt: proofs/phase7-....receipt.json (verified)
✓ Plan/state coherence: validate-plan passed
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

## Implementation Phases

### Phase 1: Core Infrastructure
- [ ] Per-loop `.ralph/loops/<loop_id>/` directory structure
- [ ] `state.json` as cache, receipts as truth
- [ ] Receipt schema + generation
- [ ] Hash verification
- [ ] Pre-committed nonces

### Phase 2: Tool Wrappers
- [ ] `ralph review design` (rp-cli + proof)
- [ ] `ralph oracle` (APR robot mode + proof + tool-backed verification)
- [ ] `ralph review plan` (rp-cli + proof)
- [ ] `ralph review post` (codex + proof)

### Phase 3: Phase Gates
- [ ] Gate checking logic with receipt verification
- [ ] `ralph advance` command (verifies before advancing)
- [ ] Tool-backed verification for Phase 5
- [ ] Integration with `ralph yolo`

### Phase 4: Verification & Audit
- [ ] `ralph verify` / `ralph verify-all`
- [ ] `ralph verify --explain` (data-driven rules)
- [ ] `ralph audit` table output
- [ ] `ralph show-proof` (on-demand human summary)
- [ ] `ralph validate-plan` (lint)

### Phase 5: Waiver System
- [ ] TTY detection and challenge
- [ ] `ralph waive` with confirmation
- [ ] Waiver receipt generation
- [ ] Gate integration (proof OR waiver)

### Phase 6: Robot Mode
- [ ] `ralph robot status/gates/verify-all`
- [ ] `ralph robot run-phase`
- [ ] `ralph robot yolo-preflight`
- [ ] Consistent JSON envelope

### Phase 7: Recovery & Import
- [ ] Stale run detection
- [ ] `ralph retry` command
- [ ] `ralph import-proof` command

## Success Criteria

1. **Agents actually call tools** - Receipts exist with valid transcripts
2. **Receipts are verifiable** - SHA256 hashes match, substance checks pass, tool-backed verification for Phase 5
3. **Phases are gated** - `ralph yolo` fails without verified receipts
4. **Skips are explicit** - Waivers require TTY + challenge, create audit trail
5. **Receipts are authoritative** - Not state.json status, not agent narrative
6. **Human can audit** - `ralph audit` provides quick verification
7. **Machines can query** - `ralph robot` provides structured JSON API

## Open Questions

1. Should proofs be committed to git? (Audit trail vs. repo noise)
2. How long should loops be retained before archival?
3. Should there be a `ralph clean` to remove old loop directories?
4. What's the right TTY challenge complexity? (Balance security vs. annoyance)
