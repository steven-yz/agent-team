# Cross-thread Message Format

All agents use this format for `send_message_to_thread` messages. Deviating from this format breaks traceability.

## Standard envelope

```
[MSG_ID: {team}-{version}-{seq}]
FROM: {role}
TO: {role}
SUBJECT: {one-line summary}

{body -- structured with clear sections}

ACTION REQUIRED: {exactly what the receiver must do next}
```

## Fields

| Field | Format | Example |
|-------|--------|---------|
| MSG_ID | `{team}-{version}-{seq}` | `myapp-v1.2-001` |
| version | Matches spec version | `v1.2` |
| seq | Sequential 3-digit number | `001`, `002`, `003` |
| FROM | Agent role (manager/executor/verifier) | `executor` |
| TO | Target agent role | `verifier` |
| ACTION REQUIRED | Concrete next step. Never "review" or "check" -- say exactly what to do. | "Verify all criteria in specs/v1.2.md. Read that file directly." |

## Sequence numbering

```
Manager -> Dev:  {team}-{version}-001  (spec handoff)
Dev -> QA: {team}-{version}-002  (completion notice)
QA -> Dev: {team}-{version}-003  (FAIL report)
Dev -> QA: {team}-{version}-004  (fixes applied)
QA -> Manager:  {team}-{version}-005  (PASS notification)
```

Failure loop increments: `-003`, `-004`, `-005`, `-006`... until PASS.

## Message types

### Spec handoff (Manager -> Dev)

```
[MSG_ID: {team}-{version}-001]
FROM: manager
TO: executor
SUBJECT: Spec for {version}
## Specification
Full spec at specs/{version}.md
## Acceptance Criteria
- AC1: [specific, measurable criterion]
- AC2: [specific, measurable criterion]
ACTION REQUIRED: Implement per spec. Write output to the EXACT file path in the spec. When done, notify the QA.
```

### Completion notice (Dev -> QA)

```
[MSG_ID: {team}-{version}-002]
FROM: executor
TO: verifier
SUBJECT: Implementation complete for {version}
## Changes Made
- [file path]: [what was implemented]
## Spec Reference
Full spec at specs/{version}.md
ACTION REQUIRED: Verify all acceptance criteria in specs/{version}.md. Read that file directly.
```

### FAIL report (QA -> Dev)

```
[MSG_ID: {team}-{version}-003]
FROM: verifier
TO: executor
SUBJECT: FAIL -- {version} needs fixes
## Issues Found
- AC1: FAIL -- [expected vs actual]
## Passed
- AC2: PASS
ACTION REQUIRED: Fix all failing items above. Re-notify me with incremented MSG_ID.
```

### Fixes applied (Dev -> QA)

```
[MSG_ID: {team}-{version}-004]
FROM: executor
TO: verifier
SUBJECT: Fixes applied for {version}
## Fixes Applied
- AC1: [what was fixed and how]
ACTION REQUIRED: Re-verify all criteria in specs/{version}.md.
```

### PASS notification (QA -> Manager)

```
[MSG_ID: {team}-{version}-005]
FROM: verifier
TO: manager
SUBJECT: PASS -- {version} verified
## Verification Results
- AC1: PASS
- AC2: PASS
All {N} criteria passed.
ACTION REQUIRED: None. Version complete. Notify user.
```
