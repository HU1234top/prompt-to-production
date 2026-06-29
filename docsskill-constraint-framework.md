# From Prompt to Production: A Systematic Approach to Reliable LLM Agent Skills

> Based on real-world deployment experience with the [OpenClaw](https://github.com/openclaw/openclaw) framework.  
> Contributors: Author (system design), Boss (architecture review), EPC Engineer (engineering feedback), AI Agent (editorial integration)  
> Date: 2026-06-18

---

## 1. Problem Definition

### 1.1 Core Conclusion

**Pure prompt-based constraints cannot achieve 99%+ reliable execution of multi-step LLM Agent skills.**  
This is a universal challenge across all Agent frameworks, not specific to OpenClaw.

The current skill loading mechanism in OpenClaw is **loosely coupled**: the system prompt only injects the skill's `name` + `description` + `path`. The full SKILL.md content relies on the model actively calling `read`.

### 1.2 Three Types of Execution Deviations Observed

| Deviation Type | Real Case | Root Cause |
|---|---|---|
| **Doesn't read the skill, invents its own flow** | User says "enable solar proposal", Agent skips SKILL.md and improvises | Reflex rules written in prompt, but model attention misses them |
| **Reads but skips steps** | Received billing PDF, skipped Step 2 (data lookup), jumped straight to calculation | Instruction priority drops as context grows |
| **Reads but over-interprets** | After execution failure, Agent "audits" SKILL.md and lists 7 modification suggestions; only 1 was actually problematic | Misattribution of failure by the model |

### 1.3 Three-Layer Failure Chain

```
Model doesn't read SKILL.md → Improvises flow (Deviation Type 1)
      ↓
Reads but attention drifts → Skips steps / skips pre-checks (Deviation Type 2)
      ↓
Context decay → Rules blur after long multi-turn conversations, free-styling begins (Deviation Type 3)
```

### 1.4 Root Cause Analysis

| Root Cause | Layer | Explanation |
|---|---|---|
| **Framework level** | Input | Skill loading is "summary injection"; full content depends on model主动 read |
| **Model level** | Attention | Autoregressive attention drift; instruction compliance decays over multi-turn conversations |
| **Engineering level** | Runtime | No runtime validation; model execution results lack hard constraint interception |

---

## 2. Improvement Framework (Priority Ordered)

### Mechanism Overview

| Proposal | Target Layer | Why It Works (One Sentence) | Why It's Not Sufficient Alone |
|---|---|---|---|
| P0 | Input (prompt wording) | Increases SKILL.md rule attention weight | Decays when context dilutes |
| P1 | Input (behavioral examples) | Pattern matching has shorter computational path than abstract rules | Examples consume tokens, squeeze other instructions |
| P2 | Output (generation constraint) | JSON schema is more controllable than open-ended text | Only constrains output format, not execution logic |
| P3 | Trigger (entry point) | Bypasses model semantic matching, removes one failure point | Doesn't solve execution-phase problems |
| P4 | Runtime (code) | Deterministic code, doesn't depend on model behavior | Higher maintenance cost |
| P5 | Context (isolation) | Eliminates context decay root cause | Expensive data passing and user intervention |

---

### P0: Enforced Mandatory Read

**Problem**: Model doesn't read SKILL.md and improvises flow (Deviation Type 1)

**Approach**: Add stronger constraint markers to existing reflex rules:

```markdown
## [CRITICAL] Skill Trigger Rules

When user message matches these keywords, you MUST read the full SKILL.md 
BEFORE executing any other operation:

- "solar proposal" → read skills/solar-proposal/SKILL.md
- "consumption rate" → read skills/consumption-rate/SKILL.md

**Executing skill operations without reading SKILL.md is a critical error.**
```

**Cost**: Minimal (modify existing prompt wording)  
**Effect**: Reduces probability of "completely unread" cases. Doesn't solve "read but doesn't follow".  
**Rollback**: Remove `[CRITICAL]` marker to restore.

### P1: Few-shot Examples

**Problem**: Low model compliance with skill rules

**Approach**: Add 2-3 correct execution examples and 1-2 counter-examples before the steps section:

```markdown
## Execution Examples

### ✅ Correct
User: "Enable solar proposal"
Agent: read SKILL.md → Step 1 confirm data → Step 2 calculate → ...
Key: Read first, execute in order, report after each step

### ❌ Incorrect  
User: "Received billing PDF"
Error: Skipped Step 2, jumped straight to calculation
Cause: Did not follow SKILL.md step sequence
```

**Key Guidelines**:
- Keep total under 1KB (~2-3 correct + 1-2 counter-examples)
- Examples must come from real execution logs, not fabricated
- Add time annotations (e.g., `← Jun 2026 actual case`)
- Examples may expire with model updates; review monthly

**Cost**: Low (write + maintain examples)  
**Effect**: Models show higher compliance with concrete examples than abstract rules (established LLM research finding)  
**Side Effect**: Examples consume tokens and compete with P0 instructions for attention space  
**Rollback**: Delete example section to restore.

### P2: Structured Output + Schema Validation

**Problem**: Reliability and verifiability of single-step execution results

**Approach**: Require fixed-format JSON output after each step:

```markdown
### Step 2 Completion Criteria
After execution, output this JSON (write to /tmp/step2_result.json):

{
  "step": 2,
  "status": "success | failed",
  "data": { ... },
  "next_step": 3
}

If status is failed, STOP and report error. Do NOT proceed to next step.
```

**Sanity Checks**: Schema pass doesn't guarantee semantic correctness. Critical numerical steps need range validation:
- Consumption rate should be 0-100%, exceeding marks suspicious
- Installed capacity should match project card (±5% tolerance)

**Cost**: Medium (design schema + sanity rules per step)  
**Effect**: More reliable than relying on model self-discipline; catches obvious execution anomalies  
**Rollback**: Remove schema requirements to restore.

### P3: Slash Command Explicit Trigger

**Problem**: Uncertainty in model semantic matching

**Approach**: Set `user-invocable: true` in SKILL.md frontmatter:

```yaml
---
name: solar-proposal
description: Full solar investment proposal pipeline
user-invocable: true
---
```

**Effect**: User types `/solar-proposal` → 100% trigger rate, no semantic matching needed. Works with P0: slash trigger → forced read → step-by-step execution.

**Cost**: Minimal technically (one config line)  
**Effect**: Eliminates trigger-phase randomness  
**Rollback**: Remove `user-invocable: true` to restore.

### P4: Critical Step Checkpoint

**Problem**: Code-level step-skip prevention

**Approach**: Add deterministic validation between critical steps:

```python
def run_step_3():
    if not os.path.exists('/tmp/step2_result.json'):
        raise RuntimeError("Step 2 not completed, cannot execute Step 3")
    result = json.load(open('/tmp/step2_result.json'))
    if result['status'] != 'success':
        raise RuntimeError(f"Step 2 failed: {result.get('error')}")
    # Execute Step 3
    ...
```

**Apply Only To**:
- Database writes (irreversible)
- External message sends (unrevocable)
- Steps that critically depend on precise prior output

**Cost**: Medium  
**Effect**: Hard constraint that works even when model attention drifts. The only constraint that doesn't depend on model behavior.  
**Side Effect**: With P2, model bears JSON output + checkpoint error handling overhead per step.  
**Rollback**: Remove checkpoint code to restore.

### P5: Sub-agent Isolation (Enable Only For Specific Scenarios)

**Problem**: Context decay causing rule ambiguity

**Approach**: Spawn independent sub-tasks per step:

```
Main: Detects user request → spawn Step1 sub-task
Step1: Only sees Step1 instructions → completes → returns result
Main: Receives Step1 result → spawn Step2 sub-task  
Step2: Only sees Step2 instructions + Step1 result → completes → returns result
```

**Trade-offs**:

| Cost | Measured Impact |
|---|---|
| Token consumption doubles | Each step initializes fresh context |
| Slower execution | Sub-task init 10-15 seconds |
| Data passing field loss | JSON serialization errors (e.g., 541.03→541.3) |
| No mid-execution user intervention | Sub-task can't see user saying "wait" |

**Enable Only When** (all conditions must be met):
- Zero user interaction between steps (e.g., batch data processing)
- High structured I/O per step
- Extreme reliability requirements (financial, compliance)
- Client has clear acceptance criteria and is paying

**Cost**: High  
**Effect**: Most thorough isolation, but not suitable for interactive scenarios

### P6: Framework-level Forced Injection (Long-term Direction)

**Problem**: Eliminate the root cause of "model doesn't read skills"

**Ideal**: When a skill is triggered, the framework injects full SKILL.md content directly into current context.

**Current Status**: OpenClaw doesn't support slash command handlers or preprocessing hooks. This is a framework-level improvement requiring official support.

**Recommendation**: File feature requests, track roadmap. Use P0 as short-term substitute.

---

## 3. Strategy Summary

### 3.1 Existing Constraint Mechanism: .learnings Promotion

An **error-driven rule strengthening mechanism** already exists:

```
Execution deviation occurs → Record in .learnings/
→ Recurs 3+ times → Promote to SOUL.md reflex (hard constraint)
```

This mechanism is slow (requires multiple recurrences) but **adaptive** — rules come from real failures. Complements P0-P4 without conflict.

### 3.2 Phased Implementation Strategy

**Phase 1 (This Week): P0 + P3 — Low-risk incremental changes**

| Action | Nature | Risk |
|---|---|---|
| P0: Add [CRITICAL] markers to reflex rules | Wording change | Minimal |
| P3: Add `user-invocable: true` to core skills | Config addition | Minimal |

Validation: Run solar proposal skill end-to-end after changes.

**Phase 2 (Within 2 Weeks): P1 — Add few-shot examples**

| Action | Nature | Risk |
|---|---|---|
| P1: Extract failure cases, write into SKILL.md | Append examples | Low |

Validation: Run 2-3 different projects, observe skip/over-interpretation reduction.

**Phase 3 (After Stabilization, Within 1 Month): P2 + P4 — Runtime validation**

| Action | Nature | Risk |
|---|---|---|
| P2: Design JSON schema per step | Output constraint | Medium |
| P4: Add checkpoint for database writes | Code-level skip prevention | Medium |

Validation: Test each addition incrementally, rollback immediately if issues arise.

**Phase 4 (Long-term): P5 + P6 — Enable on demand**

P5 only when all enable conditions are met. P6 tracks OpenClaw roadmap continuously.

### 3.3 Client Expectation Management

**Core Principle: Don't deliver Agent skills as "100% automated black box".**

- ❌ Client expects: "Say one sentence and everything works perfectly"
- ✅ Set expectation: "AI automates most work, I'll confirm key checkpoints"

---

## 4. Comparison Matrix

| Proposal | Solves | Tech Cost | Rollout Cost | Expected Effect | Rollback Difficulty |
|---|---|---|---|---|---|
| P0 Enforced Read | Not reading skill | Minimal | None | Medium | Minimal |
| P1 Few-shot | Low compliance | Low | Low | Medium-high | Low |
| P2 Structured Output | Single-step reliability | Medium | Low | High | Medium |
| P3 Slash Command | Trigger uncertainty | Minimal | Medium | High | Minimal |
| P4 Checkpoint | Step skipping | Medium | Low | Very High | Medium |
| P5 Sub-agent | Context decay | High | Low | Very High | High |
| P6 Framework Injection | Root cause | High | None | Fundamental | N/A |

---

## 5. Combination Guidelines

### 5.1 Layer Relationships: Serial Defense

```
P3 Slash Command (reliable trigger)
  → P0 Forced Read (ensures content is loaded)
    → P1 Few-shot (compliance after reading)
      → P2 Structured Output (controllable results)
        → P4 Checkpoint (skip prevention fallback)
          → P5 Sub-agent (no context decay)
```

Each layer only defends its own failure point. P0 can't prevent "read but examples don't apply", but P4 checkpoint catches it.

### 5.2 Combination Side Effects: Limited but Exist

**Side Effect 1**: P1 Few-shot competes with P0 for attention pool.  
Mitigation: Keep examples ≤ 1KB.

**Side Effect 2**: P2 Schema + P4 Checkpoint adds non-business overhead.  
Mitigation: P4 only for high-risk steps (database writes).

**Side Effect 3**: P5 + P4/P2 increases maintenance cost.  
Mitigation: Only P4 for critical steps; P5 only in specific scenarios.

### 5.3 Residual Risks

| Risk | Description | Mitigation |
|---|---|---|
| Few-shot examples expire | Static examples may not match evolving business | Time annotations, monthly review |
| Schema passes but semantics wrong | JSON format correct but calculation incorrect | Range validation for critical values |
| Context decay (without P5) | Rules blur in long conversations | Re-read key SKILL.md sections at each step start |
| Model version changes | Attention behavior may change after model swap | Re-test full pipeline after model change |

---

## 6. Action Plan

| Phase | Priority | Action | Suggested Timeline | Acceptance Criteria |
|---|---|---|---|---|
| 1 | P0 | Strengthen reflex rules with [CRITICAL] markers | This week | Rules contain [CRITICAL] and "executing without reading is critical error" |
| 1 | P3 | Add `user-invocable: true` to core skills | This week | Slash command `/skill-name` works |
| 2 | P1 | Extract failure cases, write few-shot examples | 2 weeks | SKILL.md has 2 correct + 1 counter-example, ≤1KB total, time-stamped |
| 3 | P2 | Design per-step JSON schema + sanity validation | 1 month | Each critical step has JSON schema + value range checks |
| 3 | P4 | Write checkpoint validation for database writes | 1 month | Write steps have code-level skip prevention |
| 4 | P5 | Evaluate sub-agent isolation needs | On demand | Only implement when all conditions are met |

---

## Appendix: Discussion History

| Version | Date | Key Changes |
|---|---|---|
| v1.0 | 2026-06-18 | Initial draft, three-party discussion |
| v1.1 | 2026-06-18 | Author revision: P0 deployment notes, removed effect percentages, added .learnings reference |
| v2.0 | 2026-06-18 | Editorial integration: mechanism overview table, combination side-effect analysis, rollback difficulty column, corrected P6 slash command description, four-phase strategy, residual risk catalog |

---

*Document Version: v2.0-final | 2026-06-18 | Four-party discussion finalized*
