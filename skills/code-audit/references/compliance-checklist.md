# Code Audit Compliance Checklist

Universal checks that apply to any project. Project-specific rules should be defined in CLAUDE.md.

## Security (OWASP-Based)

### Authorization (OWASP A01:2021)
- Public endpoints are intentional and documented
- Auth middleware applied to protected routes
- Role/permission checks where needed
- IDOR prevention (validate user owns resource)
- Horizontal/vertical privilege escalation blocked

### Secrets & Credentials (OWASP A02:2021)
- No hardcoded secrets, API keys, or passwords in code
- Secrets loaded from environment variables or secret managers
- No secrets in git history (check for accidental commits)
- Sensitive data not logged (passwords, tokens, PII)
- Debug/verbose modes don't expose secrets
- Error messages don't leak internal paths or stack traces

### Input Validation (OWASP A03:2021)
- User/API input sanitized before use
- SQL/NoSQL injection prevention (parameterized queries, ORM) - if using databases
- Command injection prevention (avoid shell execution with user input)
- Path traversal prevention (`../` sequences blocked)
- SSRF prevention — server-side requests use allowlisted URLs/domains, no user-controlled URLs passed directly to fetch/HTTP clients
- External API response validation (don't trust third-party data)
- File path/name validation when processing external files

### Authentication (OWASP A07:2021)
- Bearer/API tokens validated on every request
- Token secrets loaded from environment (not hardcoded)
- Auth middleware applied consistently to protected routes
- Service account credentials properly scoped
- JWT validation complete (signature, expiry, issuer) - if using JWTs
- Constant-time comparison for sensitive values (tokens, API keys, session IDs) — use `crypto.timingSafeEqual()` or equivalent, not `===`

### HTTPS & Transport
- External API calls use HTTPS
- Certificate validation not disabled

### Security Headers
- Content-Security-Policy (CSP) configured — prevents XSS and data injection
- X-Content-Type-Options: nosniff — prevents MIME sniffing
- X-Frame-Options or CSP frame-ancestors — prevents clickjacking
- Strict-Transport-Security (HSTS) — enforces HTTPS
- Referrer-Policy — limits referrer information leakage
- Permissions-Policy — restricts browser features (camera, geolocation, etc.)

## Type Safety

### Unsafe Casts
- No unsafe `any` casts without justification
- Type assertions (`as Type`) verified correct
- Generic constraints appropriate

### Type Guards
- Union types have exhaustive handling
- Nullable types explicitly handled (null, undefined)
- External data validated before use (API responses, AI outputs, file parsing)
- Parsed data matches expected schema (dates, numbers, enums)

### Runtime Validation
- API inputs validated (zod, io-ts, or manual)
- Config values validated at startup
- Type mismatches detected early (fail fast)

## Logic & Correctness

### Common Bug Patterns
- Off-by-one errors in loops/indices
- Empty array/object edge cases not handled
- Integer overflow/underflow (rare in JS but possible)
- Floating point comparison issues (use epsilon)
- Assignment vs comparison (= vs ==)

### Boundary Conditions
- Empty inputs handled (null, undefined, "", [], {})
- Single-element collections work correctly
- Maximum size inputs don't break logic
- Negative numbers where only positive expected
- Zero values handled appropriately
- Unicode edge cases (emojis, RTL, combining chars)

### State Management
- Race conditions in shared state
- State mutations in wrong order
- Missing state cleanup after operations
- Stale state references in closures

## Memory Leaks

### Unbounded Collections
- Arrays/Maps/Sets that grow without bounds
- Caches without eviction policy or size limits
- Queues that accumulate faster than they drain

### Event Listeners
- `.on()` without corresponding `.off()` or `.removeListener()`
- Event emitters in loops creating multiple listeners
- Missing `once()` for one-time events

### Streams and Handles
- Streams not `.destroy()`ed on error
- File handles not closed in finally blocks
- Response streams not properly ended

### Timers
- `setInterval()` without `clearInterval()`
- `setTimeout()` in loops without cleanup
- Timers not cleared on component/service shutdown

### Closures
- Closures capturing large objects unnecessarily
- Callbacks holding references to parent scopes

## Resource Leaks

### Connections
- Database connections not returned to pool
- HTTP connections not closed on error paths
- Connection pools not properly configured

### File Handles
- Files opened without corresponding close
- Streams created but not consumed or destroyed

## Async Error Handling

### Unhandled Promises
- Promises without `.catch()`
- `async` functions called without `await` or `.catch()`
- Promise chains missing error handlers

### Missing Try/Catch
- `async` functions without try/catch around await calls
- Errors not propagated to caller

### Error Swallowing
- Empty catch blocks
- Catch blocks that log but don't rethrow or handle appropriately

## Timeout and Hang Scenarios

### External API Calls
- HTTP requests without timeout option
- Third-party API calls that could hang indefinitely (Claude, Fitbit, Google)
- No circuit breaker for unreliable dependencies
- Missing retry logic for transient failures (network errors, 5xx)

### Blocking Operations
- Synchronous file/network operations in async code
- CPU-intensive loops without yielding
- Database queries without timeout

## Graceful Shutdown

### Server Shutdown
- SIGTERM/SIGINT handlers registered
- New requests rejected during shutdown
- Existing requests allowed to complete (drain)

### Resource Cleanup
- API connections closed
- File handles released
- Timers cleared
- Pending API requests aborted

## Dependency Vulnerabilities

### Package Audits
- Run `npm audit`
- Check for critical/high severity issues

### Supply Chain
- Dependencies from trusted sources
- No typosquatting package names
- **Hallucinated packages** — verify all imports resolve to real packages (AI frequently invents non-existent package names that may be claimed by attackers)
- Lock files committed and up to date
- Dependencies pinned to specific versions or known-safe ranges

## Logging

### Log Level Correctness

Verify each log statement uses the appropriate level:

| Level | Correct Usage | Anti-patterns |
|-------|---------------|---------------|
| **FATAL** | Critical failures preventing app from continuing (missing required config, DB unavailable at startup, security breaches) | Using for recoverable errors |
| **ERROR** | Operations that fail but app continues (API failures after retries exhausted, resource creation failures, unexpected exceptions) | Logging expected exceptions, errors with auto-recovery |
| **WARN** | Unexpected but recoverable conditions (resource thresholds approaching limits, deprecated config, excessive failed logins, slow API responses) | Normal operational events |
| **INFO** | Significant business events (state changes, successful completions, service startup/shutdown, milestones) | Excessive details, sensitive data |
| **DEBUG** | Implementation details for troubleshooting (DB queries, API calls/responses, config values, timing) | Enabled in production continuously |

### Log Coverage

Ensure sufficient logging for operational visibility:

- **Error paths**: All catch blocks log the error with context (operation, inputs, stack trace)
- **API boundaries**: Incoming requests and outgoing responses logged at INFO or DEBUG
- **State transitions**: Key business state changes logged at INFO
- **External calls**: Third-party API calls logged at DEBUG with timing
- **Authentication events**: Login success/failure, token refresh, session creation logged at INFO
- **Startup/shutdown**: Service initialization and graceful shutdown logged at INFO
- **Scheduled jobs**: Job start, completion, and failures logged at INFO

### Debug Coverage for Investigation

Verify DEBUG logs exist for troubleshooting all potential errors:

- **Every external API call**: Request parameters, response status, timing
- **Database operations**: Queries, parameters (sanitized), row counts
- **Business logic decisions**: Key conditional branches, computed values
- **Data transformations**: Input/output shapes, validation results
- **Configuration**: Loaded config values at startup (not secrets)
- **Queue/async operations**: Job enqueue, dequeue, processing steps

### Log Overflow Prevention

Check for patterns that could saturate the logging backend:

- **No logging in tight loops**: Avoid `logger.debug()` inside `for`/`while` processing many items
- **Batch logging for bulk operations**: Log summary (e.g., "Processed 1000 items") not individual items
- **No redundant logs**: Same information not logged multiple times per request
- **Conditional verbose logging**: High-volume debug logs should be behind feature flags or log level checks
- **Request/response body limits**: Large payloads truncated or summarized, not logged in full
- **Error log deduplication**: Repeated identical errors sampled or aggregated (e.g., "Error X occurred 50 times in last minute")
- **No stack traces at INFO/WARN**: Stack traces only at ERROR/DEBUG
- **Streaming/SSE logs**: Don't log every chunk/event in streaming operations

### Structured Logging

- **JSON format**: Logs should be structured (JSON) not plain text for machine parsing
- **Consistent fields**: Request ID, user ID, operation name included consistently
- **Proper logger used**: No `console.log`/`console.error` in production code - use proper logging framework

### Log Security

- **No sensitive data**: Passwords, tokens, API keys, session secrets never logged
- **No PII without consent**: Email, phone, address not logged or properly redacted
- **No request bodies with secrets**: Auth headers, cookie values not logged
- **Error messages sanitized**: Stack traces don't expose internal paths in production
- **Image/binary data**: Never log raw binary data or base64-encoded images

### Search Patterns for Logging Issues

Use Grep tool to find potential logging issues:

**Wrong level usage:**
- `logger\.info.*error|logger\.info.*fail|logger\.info.*exception` - errors logged at INFO
- `logger\.debug.*critical|logger\.debug.*fatal` - critical issues at DEBUG
- `logger\.error` in catch blocks for expected/recoverable errors

**Missing logs:**
- `catch\s*\([^)]*\)\s*\{[^}]*\}` - empty or log-less catch blocks
- API route handlers without any logger calls

**Log overflow risks:**
- `for.*\{[^}]*logger\.|while.*\{[^}]*logger\.` - logging inside loops
- `\.map\([^)]*logger\.|\.forEach\([^)]*logger\.` - logging in array iterations
- `logger\.(debug|info).*JSON\.stringify` - potentially large objects logged

**Security issues:**
- `logger\..*(password|secret|token|key|auth)` - potential secrets in logs
- `logger\..*req\.body|logger\..*request\.body` - request bodies might contain secrets
- `logger\..*headers` - headers might contain auth tokens

## Rate Limiting

### External API Quotas
- Rate limit handling for third-party APIs (Claude, Fitbit, Google)
- Backoff/retry logic for 429 responses
- Quota monitoring
- Token/request budgeting for AI APIs

## Test Quality (if tests exist)

### Test Coverage
- Critical paths have test coverage
- Edge cases tested
- Error paths tested

### Test Validity
- Tests have meaningful assertions (not just "doesn't throw")
- No tests that always pass
- No duplicate tests
- Mocks don't hide real bugs

### Test Data
- No real customer/user data in tests
- No production credentials in test files
- Test data clearly fictional

## Search Patterns

Use Grep tool (not bash grep) to find potential issues:

**Security:**
- `password|secret|api.?key|token` (case insensitive) - potential hardcoded secrets
- `eval\(|new Function\(` - dangerous code execution
- `exec\(|spawn\(` with variable input - command injection risk
- `fetch\(.*\$|fetch\(.*\+|fetch\(.*\`|axios.*\$|axios.*\+` - potential SSRF (user-controlled URLs in server-side fetch)

**Type Safety:**
- `as any` - unsafe type cast
- `as unknown as` - double cast (often hiding type issues)
- `@ts-ignore|@ts-expect-error` - suppressed type errors

**Memory/Resource:**
- `\.on\(` - event listeners (check for cleanup)
- `setInterval` - timers (check for clearInterval)
- `setTimeout` in loops - potential accumulation
- `new Map\(|new Set\(|\[\]` at module level - potential unbounded growth

**Async:**
- `\.then\(` without `.catch` nearby - unhandled promise
- `async ` functions - verify try/catch coverage
- `Promise\.all` - verify error handling

**Logging:**
- `console\.log|console\.warn|console\.error` - should use proper logger

## Claude API & AI Integration

This project uses Claude's tool_use API for food analysis and conversational chat. Review all Claude API integration code for these issues.

### Tool Definitions

- Tool descriptions are detailed (3-4+ sentences minimum) — the #1 factor in tool selection accuracy
- Each parameter has a `description` with examples of valid values (e.g., "Date in YYYY-MM-DD format")
- Constrained values use `enum` arrays (not free-form text described in the description)
- Required vs optional parameters correctly distinguished in `required` array
- Tool names follow `^[a-zA-Z0-9_-]{1,64}$` pattern
- `input_schema` uses `type: "object"` at the top level
- Tier 1 nutrients and other nullable fields explicitly allow null in the schema
- `strict: true` used on tool definitions for guaranteed schema conformance (eliminates parsing errors, type mismatches). Requires `additionalProperties: false` on all objects. Runtime validation still needed for `max_tokens` truncation and `refusal` edge cases.

### System Prompts

- Clear role/persona definition in the system prompt
- Tool usage guidance included: when to use each tool, when NOT to use
- No sensitive data embedded in system prompts (no API keys, user tokens, PII)
- System prompt kept in sync with tool definitions (no references to removed/renamed tools)
- Behavioral rules clear and unambiguous (Claude follows last instruction on conflict)
- For Claude 4+ models: avoid over-prompting tool use (e.g., "CRITICAL: You MUST use this tool") — newer models follow instructions more precisely and will overtrigger

### Prompt Caching

- Static content (tools, system prompt) marked with `cache_control: {type: "ephemeral"}` to enable caching (up to 90% input cost reduction, 85% latency reduction)
- Minimum cacheable length met: 1,024 tokens for Sonnet 4/4.5, 4,096 tokens for Opus 4.6/Haiku 4.5
- Cache breakpoints placed at end of static content sections
- In multi-turn conversations, final block of final message marked with `cache_control` each turn
- Cache invalidation triggers understood: changing tool definitions invalidates entire cache; changing `tool_choice`, images, or thinking params invalidates message cache
- JSON key ordering in tool_use blocks is stable (randomized key order breaks caching)
- `cache_creation_input_tokens` and `cache_read_input_tokens` monitored in API responses

### Tool Use Lifecycle (Agentic Loop)

- `stop_reason` checked for ALL values:
  - `"tool_use"` → extract tool calls, execute, send results back
  - `"end_turn"` → extract final text + optional tool output
  - `"max_tokens"` → handle truncation (incomplete tool_use blocks are invalid, retry with higher max_tokens)
  - `"refusal"` → Claude refused for safety reasons. Handle gracefully with user-facing message (Claude 4+)
  - `"model_context_window_exceeded"` → context window limit hit. Distinct from max_tokens — requires conversation compaction or message truncation (Claude 4+)
- `tool_result.tool_use_id` matches corresponding `tool_use.id` from assistant message
- All parallel `tool_result` blocks sent in a SINGLE user message (splitting degrades future parallel behavior)
- `tool_result` content blocks come BEFORE any `text` blocks in user messages
- `is_error: true` set on `tool_result` for tool execution errors (Claude handles gracefully)
- Agentic loops capped with max iteration count (prevent infinite tool-call cycles)
- After max iterations, return best available response (don't hang or throw)
- `max_tokens` sufficient for expected response + tool call overhead

### Response Validation

- `tool_use.input` validated at runtime — Claude can produce unexpected shapes even with good schemas (and even with `strict: true` on `max_tokens` truncation or `refusal`)
- Handle `stop_reason: "max_tokens"` where last content block is incomplete `tool_use` (retry with higher `max_tokens`)
- Handle empty text responses (Claude may respond with only `tool_use` blocks)
- Numeric fields validated as non-negative where appropriate
- String fields validated for non-empty where required

### Cost & Token Management

- `max_tokens` not unnecessarily large (wastes allocation budget)
- Tool definitions add tokens to every request — keep descriptions useful but not bloated
- Token usage recorded for monitoring (fire-and-forget, non-blocking — don't fail the request if recording fails)
- `tool_choice` set appropriately: `"auto"` for optional, `{"type": "tool"}` for forced, `"any"` for must-use-one, `"none"` to disable tools
- **`tool_choice` + thinking compatibility:** `{type: "tool"}` and `{type: "any"}` are INCOMPATIBLE with extended/adaptive thinking — only `"auto"` and `"none"` work with thinking enabled
- Conversation message limits enforced (prevent unbounded token growth)
- Context window management: token count checked before API calls, conversation compaction or message truncation when approaching limits
- Rate limiting applied to routes that call Claude API
- Prompt caching used on static content to reduce costs (see Prompt Caching section above)

### Model Configuration

- Production code pins exact model snapshot IDs (e.g., `claude-sonnet-4-20250514`) — aliases (e.g., `claude-sonnet-4`) can drift to newer snapshots with behavioral changes
- `temperature` and `top_p` NOT used simultaneously — this is a breaking change in Claude 4+ (use one or the other)
- For Claude Opus 4.6: use `thinking: {type: "adaptive"}` with `output_config: {effort: "..."}` — manual `budget_tokens` is deprecated
- For older models (Sonnet 4, Sonnet 4.5): use `thinking: {type: "enabled", budget_tokens: N}` when extended thinking is desired

### AI-Specific Security

- No user-controlled text injected directly into system prompts without sanitization (prompt injection risk)
- Claude API key loaded from environment variable (not hardcoded, not logged)
- Tool results returned to Claude don't include raw tokens, passwords, or session secrets
- AI-generated content (food names, descriptions) sanitized before rendering in HTML (XSS prevention)
- User descriptions validated/sanitized before inclusion in Claude API calls
- Rate limiting prevents abuse of expensive Claude API endpoints

### Error Handling

- Claude API errors (5xx, network) caught and mapped to appropriate HTTP error codes
- Claude API 429 (rate limit) handled with retry/backoff using `Retry-After` header or propagated as user-facing error
- Claude API 529 (overloaded) handled separately from 500 — transient, retry with exponential backoff
- Timeouts configured on Claude API client (don't hang on slow responses)
- Token usage recording failures don't break the main request flow
- Partial failures handled: if analysis succeeds but usage recording fails, return the analysis
- Request size under 32MB limit for Messages API
- Traffic ramp-up gradual to avoid acceleration limits (sudden spikes trigger 429s)

### Search Patterns for AI Integration Issues

Use Grep tool to find potential AI integration issues:

**Tool definitions:**
- `tool_choice` — verify appropriate setting for each use case; check compatibility with thinking if enabled
- `tools:.*\[` — find tool definition arrays, check descriptions are detailed
- `report_nutrition|search_food_log|get_nutrition_summary|get_fasting_info` — tool name references
- `strict` — check if `strict: true` is set on tool definitions for schema conformance

**Response handling:**
- `stop_reason` — verify ALL values handled (tool_use, end_turn, max_tokens, refusal, model_context_window_exceeded)
- `tool_use_id|tool_result` — verify ID matching and result formatting
- `validateFoodAnalysis|validateTool` — verify runtime validation exists

**Caching:**
- `cache_control` — verify prompt caching configured on static content (system prompt, tools)
- `cache_creation_input_tokens|cache_read_input_tokens` — verify cache metrics monitored

**Security:**
- `ANTHROPIC_API_KEY` — verify loaded from env, not hardcoded or logged
- `system:.*\$|system:.*user` — potential prompt injection (user input in system prompt)

**Cost & model config:**
- `max_tokens` — verify reasonable limits set
- `recordUsage` — verify usage tracking calls exist
- Model ID — verify pinned to snapshot (not alias) in production
- `temperature.*top_p|top_p.*temperature` — verify not both set (breaking change in Claude 4+)

## AI-Generated Code Risks

All code in this project is AI-assisted. Apply extra scrutiny for patterns AI models commonly introduce:

### Common AI Code Vulnerabilities
- **XSS vulnerabilities** (2.74x higher frequency in AI code) — check all dynamic content rendering
- **Logic errors** (75% more common) — verify branching, loop bounds, comparisons
- **Missing input validation** — AI often skips server-side validation
- **Hardcoded secrets** — AI trains on public repos full of exposed credentials
- **Code duplication** — AI frequently generates similar code instead of reusing existing abstractions
- **~45% of AI code contains security flaws** — treat all AI output as untrusted until reviewed

### AI-Specific Anti-patterns
- **Hallucinated APIs** — methods, functions, or library features that don't exist. Verify imports resolve to real exports.
- **Hallucinated packages** — non-existent npm packages that may be claimed by attackers (supply chain risk). Verify every `import` references a real, trusted package.
- **Outdated patterns** — AI may use deprecated APIs or old security practices from training data
- **Over-engineering** — unnecessary abstractions, extra error handling for impossible scenarios
- **Missing business context** — AI may not understand domain constraints, leading to incorrect logic

### Search Patterns for AI Code Issues
- Check `import.*from` for packages that don't exist in `package.json`
- Check `require\(` for the same
- Look for API method calls that don't match the library's actual interface
- Look for copied patterns (similar code blocks in multiple files that should be shared)
