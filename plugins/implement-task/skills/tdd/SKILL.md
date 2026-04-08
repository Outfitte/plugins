---
name: tdd
description: TDD methodology for all coding tasks. Use when implementing features, fixing bugs, writing functions, or any task that involves producing or modifying code.
user-invocable: false
---

Apply the RED-GREEN-REFACTOR cycle strictly. No production code exists before a failing test.

> **Language note:** The examples below use Go. Apply the same methodology in whatever language the project uses — the cycle, ordering, and extraction rules are language-agnostic.

## Test Naming

Name every test to make the intent self-documenting. A good pattern: `<WhatIsTested>_<Expectation>_<Given>` or `<WhatIsTested>Should<Expectation>When<Given>` — adapt to the project's existing convention.

Examples:
- `TestParseShouldFailWhenGivenNilInput`
- `TestCalculateTotalShouldReturnZeroWhenCartIsEmpty`
- `TestAuthShouldRejectWhenTokenIsExpired`

## Test Order

**Failure cases before happy path.** Within a cycle:
1. Write failure/error case tests first (file not found, invalid input, cancelled context, etc.)
2. Write the happy path last

This ordering guarantees the stub fails each test for the right behavioral reason before any logic is written.

## Stub Pattern

Start every new function/method with a stub so the very first test always fails:

```go
// Go example
func (p *Provider[T]) Save(_ context.Context, _ T) error {
    return errors.New("not implemented")
}
```

Keep the "not implemented" stub at the bottom through each intermediate implementation step. Only remove it when implementing the final happy-path test.

## RED Phase

1. Write one test that describes the next desired behavior
2. Run it — confirm it fails for the right reason (behavioral failure, not compile/syntax error)
3. If it fails due to a compile/syntax error, fix compilation without adding any logic

## GREEN Phase

1. Write the minimum production code to make the test pass
2. Do not implement more than what the current test requires
3. Run the test — confirm it passes
4. No other tests should break

## REFACTOR Phase

After every GREEN phase, scan the new production code for extraction candidates before moving to the next RED phase. Repeat the same scan at the end of the full TDD cycle.

### Identifying extraction candidates

Extract a block into a helper when it:
- Represents one coherent logical step with a clear name (e.g., "find and verify user", "save session", "retrieve and validate session")
- Spans multiple lines that could be read as a single sentence
- Is referenced by a comment that narrates what the block does — the comment title becomes the helper name

Do **not** extract:
- Single-line expressions
- Blocks with no clear single-responsibility label
- Code that would require passing many unrelated arguments

### Naming helpers

Use active-verb names that describe *what is done*, not *how*:
- `findAndVerifyUser(ctx, email, password)` — not `getUserWithCheck`
- `saveSession(ctx, userID, rawToken)` — not `sessionCreation`
- `retrieveSession(ctx, sessionID, rawRandom)` — not `getSessionHelper`

### Example (Go)

Before (post-GREEN, no extraction yet):
```go
func (s *AuthService) Login(ctx context.Context, email, password string) (string, error) {
    user, err := s.users.FindByEmail(ctx, email)
    if err != nil {
        return "", err
    }
    if !verifyPassword(user.PasswordHash, password) {
        return "", domain.ErrUnauthorized
    }
    rawToken := generateToken()
    session := domain.Session{UserID: user.ID, TokenHash: hash(rawToken)}
    if err := s.sessions.Save(ctx, session); err != nil {
        return "", err
    }
    return rawToken, nil
}
```

After (post-REFACTOR, helpers extracted):
```go
func (s *AuthService) Login(ctx context.Context, email, password string) (string, error) {
    user, err := s.findAndVerifyUser(ctx, email, password)
    if err != nil {
        return "", err
    }
    return s.saveSession(ctx, user.ID)
}
```

Candidates identified: "find user + verify password" (3 lines, one coherent step) → `findAndVerifyUser`; "create + persist session" (3 lines, one coherent step) → `saveSession`.

### Extraction steps

1. Identify every candidate block in the function just written
2. For each candidate: extract to a named helper method, passing only required arguments
3. Run all tests after each extraction — must stay green
4. Verify extracted helpers have 100% coverage independently
5. Coverage must not decrease after refactoring

## Error Assertions

Assert on **domain/sentinel errors** using the language's idiomatic equality check, never pin to error message strings:

```go
// Go — correct
require.ErrorIs(t, err, domain.ErrIO)

// Go — wrong (brittle, couples test to stdlib internals)
require.ErrorContains(t, err, "no such file or directory")
```

Wrap infrastructure errors at the boundary before returning so callers receive typed sentinels:

```go
// Go example
return fmt.Errorf("%w: %w", domain.ErrIO, err)
```

Add error sentinels at the point they are first needed — no pre-emptive catalogue.

## Context / Cancellation Conventions (Go)

- Name the parameter `ctx`, never blank it with `_`
- Check `ctx.Err()` before any lock or I/O at the start of the function
- In tests, use `t.Context()` instead of `context.Background()`
- For cancellation tests, derive from `t.Context()`: `ctx, cancel := context.WithCancel(t.Context())`

## Test Entity Values

Use explicit, readable values (e.g. `"42"`, `"alice@example.com"`) rather than zero/empty defaults — intent is clearer and failures are easier to diagnose.

## Coverage Rules

- Coverage must never decrease between phases
- After GREEN: coverage must increase relative to before RED
- After REFACTOR: coverage must remain stable or improve
- If coverage drops, add tests for the uncovered paths before proceeding

## End-of-Cycle Helper Review

After all RED-GREEN-REFACTOR cycles are complete and tests pass, do a final pass over all production code written in this session:

1. Re-read each function top to bottom
2. Flag any block of 3+ lines that could be expressed as a single sentence
3. Extract remaining candidates into named helpers
4. Re-run tests and confirm coverage is stable

## TDD Compliance Checks

Before writing any production code, confirm:
- [ ] A failing test exists
- [ ] The test fails for a behavioral reason, not a compile/syntax error
- [ ] The test name follows a clear `<what>_<expectation>_<condition>` pattern

Before closing a GREEN phase, confirm:
- [ ] All tests pass
- [ ] No unrelated tests were broken
- [ ] Implementation is minimal — no speculative code

Before closing a REFACTOR phase, confirm:
- [ ] Scanned every new function for multi-line blocks representing a coherent logical step
- [ ] Each candidate was either extracted into a named helper or explicitly ruled out (single-line or unclear responsibility)
- [ ] All tests still pass
- [ ] Coverage did not decrease
- [ ] Extracted helpers are independently tested and fully covered
