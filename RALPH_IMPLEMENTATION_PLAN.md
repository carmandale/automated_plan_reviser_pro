# Ralph Reliability Layer - Implementation Plan

> **Status:** Ready for implementation (APR Round 13 - CONVERGED)
> **Last Updated:** 2026-01-15

## Executive Summary

After 13 rounds of APR review, the Ralph Reliability Layer specification has **conceptually converged** on the right enforcement primitives. The core design addresses the three failure modes (skip / don't wait / fabricate) through:

1. **Signed receipts** - Anti-fabrication tripwire (HMAC-SHA256)
2. **Blocking tool wrappers** - Prevents "didn't wait" failures
3. **Input hash binding** - Invalidates stale reviews
4. **Phase 5 tool-backed verification** - APR robot mode introspection
5. **TTY-gated overrides** - Waivers/approvals require human presence

Round 13's explicit recommendation: **STOP spec iteration, START implementation + dogfooding.**

---

## Threat Model (Scope)

**In-scope:**
- Lazy agents that skip steps
- Impatient agents that don't wait for tools
- Accidental skipping due to context loss
- "Tool call theater" - narrating without executing

**Out-of-scope:**
- Adversarial agents with filesystem access forging artifacts
- Compromising Ralph binary itself
- Supply chain attacks on APR/Oracle/rp-cli

---

## Core Architecture

### Source of Truth

```
Receipts → Truth
state.json → Cache (recomputed from receipts on every status check)
```

### Directory Structure

```
.ralph/
├── loops/
│   └── <loop_id>/
│       ├── state.json
│       ├── research.md, interview.md, design.md, plan.md
│       ├── proofs/
│       │   ├── phase4-<ts>-<id>.out
│       │   ├── phase4-<ts>-<id>.receipt.json
│       │   └── ...
│       ├── approvals/
│       │   └── phase8-<ts>.json
│       └── waivers/
│           └── phase5-<ts>.json
├── current                    # Pointer to active loop_id
└── config.yaml
```

### Receipt Signing

```bash
# Setup (ralph init)
~/.local/share/ralph/secret.key  # chmod 600

# Every receipt:
sig = HMAC-SHA256(secret, canonical_json_without_sig)
```

---

## Phase Requirements

| Phase | Artifact | Verification |
|-------|----------|--------------|
| 1 Research | `research.md` | Lint: `## Files Inspected` with paths |
| 2 Interview | `interview.md` | Lint: 5 categories present |
| 3 Design | `design.md` | Lint: "Files to create/modify" section |
| 4 Expert Review | Receipt | Signed, hash-bound to design.md |
| 5 Oracle | Receipt | Tool-backed (APR robot), signed |
| 6 Create Plan | `plan.md` | `ralph lint plan` passes |
| 7 Plan Review | Receipt | Signed, hash-bound to plan.md |
| 8 Finalize | Approval receipt | TTY + challenge verified |
| 9 Execute | `ralph yolo` | Single choke point |
| 10 Post Review | Receipt | Signed, hash-bound to git HEAD |

---

## Round 13 Changes to Integrate

### Change 1: Simplify Phase 5 Policy

**Current:** Policy receipt + evidence tracking
**Target:** `phase5_required := true unless waiver_exists`

- DELETE: `phase5_policy.evidence`, policy receipt machinery
- KEEP: Simple boolean in state.json, waiver check

### Change 2: Restrict import-proof

**Current:** Import allowed for all phases with TTY fallback
**Target:** Import defaults to Phase 5 only

```bash
ralph import-proof phase5 --from <path>   # ✅ Allowed (tool-backed verify)
ralph import-proof phase4 --from <path>   # ❌ Disabled by default
                                          # Use --allow-unverifiable-import for break-glass
```

### Change 3: Phase 10 Input Binding

**Current:** Substance checks only (bytes, header)
**Target:** Bind to git HEAD

```json
{
  "inputs": {
    "git_head": "abc123...",
    "git_dirty": false
  }
}
```

Remediation: `BLOCKED: code changed after post review. Run: ralph review post`

### Change 4: Add --final Flag

**Current:** Separate wrapper → advance steps
**Target:** Atomic convenience flag

```bash
ralph review design --final   # Run wrapper → verify → advance (atomic)
ralph oracle --final          # Same pattern
ralph review plan --final     # Same pattern
ralph review post --final     # Same pattern
```

On hash mismatch: fail with single remediation line.

### Change 5: Consolidate Substance Checks

**Current:** Tool-format markers (`chat_name`, `"mode": "plan"`)
**Target:** Signed receipts + hashes + sizes

Remove from verification:
- `chat_name` marker requirement (Phase 4/7)
- `"mode": "plan"` marker requirement

Keep:
- `transcript_bytes >= threshold`
- `sha256(transcript) == receipt.transcript_sha256`
- Valid `sig`
- Phase 5: nonce-in-output (tool-backed verified)

---

## Implementation Phases

### Phase 1: Core Infrastructure (Week 1)

- [ ] Per-loop directory structure (`.ralph/loops/<loop_id>/`)
- [ ] `state.json` schema + CRUD operations
- [ ] Secret key generation (`ralph init`)
- [ ] Receipt schema + signing (HMAC-SHA256)
- [ ] Receipt verification pipeline

### Phase 2: Non-Tool Phase Validation (Week 1)

- [ ] `ralph advance phase1` - verify research.md + lint
- [ ] `ralph advance phase2` - verify interview.md + lint
- [ ] `ralph advance phase3` - verify design.md + lint
- [ ] `ralph advance phase6` - verify plan.md + `ralph lint plan`
- [ ] `ralph lint plan` implementation

### Phase 3: Tool Wrappers (Week 2)

- [ ] `ralph review design` - rp-cli wrapper + auto-context + receipt
- [ ] `ralph oracle` - APR wrapper + blocking + receipt
- [ ] `ralph review plan` - rp-cli wrapper + auto-context + receipt
- [ ] `ralph review post` - codex wrapper + git HEAD binding + receipt
- [ ] `--final` flag for all wrappers

### Phase 4: Phase Gates (Week 2)

- [ ] Input hash binding verification (Phase 4/5/7)
- [ ] Git HEAD binding verification (Phase 10)
- [ ] `ralph advance` gate checks
- [ ] Tool-backed verification for Phase 5 (APR robot status/history)
- [ ] `ralph yolo` preflight (single choke point)

### Phase 5: Override System (Week 3)

- [ ] TTY detection
- [ ] Challenge generation + verification
- [ ] `ralph waive <phase>` with scoped receipts
- [ ] `ralph approve` (Phase 8) with hash binding
- [ ] Agent mode detection (non-TTY → fail-closed)

### Phase 6: Import System (Week 3)

- [ ] `ralph import-proof phase5` - APR robot verify + adopt
- [ ] Phase 4/7/10 import disabled by default
- [ ] `--allow-unverifiable-import` escape hatch (TTY-gated)

### Phase 7: Robot Mode (Week 4)

- [ ] `ralph robot status`
- [ ] `ralph robot gates`
- [ ] `ralph robot verify-all`
- [ ] `ralph robot yolo-preflight`
- [ ] `ralph robot lint-plan`
- [ ] Consistent JSON envelope + error codes

### Phase 8: Polish (Week 4)

- [ ] `ralph audit` table output
- [ ] `ralph show-proof <phase>` on-demand summary
- [ ] `ralph status` human-friendly output
- [ ] Error messages with canonical remediation lines

---

## Key Commands

### Core Workflow

```bash
ralph init <feature>           # Start new loop
ralph status                   # Show state (recomputed from receipts)
ralph advance <phase>          # Complete phase (verify first)
ralph gates                    # Show what's blocking
ralph yolo                     # Execute (verifies ALL gates)
```

### Tool Wrappers

```bash
ralph review design [--final]  # Phase 4
ralph oracle [--final]         # Phase 5
ralph review plan [--final]    # Phase 7
ralph review post [--final]    # Phase 10
```

### Verification

```bash
ralph verify <receipt>         # Single receipt
ralph verify-all               # All receipts
ralph lint plan                # Plan validation
ralph audit                    # Human-readable table
```

### Overrides (TTY Required)

```bash
ralph waive <phase> --reason "..."
ralph approve
ralph import-proof phase5 --from <path>
```

---

## Success Criteria

1. **All phases artifact-gated** - No completion without verifiable work
2. **Receipts are verifiable** - Hash + signature + tool-backed where applicable
3. **Input binding works** - Stale reviews are rejected
4. **Phases are gated** - `ralph yolo` fails without verified receipts
5. **Skips are explicit** - Waivers require TTY + challenge
6. **Approvals are binding** - Phase 8 binds to specific versions
7. **Plans are linted** - Malformed plans block execution
8. **Receipts are authoritative** - Not state.json, not narrative
9. **Human can audit** - `ralph audit` provides verification
10. **Machines can query** - `ralph robot` provides JSON API

---

## APR Integration Reference

Ralph Phase 5 uses APR robot mode:

```bash
# Validate before run
apr robot validate <N> -w ralph-oracle

# Execute (returns immediately with slug/pid)
apr robot run <N> -w ralph-oracle

# Check completion
apr robot history -w ralph-oracle
apr robot status

# Response fields for receipt
{
  "slug": "apr-...",
  "output_file": ".apr/rounds/.../round_N.md",
  "log_file": ".apr/logs/oracle_....log",
  "status": "completed"
}
```

**Invariant:** Phase 5 receipt only written after APR reports `status: completed` and output file exists with nonce.

---

## Next Steps

1. Begin Phase 1 implementation (core infrastructure)
2. Create test fixtures for receipt verification
3. Implement `ralph init` with secret key setup
4. Build receipt signing/verification pipeline
5. Add non-tool phase validation (`ralph advance phase1-3,6`)

**Post-implementation:** Run another APR round constrained to: "What did agents still manage to fabricate or skip, despite gates?"
