# Comprehensive Code Review Checklist

Reference for plan-review-implementation skill.

## Priority Tiers

| Tier | Severity | Examples |
|------|----------|----------|
| **CRITICAL** | Immediate fix required | Security vulnerabilities, data corruption, crashes |
| **HIGH** | Fix before merge | Logic errors, race conditions, auth issues |
| **MEDIUM** | Should fix | Edge cases, type safety, error handling gaps |
| **LOW** | Nice to have | Code style, documentation, minor improvements |

## Security Checks (OWASP-Based)

### Input Validation
- [ ] All user inputs validated server-side (never trust client)
- [ ] Allowlist validation preferred over blocklist
- [ ] Command injection prevention (avoid shell execution with user input)
- [ ] Path traversal prevention (`../` sequences blocked)
- [ ] XSS prevention (context-appropriate encoding: HTML, JS, CSS, URL)
- [ ] SSRF prevention (server-side requests use allowlisted URLs, no user-controlled URLs in fetch)
- [ ] File upload validation (content type, size, extension)
- [ ] Input length limits enforced
- [ ] Special characters handled appropriately

### Authentication & Session
- [ ] Session tokens cryptographically random (>=128 bits)
- [ ] Session invalidation on logout
- [ ] Session timeout for inactivity
- [ ] Cookie flags set (HttpOnly, Secure, SameSite)
- [ ] Refresh tokens rotated on use
- [ ] Constant-time comparison for sensitive values (tokens, API keys) — use `crypto.timingSafeEqual()`, not `===`

### Authorization
- [ ] Access controls enforced server-side
- [ ] Default deny policy
- [ ] IDOR prevention (validate user owns resource)
- [ ] Admin functions protected
- [ ] API endpoints match expected access level

### Secrets & Credentials
- [ ] No hardcoded secrets, API keys, passwords
- [ ] Secrets loaded from env vars or secret manager
- [ ] No secrets in git history
- [ ] Sensitive data not logged
- [ ] Error messages don't leak internal info

### Security Headers
- [ ] Content-Security-Policy (CSP) configured
- [ ] X-Content-Type-Options: nosniff
- [ ] X-Frame-Options or CSP frame-ancestors
- [ ] Strict-Transport-Security (HSTS)
- [ ] Referrer-Policy configured

## Logic & Correctness

