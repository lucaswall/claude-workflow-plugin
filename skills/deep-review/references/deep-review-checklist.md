# Deep Review Checklist

Cross-domain checklist organized around interaction patterns. Each section requires reasoning about how multiple files work together — not just individual file quality.

## 1. Data Flow Integrity

Trace data from source to display and back.

### API Contract Alignment
- Response type from API route matches what the client component expects
- All fields the component accesses actually exist in the API response
- Optional/nullable fields in the response are handled in the component (not bare `.field` access)
- Error response shape from API is handled by the client error handling
- Request body shape from client matches what the API route validates/expects

### Type Safety Across Boundaries
- Shared types in `src/types/` used by BOTH the API route and the client
- No `as any` or `as Type` casts that paper over a contract mismatch
- API response transformations preserve type safety (no lossy conversions)
- Fitbit API responses validated before use (external data boundary)
- Claude API responses validated before use (AI output boundary)

### Data Transformation
- Date/time values serialized and deserialized correctly across client/server boundary
- Numeric values (calories, weight) maintain precision through transformations
- Arrays/collections handled correctly when empty, single-element, or large
- Undefined vs null vs missing key handled consistently

## 2. State Lifecycle

### Staleness
- SWR cache keys unique enough to prevent showing wrong data
- SWR revalidation triggers when data should refresh (after mutations, on focus, on reconnect)
- Component state reset when route params change (key prop or useEffect dependency)
- No stale closures capturing old state in event handlers or callbacks
- Optimistic updates (if any) correctly roll back on failure

### Race Conditions
- Rapid user actions don't cause interleaved API calls with wrong results
- Navigation away during pending API call doesn't cause state update on unmounted component
- Concurrent mutations to same resource handled (last-write-wins or conflict detection)
- AbortController used for cancellable requests when component unmounts or user navigates

### Mount/Unmount
- useEffect cleanup functions clean up subscriptions, timers, AbortControllers
- Component handles being unmounted and remounted (React StrictMode double-mount)
- Event listeners added in useEffect have corresponding removal in cleanup

## 3. Error Path Completeness

### API Route Errors
- Every `await` in API routes wrapped in try/catch or within a try block
- Database errors return appropriate HTTP status (500, not swallowed)
- External API errors (Fitbit, Claude) return appropriate HTTP status
- Validation errors return 400 with useful error message
- Auth failures return 401/403 consistently
- Error responses use standardized format (ErrorCode from src/types)

### Client Error Handling
- fetch/useSWR error states rendered with user-friendly messages
- Network errors (offline, timeout) distinguished from server errors
- Error messages include actionable guidance ("Try again" / "Check your connection")
- Error boundaries catch rendering errors in the component tree
- Form submission errors shown inline, not just console.error

### Error Recovery
- User can retry after transient failures without full page refresh
- Form data preserved after submission failure (not cleared on error)
- Partial success states handled (e.g., analysis succeeded but Fitbit log failed)

## 4. Edge Cases

### Empty/Missing Data
- Component renders correctly with no data (empty arrays, null values)
- Empty state has clear messaging and call to action
- API returns appropriate response for empty results (empty array, not 404)
- First-time user experience works (no prior data, no Fitbit connection)

### Boundary Values
- Very long text (food descriptions, names) handled (truncation, wrapping)
- Very large numbers (high-calorie meals) display correctly
- Zero values displayed correctly (0 calories is valid, not treated as "missing")
- Special characters in user input don't break display or queries
- Multiple rapid submissions handled (debounce or button disable)

### Navigation
- Multiple browser tabs don't conflict
- Back/forward browser navigation preserves expected state
- Deep linking works (URL directly to a specific state)
- Page refresh preserves or gracefully resets state

## 5. Security Surface

### Authentication & Authorization
- Route handler checks session/auth before processing
- Session validation matches CLAUDE.md convention (getSession + validateSession for browser, validateApiRequest for v1)
- OAuth tokens stay server-side, never exposed to client
- Allowlist enforcement (ALLOWED_EMAILS) applied consistently
- Constant-time comparison for sensitive values (tokens, API keys) — use `crypto.timingSafeEqual()`, not `===`

### Input Validation
- User input sanitized before database operations
- File uploads validated (size, type) before processing
- API request bodies validated with appropriate schemas
- No path traversal via user-controlled file paths
- No XSS via user-controlled content rendered as HTML (dangerouslySetInnerHTML, unescaped output)
- No SSRF — server-side requests use allowlisted URLs/domains, no user-controlled URLs passed to fetch

### Sensitive Data
- Tokens, secrets, session data not logged (check pino calls)
- Sensitive data not in client-accessible responses
- Images not logged or stored longer than needed
- Error responses don't leak internal details (stack traces, file paths)

