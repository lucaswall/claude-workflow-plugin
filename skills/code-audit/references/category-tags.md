# Category Tags Reference

## Audit Tags (validated during code audit)

| Tag | Description | OWASP |
|-----|-------------|-------|
| `[security]` | Injection, exposed secrets, missing auth | A01-A03, A07 |
| `[memory-leak]` | Unbounded growth, unclosed resources, retained refs | - |
| `[bug]` | Logic errors, data corruption, off-by-one | - |
| `[resource-leak]` | Connections, file handles, timers not cleaned up | - |
| `[async]` | Unhandled promises, race conditions, missing error propagation | - |
| `[timeout]` | Missing timeouts, potential hangs, no circuit breaker | - |
| `[shutdown]` | Graceful shutdown issues | - |
| `[edge-case]` | Unhandled scenarios, boundary conditions | - |
| `[convention]` | CLAUDE.md violations | - |
| `[type]` | Unsafe casts, missing guards, unvalidated external data | - |
| `[dependency]` | Vulnerable or outdated packages | A06 |
| `[rate-limit]` | API quota exhaustion risks | - |
| `[logging]` | Missing logs, wrong levels, log overflow, insufficient debug coverage | - |
| `[dead-code]` | Unused functions, unreachable code | - |
| `[duplicate]` | Repeated logic | - |
| `[test]` | Useless/duplicate tests, no assertions | - |
| `[practice]` | Anti-patterns | - |

**OWASP Top 10 (2021) Reference:**
- A01: Broken Access Control (auth bypass, IDOR, privilege escalation)
- A02: Cryptographic Failures (secrets exposure, weak crypto)
- A03: Injection (SQL, NoSQL, command, XSS)
- A06: Vulnerable Components (outdated dependencies)
- A07: Authentication Failures (weak sessions, missing auth)

## Non-Audit Tags (preserved without validation)

| Tag | Description |
|-----|-------------|
| `[feature]` | New functionality to add |
| `[improvement]` | Enhancement to existing functionality |
| `[enhancement]` | Similar to improvement |
| `[refactor]` | Code restructuring without behavior change |
| `[docs]` | Documentation updates |
| `[chore]` | Maintenance tasks |

Non-audit issues are preserved in Linear Backlog without validation.

## Linear Label Mapping

When creating Linear issues, map category tags to Linear labels:

| Category Tags | Linear Label |
|---------------|--------------|
| `[security]`, `[dependency]` | Security |
| `[bug]`, `[async]`, `[shutdown]`, `[edge-case]`, `[type]`, `[logging]` | Bug |
| `[memory-leak]`, `[resource-leak]`, `[timeout]`, `[rate-limit]` | Performance |
| `[convention]` | Convention |
| `[dead-code]`, `[duplicate]`, `[test]`, `[practice]`, `[docs]`, `[chore]` | Technical Debt |
| `[feature]` | Feature |
| `[improvement]`, `[enhancement]`, `[refactor]` | Improvement |

## Linear Priority Mapping

Map priority levels to Linear priority values:

| Priority Tag | Linear Priority |
|--------------|-----------------|
| `[critical]` | 1 (Urgent) |
| `[high]` | 2 (High) |
| `[medium]` | 3 (Medium) |
| `[low]` | 4 (Low) |