### Common Bug Patterns
- [ ] Off-by-one errors in loops/indices
- [ ] Null/undefined handling (especially from external data)
- [ ] Empty array/object edge cases
- [ ] Floating point comparison issues
- [ ] String encoding issues (UTF-8)
- [ ] Timezone handling in dates
- [ ] Boolean logic errors (De Morgan's law violations)
- [ ] Assignment vs comparison (= vs ==)

### Boundary Conditions
- [ ] Empty inputs handled
- [ ] Single-element collections
- [ ] Maximum size inputs
- [ ] Negative numbers where unexpected
- [ ] Zero values
- [ ] Unicode edge cases (emojis, RTL, combining chars)
- [ ] Very long strings
- [ ] Deeply nested objects

### State Management
- [ ] Race conditions in shared state
- [ ] State mutations in wrong order
- [ ] Missing state cleanup
- [ ] Stale state references (closures)
- [ ] Concurrent modification issues

## Async & Concurrency

### Promise/Async Handling
- [ ] All promises have error handlers (.catch or try/catch)
- [ ] Async functions called with await or .then/.catch
- [ ] Promise.all failures handled appropriately
- [ ] No fire-and-forget async (unless intentional)
- [ ] Errors propagated correctly up the chain

### Race Conditions
- [ ] Shared mutable state protected
- [ ] Check-then-act patterns atomicized
- [ ] Concurrent writes to same resource
- [ ] Event ordering assumptions valid

### Deadlocks & Hangs
- [ ] External API calls have timeouts
- [ ] Circuit breakers for unreliable services
- [ ] No await in infinite loops without yield

## Resource Management

### Memory Leaks
- [ ] Event listeners removed when done (.off, removeListener)
- [ ] Intervals cleared (clearInterval)
- [ ] Caches have eviction/size limits
- [ ] Streams destroyed on error/completion
- [ ] Large objects not held unnecessarily in closures
- [ ] Collections don't grow unbounded

### Resource Leaks
- [ ] HTTP connections closed on error
- [ ] External subscriptions cancelled
- [ ] Temporary files cleaned up

## Error Handling

### Error Propagation
- [ ] Errors not swallowed silently
- [ ] Empty catch blocks justified
- [ ] Errors logged with context
- [ ] Original error preserved when wrapping
- [ ] Appropriate error types used

### Error Recovery
- [ ] Retry logic for transient failures
- [ ] Backoff strategies prevent thundering herd
- [ ] Fallback behavior for non-critical features
- [ ] Partial failures handled gracefully

### Error Information
- [ ] Error messages are actionable
- [ ] No sensitive data in error messages
- [ ] Errors logged for debugging

## Type Safety

### TypeScript/Type Checks
- [ ] No unsafe `any` casts
- [ ] Type guards for narrowing
- [ ] Nullable types handled (null, undefined)
- [ ] Union types exhaustively matched
- [ ] External data validated/parsed
- [ ] Type assertions justified and correct

## Test Quality (When Tests Are Changed)

### Test Validity
- [ ] Tests have meaningful assertions
- [ ] Not just "doesn't throw" tests
- [ ] Assertions match test description
- [ ] Mocks don't hide real bugs
- [ ] Edge cases covered
- [ ] Error paths tested

### Test Independence
- [ ] Tests don't depend on execution order
- [ ] Shared state cleaned up
- [ ] No flaky timing dependencies

### Test Data
- [ ] No real customer/user data
- [ ] No production credentials
- [ ] Test data clearly fictional

## Project-Specific (CLAUDE.md)

Always check CLAUDE.md for project-specific rules including:
- Import conventions
- Error handling patterns
- Testing requirements
- Naming conventions
- Security requirements (auth middleware)
- Any other project-specific standards

## Claude API Integration (When Changed Files Touch AI Code)

When reviewing changes to Claude API integration code (`src/lib/claude.ts`, `src/lib/chat-tools.ts`, API routes calling Claude), verify:

### Stop Reason Handling
- `stop_reason` checked for ALL values: `"tool_use"`, `"end_turn"`, `"max_tokens"`, `"refusal"` (safety refusal, Claude 4+), `"model_context_window_exceeded"` (context limit, Claude 4+)
- Unknown stop reasons handled gracefully (don't crash on new values)

### Tool Configuration
- `strict: true` on tool definitions for guaranteed schema conformance (runtime validation still needed for truncation/refusal)
- `tool_choice` + thinking compatibility: `{type: "tool"}` and `{type: "any"}` are INCOMPATIBLE with extended/adaptive thinking — only `"auto"` and `"none"` work
- `temperature` and `top_p` NOT used simultaneously (breaking change in Claude 4+)

### Prompt Caching
- Static content (tools, system prompt) marked with `cache_control: {type: "ephemeral"}` for cost reduction
- Cache invalidation triggers not introduced (changing tool definitions invalidates entire cache)

### Error Handling
- Claude API 429 uses `Retry-After` header for backoff
- Claude API 529 (overloaded) handled separately from 500
- Model snapshot IDs pinned in production (not aliases)

## AI-Generated Code Risks

All code in this project is AI-assisted. Apply extra scrutiny for these patterns:
- **Logic errors** (75% more common in AI code) — verify branching, loop bounds, comparisons
- **XSS vulnerabilities** (2.74x higher frequency) — check all dynamic content rendering
- **Code duplication** (frequent AI pattern) — similar code that should use shared abstractions
- **Security flaws** (~45% of AI code contains them) — treat all AI output as untrusted until reviewed
- **Missing input validation** — AI often skips server-side validation
- **Missing context** (AI may not understand business logic) — verify domain-specific constraints
- **Hallucinated APIs** (non-existent methods/libraries) — verify imports resolve to real exports
- **Hallucinated packages** — non-existent npm packages that may be claimed by attackers. Verify every `import` references a real package in `package.json`.
- **Outdated patterns** — AI may use deprecated APIs or old security practices from training data
- **Over-engineering** — unnecessary abstractions or error handling for impossible scenarios
