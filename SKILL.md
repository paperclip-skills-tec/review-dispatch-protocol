---
name: review-dispatch-protocol
description: "Dispatch protocol for Code Review Lead after scope triage classifies a task as Standard Gate or Targeted Validation. Use this skill immediately after code-review-scope-triage to create specialist child issues with correct context, assign them to the right reviewers, set the blocker chain, and post the dispatch comment. Required before creating any child review issues — do not create any specialist issues without it."
---

# Review Dispatch Protocol

Use this skill **immediately after `code-review-scope-triage` classifies a task**. It handles everything needed to turn a classification decision into a correctly structured set of specialist child issues: extracting context, building issue descriptions, assigning the right reviewer, setting up the wake-on-completion chain, and posting the dispatch comment.

The Code Review Lead is an orchestrator. Your job here is precision setup — the specialists do the reading; you do the routing. A child issue with missing context wastes a specialist's heartbeat. A child issue with the wrong assignee wastes everyone's time.

---

## Step 1 — Extract context from the parent review task

Read the parent issue carefully before creating any child issues. Collect exactly these fields:

| Field | Where to find it |
|---|---|
| **PR / diff link** | Issue description or triggering comment. Look for GitHub PR URL, branch name, or commit range. |
| **Changed files summary** | Mentioned in the description, or infer from the PR title/scope notes. Do not open the diff yourself. |
| **Source issue / ticket** | The Paperclip issue the PR implements (e.g. `TEC-NNN`). Linked in description or PR title. |
| **Scope boundaries** | Any explicit constraints from the requester ("only review the auth module", "ignore the migration"). |
| **Urgency / priority** | The priority on the parent issue. Child issues inherit this priority. |
| **Review classification** | From the scope triage: STANDARD GATE (all 5) or TARGETED VALIDATION (subset). |
| **Omitted dimensions** | If TARGETED VALIDATION, the list of omitted dimensions and their rationale. |

If the PR link is absent, comment on the parent issue asking for it before creating any child issues. Do not proceed without a reviewable artifact.

---

## Step 2 — Pre-flight guard rail

Before creating any issue:

- **Standard Gate:** dispatch all 5 dimensions or none. If you cannot identify the correct assignee for any dimension (agent unreachable or role vacant), do not create a partial set — create a blocked task for Chief of Staff with the gap identified.
- **Targeted Validation:** dispatch the approved subset (minimum 2, always including Correctness). Every omitted dimension needs its rationale recorded in the dispatch comment.
- **Never create a 6th or non-standard dimension.** If a requester asks for something outside the 5 standard dimensions, escalate to Chief of Staff.

---

## Step 3 — Create specialist child issues

Create all child issues in a single pass (parallel API calls where the platform supports it). Each issue must have:

- `parentId`: the parent review issue ID
- `assigneeAgentId`: the specialist agent ID from the routing table below
- `priority`: inherited from the parent
- `goalId`: inherited from the parent (if set)
- `title`: use the standard title pattern
- `description`: use the template for that dimension

### Specialist routing table

| Dimension | Agent name | Agent ID |
|---|---|---|
| Correctness | Correctness Reviewer | `c38bcd9c-b96f-49b0-93e2-cb37b5db4b8d` |
| Code Quality | Quality Reviewer | `e9dd08e2-061a-4340-ad9d-c8864320c2dd` |
| Architecture | Architecture Reviewer | `107da8a6-3dc7-44c3-b4a1-7ec1d7ea64fc` |
| Test Coverage | Test Coverage Reviewer | `96fe0620-3407-459d-9e47-189444807b88` |
| Security | Security Reviewer | `4498f3d4-f711-413b-ac23-2999f1e160e4` |

### Standard title pattern

```
[Review] <Dimension>: <parent issue identifier> — <parent title>
```

**Examples:**
- `[Review] Correctness: TEC-1042 — Add user export endpoint`
- `[Review] Security: TEC-1042 — Add user export endpoint`

### Issue description template (fill in all placeholders)

```markdown
## Review Brief

**Dimension:** [Correctness | Code Quality | Architecture | Test Coverage | Security]
**Parent review task:** [TEC-XXXX title and link]
**Source issue:** [TEC-YYYY — the implementation ticket, or "Not specified"]
**PR / diff:** [GitHub PR URL or branch name]
**Changed files scope:** [Summary of changed areas, e.g. "routes/auth.js, services/userService.js, migrations/003_add_export_flag.sql"]
**Scope constraints:** [Any explicit limits from the requester, or "None — full diff in scope"]

## What to review

[1–3 sentences scoping the review to this dimension. Be specific about what to look for. Examples below.]
```

**Dimension-specific "What to review" guidance:**

- **Correctness** — Verify that the implementation matches the acceptance criteria stated in [source issue]. Check edge cases: null inputs, empty collections, boundary values, concurrent access if applicable. Look for silent failures: swallowed exceptions, missing `await`, unhandled rejection paths.

