# Frontend Review Checklist

Comprehensive checklist for frontend code review covering accessibility, visual design, UX, responsiveness, and performance. Based on WCAG 2.2, Core Web Vitals 2025, and modern frontend best practices.

## Accessibility & Semantics (WCAG 2.2)

### Semantic HTML
- [ ] Landmark elements used correctly (`<header>`, `<nav>`, `<main>`, `<footer>`, `<article>`, `<section>`)
- [ ] Heading hierarchy logical (h1 → h2 → h3, no skipped levels)
- [ ] Each page has exactly one `<h1>`
- [ ] Interactive elements use native HTML (`<button>`, `<a>`, `<input>`) not `<div onClick>`
- [ ] Lists use `<ul>`/`<ol>`/`<li>` instead of styled divs
- [ ] Tables use `<thead>`, `<tbody>`, `<th>` with `scope` attributes
- [ ] `<a>` tags for navigation, `<button>` for actions (not interchangeable)

### ARIA
- [ ] Interactive elements have accessible names (text content, `aria-label`, or `aria-labelledby`)
- [ ] Dynamic content uses `aria-live` regions for announcements
- [ ] Modals have `aria-modal="true"` with focus trapping
- [ ] Expandable sections have `aria-expanded` bound to React state
- [ ] Loading states use `aria-busy="true"` on the updating region
- [ ] Error messages have `role="alert"` and `aria-atomic="true"`
- [ ] No redundant ARIA (e.g., `role="button"` on `<button>`)
- [ ] ARIA attributes hyphen-cased in JSX (not camelCase)
- [ ] "No ARIA is better than bad ARIA" — verify ARIA usage is correct

### Keyboard Navigation
- [ ] All interactive elements reachable via Tab
- [ ] Tab order matches visual layout
- [ ] No keyboard traps
- [ ] Enter/Space activate buttons
- [ ] Escape closes modals/dialogs/dropdowns
- [ ] Custom shortcuts don't conflict with browser or assistive technology shortcuts
- [ ] Arrow keys navigate within composite widgets (tabs, menus, radio groups)

### Focus Management
- [ ] Focus indicators visible (>= 3:1 contrast against background)
- [ ] `outline: none` never used without replacement focus style
- [ ] Focus moves to modal content when opened
- [ ] Focus returns to trigger element when modal closes
- [ ] Skip link present and functional (targets `<main>`)
- [ ] Focus ring styling visible in both light and dark mode

### Color & Contrast
- [ ] Normal text contrast >= 4.5:1 (WCAG AA)
- [ ] Large text (>= 18pt or 14pt bold) contrast >= 3:1
- [ ] UI components and graphics contrast >= 3:1 against adjacent colors
- [ ] Information not conveyed by color alone
- [ ] Checked in both light and dark mode
- [ ] Disabled states still distinguishable (though reduced contrast is acceptable)

### Screen Readers
- [ ] Images have meaningful `alt` text (or `alt=""` for decorative)
- [ ] Icon-only buttons have `aria-label`
- [ ] Page titles unique and descriptive per route
- [ ] `<h1>` or `document.title` set for Next.js route announcer
- [ ] Form field changes announced appropriately
- [ ] Status messages use `aria-live` or `role="status"`

### Forms
- [ ] Every input has a visible `<label>` with matching `htmlFor`/`id`
- [ ] Placeholders supplement but don't replace labels
- [ ] Required fields indicated (not just by color)
- [ ] Error messages associated via `aria-describedby`
- [ ] Validation errors announced to screen readers
- [ ] Group related inputs with `<fieldset>`/`<legend>`

### Motion & Preferences
- [ ] Animations respect `prefers-reduced-motion` media query
- [ ] No auto-playing animations without pause controls
- [ ] CSS transitions/animations have `prefers-reduced-motion` overrides
- [ ] Haptic feedback has visual fallback

