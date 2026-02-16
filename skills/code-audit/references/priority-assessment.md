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
- Security breach potential → High
- Service outage/crash → High
- User-facing errors → Medium
- Performance degradation → Medium
- Developer inconvenience → Low
- Code maintainability → Low

## Likelihood Factors

- Happens on every request → High
- Happens under normal load → High
- Happens on specific inputs → Medium
- Happens only under edge conditions → Low
- Requires attacker/malicious input → Varies (High if exposed, Low if internal)

## Examples

- `[security]` missing auth on public endpoint → Critical (high impact + high likelihood)
- `[security]` missing auth on admin-only internal endpoint → Medium (high impact + low likelihood)
- `[memory-leak]` on every request → Critical
- `[memory-leak]` only on error path → High or Medium
- `[bug]` wrong date format in logs → Low (low impact)
- `[bug]` wrong calorie count in food log → Critical (high impact)