- **Code Quality** — Assess naming clarity, single-responsibility adherence, duplication, and dead code. Check that env vars are used for config, no unused imports, and patterns are consistent with the existing codebase conventions.

- **Architecture** — Assess module boundaries and coupling. Does this change introduce circular dependencies or bleed concerns across layers? Check that the data flow is coherent with the existing design. Flag any new abstractions that add complexity without clear justification.

- **Test Coverage** — Verify coverage of the happy path and at least two meaningful edge cases per changed function. Confirm tests would actually fail if the implementation were wrong (anti-tautological). Check for any `it.skip` or `xit` without a linked issue. Integration tests must hit a real database, not mocks.

- **Security** — Run against the OWASP Top 10 checkpoints in the `sdlc-process` skill. Specifically: parameterized SQL (no string interpolation), input validation at API boundaries, no secrets in code or responses, auth middleware on protected routes, `npm audit` clean.

---

## Step 4 — Set the blocker chain

After all child issues are created, collect their IDs. Then:

```
PATCH /api/issues/{parentReviewIssueId}
{
  "blockedByIssueIds": ["<child-1-id>", "<child-2-id>", "<child-3-id>", "<child-4-id>", "<child-5-id>"],
  "status": "blocked"
}
```

This does two things:
1. Marks the parent review issue as blocked (so it doesn't appear as active work for the Code Review Lead).
2. Triggers an `issue_blockers_resolved` wake when all specialists mark their issues done, bringing the Code Review Lead back for synthesis.

The `issue_children_completed` event also fires when all direct children reach a terminal state — you get two wake paths, which is intentional for resilience.

**For Targeted Validation:** only include the dispatched child issues in `blockedByIssueIds`. Omitted dimensions do not block the synthesis.

---

## Step 5 — Post the dispatch comment

Post this comment on the parent review issue **after** all child issues are created and the blocker chain is set. Fill in every placeholder.

```markdown
## Dispatch Complete

**Classification:** [STANDARD GATE — 5 specialists | TARGETED VALIDATION — N specialists]
**Dispatched:** [ISO timestamp]

### Specialist assignments

| Dimension | Issue | Assignee | Status |
|---|---|---|---|
| Correctness | [TEC-XXXX](/TEC/issues/TEC-XXXX) | Correctness Reviewer | dispatched |
| Code Quality | [TEC-YYYY](/TEC/issues/TEC-YYYY) | Quality Reviewer | dispatched |
| Architecture | [TEC-ZZZZ](/TEC/issues/TEC-ZZZZ) | Architecture Reviewer | dispatched |
| Test Coverage | [TEC-AAAA](/TEC/issues/TEC-AAAA) | Test Coverage Reviewer | dispatched |
| Security | [TEC-BBBB](/TEC/issues/TEC-BBBB) | Security Reviewer | dispatched |

[For Targeted Validation, add an "Omissions" section:]
### Omitted dimensions

| Dimension | Rationale |
|---|---|
| [dimension] | [reason documented during scope triage] |

### Next action

This issue is blocked pending specialist findings. Code Review Lead resumes when all dispatched issues reach `done`.
Expected turnaround: [1–2 heartbeats per specialist, depending on diff size].
```

---

## Error conditions

| Condition | Action |
|---|---|
| PR link absent | Comment asking for the PR URL. Do not create any child issues. Do not post a dispatch comment. |
| Specialist agent unreachable (409 on create) | Abort the full dispatch. Post a comment naming the failed dimension and that no issues were created. Create a task for Chief of Staff to investigate. |
| Parent issue has no `goalId` | Omit `goalId` from child issues — do not fabricate one. Note the absence in the dispatch comment. |
| Partial dispatch already exists (some child issues exist) | Do not create duplicates. Audit existing children, identify missing dimensions, complete the set, then update `blockedByIssueIds` with all child IDs. Note the remediation in the dispatch comment. |

---

## Quick reference — API calls

```bash
# Create one child issue (repeat per dimension)
POST /api/companies/{companyId}/issues
{
  "title": "[Review] Correctness: TEC-XXXX — ...",
  "description": "...",
  "parentId": "{reviewIssueId}",
  "assigneeAgentId": "c38bcd9c-b96f-49b0-93e2-cb37b5db4b8d",
  "priority": "{inheritedPriority}",
  "goalId": "{inheritedGoalId}"   // omit if parent has none
}

# Set blocker chain after all children created
PATCH /api/issues/{reviewIssueId}
{
  "blockedByIssueIds": ["{child1}", "{child2}", "{child3}", "{child4}", "{child5}"],
  "status": "blocked"
}

# Post dispatch comment
POST /api/issues/{reviewIssueId}/comments
{ "body": "..." }
```

All API calls must include `X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID`.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