## 6. User Experience

### Loading States
- loading.tsx exists with Skeleton matching page layout
- Async operations show loading indicators (spinner, skeleton, disabled button)
- Loading state appears immediately (no gap before indicator shows)
- Partial loading: only the updating section shows loading, not the entire screen

### Feedback
- Every user action has visible feedback (button state change, toast, inline message)
- Success confirmations are specific ("Logged 450 cal to Fitbit" not "Success")
- Destructive actions require confirmation
- Long operations show progress or at least a descriptive message

### Accessibility
- Interactive elements use semantic HTML (button, a, input — not div onClick)
- Images have meaningful alt text
- Form inputs have associated labels (htmlFor/id)
- Focus management correct for modals/dialogs (trap focus, return on close)
- Touch targets >= 44x44px
- Color not the sole indicator of state or information

### Responsive & Mobile
- Layout works at 320px width without horizontal scroll
- Touch targets have adequate spacing (no accidental taps)
- No hover-only interactions (must work on touch)
- Safe area insets handled (notch, home indicator)
- Keyboard doesn't obscure input fields on mobile

## 7. Performance

### Rendering
- No unnecessary re-renders (check useEffect/useMemo/useCallback dependency arrays)
- Large lists use virtualization or pagination if applicable
- Client Component boundary pushed as low as possible in component tree
- Server Components used for static/data-fetching content

### Bundle
- Heavy libraries dynamically imported (next/dynamic)
- No barrel imports from large packages
- `'use client'` only where truly needed (hooks, event handlers, browser APIs)

### Network
- No redundant API calls on mount
- SWR deduplicates concurrent requests for same key
- Images use next/image (auto optimization, lazy loading)
- No waterfall requests that could be parallel

## 8. AI Integration (Claude API)

This project uses Claude's tool_use API for food analysis and conversational chat. When the reviewed feature involves Claude API integration, trace the full AI data flow.

### Tool Definition Quality

- Tool descriptions are detailed (3-4+ sentences minimum) — the single most important factor for tool selection accuracy
- Each parameter has a description with examples of valid values
- Constrained values use `enum` arrays (not free-form descriptions)
- Required vs optional parameters correctly specified
- `input_schema` top-level type is `"object"`
- Tool names follow `^[a-zA-Z0-9_-]{1,64}$` pattern
- `strict: true` set on tool definitions for guaranteed schema conformance (eliminates parsing errors, type mismatches). Requires `additionalProperties: false`. Runtime validation still needed for truncation/refusal edge cases.

### System Prompt & Behavioral Design

- Clear role/persona definition in the system prompt
- Tool usage guidance: when to use each tool, when NOT to use, what information each tool does NOT return
- System prompt in sync with tool definitions (no references to renamed/removed tools)
- No sensitive data in system prompts (API keys, tokens, PII)
- Behavioral rules unambiguous (Claude follows last instruction when conflicting)
- Conversation context maintained: initial analysis baseline included when relevant
- For Claude 4+ models: avoid over-prompting tool use — newer models follow instructions precisely and will overtrigger on aggressive language like "CRITICAL: You MUST use this tool"

### Prompt Caching

- Static content (tools, system prompt) marked with `cache_control: {type: "ephemeral"}` for up to 90% input cost reduction and 85% latency reduction
- Minimum cacheable length met: 1,024 tokens for Sonnet 4/4.5, 4,096 tokens for Opus 4.6/Haiku 4.5
- In multi-turn conversations, final block of final message marked with `cache_control` each turn
- Cache invalidation triggers understood: changing tool definitions invalidates entire cache; changing tool_choice/images/thinking invalidates message cache
- `cache_creation_input_tokens` and `cache_read_input_tokens` monitored in responses

### Tool Use Lifecycle (Agentic Loop)

- `stop_reason` checked for ALL return values:
  - `"tool_use"` → extract tool calls, execute, send results back
  - `"end_turn"` → extract final text + optional tool output
  - `"max_tokens"` → handle truncation (incomplete tool_use blocks are invalid)
  - `"refusal"` → Claude refused for safety reasons. Handle gracefully with user-facing message (Claude 4+)
  - `"model_context_window_exceeded"` → context window limit hit. Requires conversation compaction or message truncation (Claude 4+)
- `tool_result.tool_use_id` matches corresponding `tool_use.id`
- ALL parallel `tool_result` blocks in a SINGLE user message (splitting across messages degrades future parallel tool behavior)
- `tool_result` content blocks come BEFORE any `text` blocks in user messages (API requirement — violating causes 400 error)
- `is_error: true` set on tool_result for execution failures (Claude handles gracefully)
- Agentic loops capped with max iteration count (prevent infinite tool-call cycles)
- After max iterations reached: return best available response, don't hang

