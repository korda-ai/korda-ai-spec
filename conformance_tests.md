# Korda AI Conformance Tests
### The Deterministic Autonomous Agent Standard
*Grammar Specification v0.1.2 — Appendix C*

A Bravo Media Group development.
contact@bravomedia.uk  
https://github.com/korda-ai/korda-ai-spec

---

## Overview

A conformant Korda AI runtime must pass all tests in this document. Tests are expressed as input/expected pairs.

| Result Type | Meaning |
|---|---|
| `PARSE_OK` | Input is valid — parser must accept it |
| `PARSE_ERROR(code)` | Input is invalid — parser must reject it with that exact error code |
| `WARNING(code)` | Input is valid but parser must emit a warning with that code |

Conformance is binary. A runtime either passes the full suite or it does not. There is no partial conformance.

---

## C.1 Routing Tests

These tests verify that verbs route to the correct domain. This is normative — the routing order is fixed.

```
delete /tmp/file.txt              → PARSE_OK  (routes to HTTP, not filesystem)
remove /tmp/file.txt              → PARSE_OK  (routes to File System, not HTTP)
delete https://api.example.com/x  → PARSE_OK  (routes to HTTP)
remove /var/log/app.log           → PARSE_OK  (routes to File System)
```

---

## C.2 Structural Impossibilities

These are the three hard constraints of the Korda AI standard. A parser that accepts any of these forms is non-conformant.

### Constraint 1 — `exec` without `timeout:`
```
exec "cmd"
    → PARSE_ERROR(parse:shell:001)

exec "cmd" timeout:
    → PARSE_ERROR(parse:shell:002)

exec "cmd" timeout:0s
    → PARSE_ERROR(parse:shell:004)
```

### Constraint 2 — `delete-rows` without `where:`
```
delete-rows db:pg "users"
    → PARSE_ERROR(parse:db:004)

delete-rows db:pg "users" where:""
    → PARSE_ERROR(parse:db:005)

update db:pg "users" set:"x=1"
    → PARSE_ERROR(parse:db:003)
```

### Constraint 3 — Elevated verbs in restricted contexts
```
exec "cmd" timeout:5s  (inside with-permission read-only:)
    → PARSE_ERROR(parse:perm:002)

spawn agent:w  (inside with-permission read-only:)
    → PARSE_ERROR(parse:perm:002)

with-permission elevated:  (inside with-permission read-only:)
    → PARSE_ERROR(parse:perm:003)

exec "cmd" timeout:5s  (inside with-permission standard:)
    → PARSE_ERROR(parse:perm:002)
```

---

## C.3 Symbol Table Tests

These tests verify correct plan-scoped symbol table behavior.

```
-- Same label in two different plans: PARSE_OK
plan plan-a: step-x: fetch https://api.example.com/a end
plan plan-b: step-x: fetch https://api.example.com/b end
    → PARSE_OK

-- Duplicate label in same plan: PARSE_ERROR
plan plan-c:
  step-x: fetch https://api.example.com/a
  step-x: fetch https://api.example.com/b
end
    → PARSE_ERROR(parse:plan:002)

-- critique with unknown label: PARSE_ERROR
plan test-plan:
  real-step: fetch https://api.example.com/data
  bad-critique: critique step:nonexistent "Is this correct?"
end
    → PARSE_ERROR(parse:reason:003)

-- critique with known label: PARSE_OK
plan test-plan:
  fetch-step: fetch https://api.example.com/data
  check: critique step:fetch-step "Did this return expected data?"
end
    → PARSE_OK

-- duplicate hypothesis in same plan: PARSE_ERROR
plan hyp-plan:
  hypothesize h1 "First hypothesis"
  hypothesize h1 "Second hypothesis"
end
    → PARSE_ERROR(parse:reason:005)

-- unique hypotheses: PARSE_OK
hypothesize h1 "First"
hypothesize h2 "Second"
    → PARSE_OK
```

---

## C.4 Retry Counting Tests

`retry:N` means N additional attempts after the first. Total attempts = N+1. N must be ≥ 1.

```
fetch https://api.example.com/x on-error:retry:3   → PARSE_OK  (4 total attempts)
fetch https://api.example.com/x on-error:retry:1   → PARSE_OK  (2 total attempts)
fetch https://api.example.com/x on-error:retry:0   → PARSE_ERROR(parse:error:003)
fetch https://api.example.com/x on-error:retry:-1  → PARSE_ERROR(parse:error:003)
fetch https://api.example.com/x on-error:retry:    → PARSE_ERROR(parse:error:006)
```

---

## C.5 Reasoning Tests