### WCAG 2.2 Additions
- [ ] Target size minimum 24x24 CSS pixels (Success Criterion 2.5.8)
- [ ] No drag-only interactions — button alternatives provided
- [ ] Consistent help mechanisms available across pages
- [ ] Redundant entry minimized (don't ask for same info twice)

---

## Visual Design & Consistency

### Design System Compliance
- [ ] Colors use theme variables from `globals.css` (no hardcoded values)
- [ ] Spacing uses Tailwind scale consistently (not random values)
- [ ] Border radius consistent across similar components
- [ ] Shadow/elevation hierarchy makes visual sense
- [ ] Button variants (primary, secondary, ghost) used correctly per context
- [ ] Typography scale consistent (no arbitrary font sizes)
- [ ] Similar UI problems solved the same way everywhere

### Visual Rhythm & Composition
- [ ] Vertical rhythm: consistent cadence of elements down the page
- [ ] Visual weight distribution: pages feel balanced, not lopsided
- [ ] Whitespace used intentionally to group, separate, and create breathing room
- [ ] Gestalt proximity: related elements grouped, unrelated separated
- [ ] Gestalt similarity: same-purpose elements look the same
- [ ] Gestalt continuity: eye follows a natural path without jarring breaks
- [ ] Most important element on each screen is immediately obvious
- [ ] Element proportions feel deliberate, sizes relate to each other
- [ ] Elements aligned to a grid or to neighbors (no "almost aligned" sloppiness)

### Typography
- [ ] Font sizes follow a consistent scale
- [ ] Body text >= 16px on mobile (prevents iOS input zoom)
- [ ] Line height 1.4–1.6 for body text
- [ ] Long text content handles overflow (truncation, wrapping)
- [ ] Heading sizes visually match semantic hierarchy
- [ ] Font weights distinguish hierarchy clearly
- [ ] Letter spacing appropriate for headings vs body
- [ ] Text line lengths comfortable (45–75 characters for body copy)

### Spacing & Layout
- [ ] Consistent padding/margin within component types
- [ ] Adequate whitespace between sections
- [ ] Content has `max-width` for readability on large screens
- [ ] Flex/grid layouts handle varying content lengths
- [ ] No horizontal scroll at any viewport width
- [ ] Spacing within groups tighter than spacing between groups
- [ ] Cards/panels use consistent internal padding

### Color & Theme
- [ ] All colors from theme (no `text-[#xxx]` or `bg-[#xxx]`)
- [ ] Dark mode: all components render correctly
- [ ] Dark mode: shadows and borders appropriate for dark backgrounds
- [ ] Dark mode: images and icons visible and have sufficient contrast
- [ ] Theme toggle provides immediate feedback
- [ ] System preference (`prefers-color-scheme`) respected as default

### Visual Hierarchy
- [ ] Primary actions visually prominent
- [ ] Secondary actions visually subordinate
- [ ] Destructive actions visually distinct (typically red/warning)
- [ ] Current state/step clearly indicated in multi-step flows
- [ ] Empty states have visual indicators and calls to action

---

## UX Patterns & User Flows

### Core User Flow
- [ ] Photo → Analyze → Edit → Log flow has clear progression
- [ ] User knows current step and can navigate between steps
- [ ] Back/undo available at each step
- [ ] Confirmation before committing (logging to Fitbit)
- [ ] Success feedback after completing the flow

### Cognitive Load & Information Architecture
- [ ] Each screen has one clear primary action (no competing CTAs)
- [ ] Progressive disclosure: advanced options hidden until needed
- [ ] Manageable number of choices per screen (Hick's law)
- [ ] Related information grouped logically
- [ ] Next step always obvious (user never left wondering "what now?")
- [ ] Complex forms broken into digestible steps
- [ ] Recognition over recall: options visible, not hidden

### Microcopy & Content Design
- [ ] Button labels describe action outcome ("Log to Fitbit" not "Submit")
- [ ] Error messages explain what went wrong AND what to do next
- [ ] Empty states guide the user with specific next action
- [ ] Loading messages set expectations where possible ("Analyzing your meal...")
- [ ] Confirmations are specific ("Logged 450 cal lunch to Fitbit" not "Success!")
- [ ] Terminology consistent throughout (don't mix "meal"/"food"/"entry" for same concept)
- [ ] Tone appropriate for health/fitness context (helpful, not cutesy)

### Emotional Design & Trust
- [ ] AI confidence display calibrated (not overselling certainty)
- [ ] Personal data not unnecessarily exposed on screen
- [ ] Success moments feel rewarding
- [ ] Error moments feel recoverable, not punishing
- [ ] Interface feels calm and focused, not busy or anxious
- [ ] Visual design communicates competence and reliability

### Micro-interactions & Animation
- [ ] State transitions feel smooth and intentional (no jarring jumps)
- [ ] Easing curves appropriate (ease-out for entering, ease-in for exiting)
- [ ] Animation duration appropriate (150–300ms for feedback, 300–500ms for transitions)
- [ ] Animations communicate meaning (expand = reveal, slide = navigate, fade = appear)
- [ ] No animations that block user from acting
- [ ] Hover/focus/active states have subtle transitions
- [ ] No flash of empty content before loading state appears

### Feedback & States
- [ ] Loading states for all async operations > 500ms
- [ ] Loading indicators clearly communicate "something is happening"
- [ ] Success confirmations for significant actions
- [ ] Error messages are user-friendly (not technical)
- [ ] Error messages include what to do next
- [ ] Disabled elements explain why they're disabled (tooltip or text)
- [ ] Skeleton/placeholder UI during content loading

### Error Handling
- [ ] Network errors handled gracefully (offline, timeout)
- [ ] API errors show user-friendly messages
- [ ] Form validation shows inline errors (not just alerts)
- [ ] Global error boundary catches unexpected errors
- [ ] Retry option available for transient failures

### Empty States
- [ ] Empty photo list has clear call to action
- [ ] No analysis yet has helpful prompt
- [ ] No Fitbit connection guides user to settings

### Responsive Design
- [ ] Mobile-first: base styles target mobile, breakpoints for larger
- [ ] Content accessible at 320px width
- [ ] Layouts adapt at breakpoints (not just scale)
- [ ] Navigation adapts for mobile
- [ ] Images scale with container
- [ ] Works in both portrait and landscape

### Touch & Mobile
- [ ] All interactive elements >= 44x44px touch targets
- [ ] Adequate spacing between touch targets (>= 8px)
- [ ] Primary actions within thumb reach
- [ ] No hover-dependent interactions (must work on touch)
- [ ] Gestures have button/tap alternatives
- [ ] Safe area insets handled (notch, rounded corners)

---

## Performance & Optimization

### Core Web Vitals
- [ ] LCP: Hero/primary content loads within 2.5s
- [ ] LCP: Primary image uses `priority` prop or `fetchpriority="high"`
- [ ] CLS: All images have explicit dimensions or `aspect-ratio`
- [ ] CLS: Fonts loaded without layout shift (`next/font`)
- [ ] CLS: Dynamic content has reserved space (skeleton/placeholder)
- [ ] INP: Event handlers avoid heavy synchronous work
- [ ] INP: Scroll/touch handlers marked `passive` where applicable

### Image Optimization
- [ ] `next/image` used instead of raw `<img>`
- [ ] Above-the-fold images NOT lazy loaded
- [ ] Off-screen images lazy loaded
- [ ] Image sizes appropriate for display dimensions
- [ ] HEIC conversion handled before upload

### Font Optimization
- [ ] `next/font` used (auto optimization, self-hosting)
- [ ] No external font CDN requests
- [ ] Font subsetting if applicable

### Bundle & Code Splitting
- [ ] `'use client'` only where necessary
- [ ] Heavy libraries dynamically imported (`next/dynamic`)
- [ ] No unused imports or dead code
- [ ] Named imports (tree-shakable), not barrel imports from large packages
- [ ] Client Component boundary pushed low in component tree

### Server vs Client Components
- [ ] Default Server Components unless interactivity needed
- [ ] Data fetching in Server Components
- [ ] No `useEffect` for data fetching that could be server-side
- [ ] State and event handlers only in Client Components

### PWA
- [ ] `manifest.json` has all required fields
- [ ] Icons at 192x192 and 512x512
- [ ] `theme_color` matches app theme
- [ ] `start_url` points to correct route
- [ ] Apple-specific meta tags present
- [ ] Standalone display mode configured

### Security Headers (Frontend-Relevant)
- [ ] Content-Security-Policy (CSP) configured — prevents inline script injection (XSS)
- [ ] X-Content-Type-Options: nosniff — prevents MIME sniffing
- [ ] X-Frame-Options or CSP frame-ancestors — prevents clickjacking
- [ ] No inline `<script>` or `style=` attributes that would violate a strict CSP
- [ ] No `dangerouslySetInnerHTML` without proper sanitization

### Resource Loading
- [ ] No unnecessary network requests on page load
- [ ] Prefetching for likely navigation targets
- [ ] Third-party scripts loaded with appropriate strategy

---

## Visual QA (Screenshot-Based)

This section is checked by analyzing rendered screenshots, not source code. It catches issues that code-only reviewers cannot detect.

### Layout & Composition
- [ ] Pages feel visually balanced (not lopsided or top-heavy)
- [ ] Whitespace is intentional: groups related elements, separates unrelated ones
- [ ] Vertical rhythm: consistent spacing cadence down each page
- [ ] Elements aligned to a grid or to neighbors (no "almost aligned" sloppiness)
- [ ] Content fills the viewport appropriately (not too sparse, not too cramped)

### Visual Hierarchy (as rendered)
- [ ] Primary action/content on each screen is immediately obvious
- [ ] Heading sizes create clear visual levels
- [ ] Secondary and tertiary elements are visually subordinate
- [ ] Eye follows a natural path through the content without jarring breaks

### Typography (as rendered)
- [ ] Font sizes readable at mobile scale without zooming
- [ ] Clear hierarchy between headings, body text, and labels
- [ ] No text truncated unexpectedly, overflowing containers, or visually cramped
- [ ] Numbers and data (calories, macros) are prominently readable

### Touch Targets (visual assessment)
- [ ] Interactive elements (buttons, links, nav items) look large enough to tap comfortably
- [ ] Adequate visual spacing between tappable items (no risk of accidental taps)
- [ ] Primary actions within thumb reach (lower 2/3 of screen)
- [ ] Bottom navigation items evenly spaced with comfortable tap zones

### Empty & Guard States
- [ ] "Set up Fitbit" guard screens look intentional, not broken
- [ ] Clear visual guidance on what to do next (CTA is prominent)
- [ ] Blank space in empty states feels like a design choice, not a missing feature
- [ ] Guard state messaging is centered and inviting

### Cross-Screen Consistency
- [ ] Same components (cards, buttons, inputs) look identical across all screens
- [ ] Spacing and padding consistent between similar elements on different pages
- [ ] Bottom navigation bar position, size, and active state highlighting consistent
- [ ] Page heading style (size, weight, position) consistent across screens
- [ ] Card/panel styling (border radius, shadow, internal padding) consistent
- [ ] Color usage consistent (same semantic meaning across screens)

### Overall Polish
- [ ] App feels cohesive — all screens look like they belong to the same product
- [ ] Visual density appropriate for a mobile utility app (functional, not decorative)
- [ ] Design communicates competence and reliability (clean, not cluttered)
- [ ] No screen feels visually "off" compared to the others

### Best Practices for Screenshot Analysis
- **Compare screens side-by-side mentally** — note any inconsistency in element sizing, spacing, or style
- **Trace the user journey** — landing → dashboard → analyze/quick-select → history → food-detail → settings
- **Check the "squint test"** — blur your focus: does the visual hierarchy still communicate clearly?
- **Evaluate empty states critically** — these are often the first thing new users see
- **Note first impressions** — what stands out immediately on each screen? Is it the right thing?

---

## Search Patterns Quick Reference

Use Grep on frontend files to find common issues:

| Pattern | What It Finds |
|---------|---------------|
| `<div.*onClick\|<span.*onClick` | Interactive elements that should be `<button>` |
| `outline:\s*none\|outline:\s*0` | Removed focus indicators |
| `<img\s` | Raw `<img>` tags (should use `next/image`) |
| `aria-label\|aria-labelledby` | ARIA usage to audit |
| `tabIndex\|tabindex` | Tab order modifications |
| `text-\[\|bg-\[\|border-\[` | Arbitrary Tailwind values (should use theme) |
| `'use client'` | Client component boundaries to audit |
| `useEffect.*fetch\|useEffect.*get` | Client-side data fetching (prefer Server Components) |
| `hover:` | Hover-only interactions (need touch alternative) |
| `prefers-reduced-motion` | Animation accessibility handling |
| `w-\d+\s.*h-\d+` | Check interactive element dimensions for touch targets |
| `dark:` | Dark mode styles to verify |
| `loading="lazy"\|loading="eager"` | Lazy loading configuration |
| `console\.log\|console\.warn` | Debug logging left in production code |
| `dangerouslySetInnerHTML` | Potential XSS vector |
| `<script` | Inline scripts that may violate CSP |
