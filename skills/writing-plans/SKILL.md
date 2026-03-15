---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- (User preferences for plan location override this default)

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## Testing Requirements

Plans that include test tasks MUST follow spec-driven testing. Workers who receive vague testing instructions ("write tests") produce tests that verify nothing.

### Before Writing Test Code: Extract Requirements

Every plan chunk that includes tests MUST have a requirements extraction step before any test code is written. The implementer reads the spec and produces a structured mapping of requirements to test cases.

For each requirement, define:
- **What the spec requires** (concrete, verifiable statement)
- **Observable assertion** (what state to check — file exists, response contains field, record in database)
- **Setup** (what preconditions must exist — seed data, running services, funded wallets)

If a requirement can't be stated as an observable assertion, it's not testable — flag it and move on.

### Observable Assertions

Tests verify **what happened**, not **how it happened**. Every test assertion must check observable state.

Valid assertions:
| Category | Example |
|----------|---------|
| Response content | `assert "signals" in data and len(data["signals"]) > 0` |
| Data shape | `assert "asset" in signal and "action" in signal` |
| State change | Record exists in database after operation |
| Error specificity | `assert resp.status_code == 422` (one code, not a set) |

**Invalid — these are fallback chains disguised as assertions:**
| Anti-pattern | Why it's wrong |
|-------------|---------------|
| `assert status in (200, 502)` | Tolerates failure — test passes without exercising behavior |
| `assert status in (200, 429)` | Masks resource exhaustion — test never verifies data shape |
| `pytest.skip("no data")` without a seed fixture | Missing test prerequisite, not a valid skip condition |

A test that tolerates an error code **is not testing the behavior it claims to test**. If the service might return 502, the plan must include a step that ensures the service is healthy before running tests — not an assertion that accepts 502.

### Fixture and Seed Data

If tests require external state (database records, funded wallets, running services), the plan MUST include explicit setup steps that create that state. Tests must not skip or degrade when state is missing — they must fail, because missing state means the test environment is broken.

**Example:** If wizard tests need WizardProfile records, the plan includes a step that writes seed records to the registry before the test task runs.

### Test Isolation

- Fixtures that create test state must be **function-scoped** (no `scope="session"` or `scope="module"`) unless the state is truly immutable (e.g., a URL string from an env var)
- Tests must not depend on execution order
- Tests must create the state they need and verify the state they produce

## Remember
- Exact file paths always
- Complete code in plan (not "add validation")
- Exact commands with expected output
- Reference relevant skills with @ syntax
- DRY, YAGNI, TDD, frequent commits

## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent (see plan-document-reviewer-prompt.md) with precisely crafted review context — never your session history. This keeps the reviewer focused on the plan, not your thought process.
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.

**Review loop guidance:**
- Same agent that wrote the plan fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect

## Execution Handoff

After saving the plan:

**"Plan complete and saved to `docs/superpowers/plans/<filename>.md`. Ready to execute?"**

**Execution path depends on harness capabilities:**

**If harness has subagents (Claude Code, etc.):**
- **REQUIRED:** Use superpowers:subagent-driven-development
- Do NOT offer a choice - subagent-driven is the standard approach
- Fresh subagent per task + two-stage review

**If harness does NOT have subagents:**
- Execute plan in current session using superpowers:executing-plans
- Batch execution with checkpoints for review
