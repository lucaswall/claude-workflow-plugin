# Priority Assessment Guide

Assess priority independently for each issue. Priority is NOT determined by tag alone.

## Impact x Likelihood Matrix

| | High Likelihood | Medium Likelihood | Low Likelihood |
|---|---|---|---|
| **High Impact** | Critical | Critical | High |
| **Medium Impact** | High | Medium | Medium |
| **Low Impact** | Medium | Low | Low |

## Impact Factors

- Data loss or corruption → High
- Security breach or data exposure → High
- Service outage/crash (uncaught exception) → High
- User-facing errors or broken flows → Medium
- Performance degradation (e.g., slow responses, UI jank) → Medium
- Developer inconvenience → Low
- Code maintainability → Low

## Likelihood Factors

- Happens on every request or app launch → High
- Happens under normal usage → High
- Happens on specific inputs → Medium
- Happens only under edge conditions (e.g., low memory, network failure, concurrent access) → Low
- Requires attacker/malicious input → Varies (High if exposed, Low if internal)

## Examples

- `[security]` missing auth on public endpoint → Critical (high impact + high likelihood)
- `[security]` missing auth on admin-only internal endpoint → Medium (high impact + low likelihood)
- `[security]` hardcoded secret in source/build artifacts → High (high impact + medium likelihood — depends on exposure)
- `[storage]` sensitive data in insecure storage → Critical (high impact + high likelihood)
- `[permission]` missing authorization check on protected resource → High (broken flow + every user)
- `[memory-leak]` on every request → Critical (happens on every request)
- `[memory-leak]` only on error path → High or Medium
- `[bug]` wrong date format in logs → Low (low impact)
- `[bug]` wrong calculation in user-facing data → Critical (high impact — data accuracy)
- `[edge-case]` crash on empty/missing data → High (first-time users or new accounts hit this)