### Response Validation

- `tool_use.input` validated at runtime even when `strict: true` is set — Claude can still produce unexpected shapes on `max_tokens` truncation or `refusal`
- Numeric fields validated as non-negative where appropriate (calories, protein, etc.)
- String fields checked for non-empty where required (food_name, notes)
- Keywords normalized (lowercase, deduplicated, capped at max count)
- Handle empty text blocks (Claude may respond with only tool_use, no text)

### AI Data Flow Tracing

Follow data through the full AI pipeline:

1. **Input preparation** — Images converted to base64, description sanitized, message history formatted
   - Are images validated (size, type) before sending?
   - Is user text sanitized before inclusion in Claude messages?
   - Are message history items properly typed and ordered?

2. **Claude API call** — System prompt, tools, tool_choice, max_tokens
   - Is tool_choice appropriate for the context? ("tool" for forced, "auto" for optional)
   - Is max_tokens sufficient for the expected response?
   - Are tools appropriate for this call? (no write tools in read-only contexts)
   - Is prompt caching configured on static content (system prompt, tools)?
   - **tool_choice + thinking compatibility:** `{type: "tool"}` and `{type: "any"}` are INCOMPATIBLE with extended/adaptive thinking — only `"auto"` and `"none"` work

3. **Response processing** — Tool calls extracted, validated, executed
   - Are all tool_use blocks processed (not just the first)?
   - Is the tool loop iterated correctly (not returning after first tool call)?
   - Are tool results formatted as clean text for Claude to interpret?

4. **Final extraction** — Text + optional analysis returned to client
   - Is the final text response extracted from all text blocks (joined)?
   - Is the optional analysis (from report_nutrition tool) validated before return?
   - Is usage recorded (fire-and-forget, non-blocking)?

5. **Client rendering** — Response displayed in chat UI
   - Are assistant messages with analysis rendered with nutrition cards?
   - Is the conversation state updated correctly (both text and analysis)?
   - Does the latest analysis computation work with the new message?

### Cost & Token Management

- max_tokens not unnecessarily large
- Tool definitions concise but complete (sent with every request)
- Conversation message limits enforced (prevent unbounded token growth)
- Context window management: token count checked before API calls, conversation compaction when approaching limits
- Rate limiting applied to Claude API routes
- Token usage tracking present and non-blocking
- Prompt caching configured on static content to reduce costs (see Prompt Caching section)
- Production code pins exact model snapshot IDs — aliases can drift to newer snapshots with behavioral changes

### Model Configuration

- `temperature` and `top_p` NOT used simultaneously (breaking change in Claude 4+)
- For Opus 4.6: use `thinking: {type: "adaptive"}` with `output_config: {effort: "..."}` — manual `budget_tokens` deprecated
- For Sonnet 4/4.5: use `thinking: {type: "enabled", budget_tokens: N}` when thinking desired

### AI-Specific Security

- No user-controlled text injected raw into system prompts (prompt injection)
- Claude API key loaded from environment variable (never hardcoded, never logged)
- Tool results don't expose raw credentials, tokens, or session secrets
- AI-generated content (food names, descriptions, notes) sanitized before HTML rendering (XSS)
- Rate limiting prevents abuse of expensive Claude API calls
- Token usage tracked for cost monitoring

### Error Handling

- Claude API 429 (rate limit) handled with retry/backoff using `Retry-After` header
- Claude API 529 (overloaded) handled separately from 500 — transient, retry with exponential backoff
- Timeouts configured on Claude API client
- Request size under 32MB limit for Messages API
- Token usage recording failures don't break the main request flow

## 9. AI-Generated Code Risks

All code in this project is AI-assisted. When tracing data flows and interactions, watch for these AI-specific patterns:

### Cross-Domain AI Issues
- **Hallucinated APIs** — API calls to methods/endpoints that don't exist or have wrong signatures. Verify against actual library docs.
- **Hallucinated packages** — non-existent npm packages that may be claimed by attackers. Verify every `import` references a real package in `package.json`.
- **Contract mismatches introduced by AI** — client assumes response fields that the API doesn't return, or vice versa
- **Copy-paste patterns** — similar handler logic duplicated across routes instead of shared through a lib module
- **Missing validation at boundaries** — AI often generates the "happy path" and skips validation of external data (Fitbit responses, Claude outputs, user input)
- **Inconsistent error handling** — some error paths return proper responses while others silently fail or return generic errors
- **Over-abstraction** — unnecessary wrappers, helpers, or config for one-time operations