```
think ""
    → PARSE_ERROR(parse:reason:001)

think "valid prompt"
    → PARSE_OK

critique "no step label"
    → PARSE_ERROR(parse:reason:002)

critique step:known-step "valid prompt"  (where known-step exists in scope)
    → PARSE_OK

hypothesize "missing label"
    → PARSE_ERROR(parse:reason:004)

hypothesize valid-label "text"
    → PARSE_OK

think "prompt" confidence:105
    → PARSE_ERROR(parse:reason:007)

think "prompt" confidence:-1
    → PARSE_ERROR(parse:reason:007)

think "prompt" confidence:100
    → PARSE_OK

think "prompt" confidence:0
    → PARSE_OK

think "prompt" confidence:0.0
    → PARSE_OK

think "prompt" confidence:1.0
    → PARSE_OK

think "prompt" confidence:1.1
    → PARSE_ERROR(parse:reason:007)

rank-options using:"criteria"
    → PARSE_ERROR(parse:reason:006)

rank-options over:{var:options} using:"criteria"
    → PARSE_OK
```

---

## C.6 Variable Scope Tests

```
{loop.item} outside loop body
    → PARSE_ERROR(parse:plan:005)

{error.message} outside error handler
    → PARSE_ERROR(parse:error:005)

{result} as first step in pipeline
    → PARSE_ERROR(parse:pipeline:001)

{reason.confidence} before any reasoning step
    → PARSE_ERROR(parse:reason:008)
```

---

## C.7 Semantic Warning Tests

```
loop over:{var:items} while:true: fetch "{loop.item}" end
    → PARSE_OK + WARNING(parse:plan:009)

loop over:{var:items} while:"count < 10": fetch "{loop.item}" end
    → PARSE_OK  (no warning — condition is not constant-true)

loop over:{var:items}: fetch "{loop.item}" end
    → PARSE_OK  (no while clause — no warning)
```

---

## C.8 Permission Tests

```
fetch https://api.example.com/x  (in read-only context)
    → PARSE_OK

think "prompt"  (in read-only context)
    → PARSE_OK

exec "ls" timeout:5s  (in read-only context)
    → PARSE_ERROR(parse:perm:002)

delete https://api.example.com/x  (in read-only context)
    → PARSE_ERROR(parse:perm:002)

exec "ls" timeout:5s  (in elevated context)
    → PARSE_OK
```

---

## C.9 Error Message Conformance

Every parse error produced by a conformant runtime must contain all five fields. A bare error string without all fields is a conformance failure.

**Required fields in every parse error:**
```
code:    string matching pattern /^parse:[a-z]+:\d{3}$/
message: non-empty string
line:    integer >= 1
col:     integer >= 1
got:     string (may be "<EOF>" for end of input)
fix:     non-empty string
```

**Required fields in every runtime error:**
```
code:    string matching pattern /^runtime:[a-z]+:\d{3}$/
verb:    non-empty string
target:  string
message: non-empty string
fix:     non-empty string
```

**Example of a conformant parse error:**
```
[3:1] [parse:shell:001] exec requires timeout: — structural constraint
[got: exec "ls"] [Fix: add timeout:Ns]
```

---

## Error Code Reference

| Code | Trigger |
|---|---|
| `parse:shell:001` | `exec` without `timeout:` |
| `parse:shell:002` | `timeout:` without value |
| `parse:shell:004` | Zero duration (`0s`) |
| `parse:db:003` | `update` without `where:` |
| `parse:db:004` | `delete-rows` without `where:` |
| `parse:db:005` | Empty `where:` value |
| `parse:perm:002` | Elevated verb in lower permission context |
| `parse:perm:003` | Permission ceiling rule violation |
| `parse:plan:002` | Duplicate step label in same plan |
| `parse:plan:005` | `{loop.item}` outside loop body |
| `parse:plan:009` | Constant-true `while:` condition (warning) |
| `parse:reason:001` | Empty `think` prompt |
| `parse:reason:002` | `critique` without `step:` |
| `parse:reason:003` | `critique` referencing unknown step label |
| `parse:reason:004` | `hypothesize` without label |
| `parse:reason:005` | Duplicate hypothesis label in scope |
| `parse:reason:006` | `rank-options` without `over:` |
| `parse:reason:007` | `confidence:` out of range |
| `parse:reason:008` | `{reason.confidence}` before any reasoning step |
| `parse:error:003` | `retry:N` where N < 1 |
| `parse:error:005` | `{error.*}` outside error handler |
| `parse:error:006` | `retry:` missing N |
| `parse:pipeline:001` | `{result}` in first pipeline step |

---

*Korda AI Grammar Specification v0.1.2*
*https://github.com/korda-ai/korda-ai-spec*
*contact@bravomedia.uk*  
*A Bravo Media Group development.*
