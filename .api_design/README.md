# Project Non-Negotiables

**These are not guidelines.** They are not softened for prototypes, demos, spikes, or "we'll fix it later." If a task appears to require violating one, stop and say so — do not implement the violation and note it in a comment.

---

## Part 1 — Secrets and configuration

1. Never hard-code secrets, API keys, tokens, DB credentials, or any sensitive value. Always read from environment variables (`process.env.X`, etc.).
2. Every secret used in code must have a matching empty/placeholder entry in `.env.example`, with a one-line comment explaining what it is and where to get it (e.g. `STRIPE_SECRET_KEY= # from Stripe Dashboard > Developers > API keys`).
3. Never commit `.env`, `.env.local`, or any real credentials file. Confirm `.gitignore` covers these before the first commit.
4. Never print, log, or return secrets or full tokens — not even in debug mode. Mask/redact (e.g. `sk_live_****1234`) if a value must be shown at all.
5. Separate secrets by environment (dev/staging/prod) and never let dev code silently fall back to prod credentials.
6. Rotate-friendly design: read secrets at request/startup time, never bake them into build artifacts.

## Part 2 — API design

7. APIs must never return more fields than the client needs — no `SELECT *`, no dumping full DB rows or full user objects. Use explicit response DTOs/serializers.
8. Sanitize all user input server-side, regardless of what client-side validation already did. Never trust the client.
9. Sensitive or destructive actions (delete account, change email/password, revoke access, refund, cancel subscription) require an explicit confirmation step, and re-auth or step-up verification for high-risk ones.
10. Rate-limit all sensitive/auth endpoints (login, signup, password reset, OTP, payment, search) and back off exponentially on repeated failures.
11. Never leak whether a specific email/username/account exists — login, signup, password-reset, and invite flows return identical generic responses *and* comparable timing.
12. Enforce strict file upload validation server-side (MIME type by content-sniffing not just extension, max size, allowed types) before the file touches business logic or storage.
13. Version APIs deliberately (e.g. `/api/v1/...`) so breaking changes don't silently break clients.
14. Apply least-privilege DB roles/service accounts — the API's DB user should not have permissions it doesn't need (no `DROP`, no cross-tenant access).

## Part 3 — Authentication, authorization, access control

15. Every route/page/component behind auth is **denied by default**: it requires an explicit authorization check, not just "not shown in the UI."
16. Enforce authorization at the data layer too (row-level checks / tenant scoping), not only at the route/middleware layer. A valid session must never let a user fetch another user's or another tenant's data by guessing an ID (IDOR prevention).
17. Store auth tokens securely on the client: prefer `httpOnly`, `Secure`, `SameSite=Strict/Lax` cookies over `localStorage`/`sessionStorage`. If a token must be client-readable, keep it short-lived.
18. Use short-lived access tokens plus refresh token rotation; invalidate refresh tokens on logout, password change, or suspected compromise.
19. Hash passwords with a slow, salted algorithm (bcrypt/argon2) — never MD5, SHA1, or plaintext.
20. Enforce session/token expiry and re-authentication for sensitive settings pages (billing, security settings, API keys).

## Part 4 — Payments and billing

21. All payment/billing logic is server-authoritative — price, quantity, discounts, and entitlements are computed and verified server-side. The client can never dictate the amount charged.
22. Make all payment operations idempotent (idempotency keys on charge, refund, and webhook handlers) so retries or double-clicks never double-charge or double-fulfill.
23. Verify payment webhooks (signature verification, e.g. Paystack webhook secret) before trusting their payload. Never accept unsigned or unverified webhook calls.
24. Reconcile subscription/entitlement state from the payment provider as source of truth, not from a client-sent flag.
25. Log payment events for audit purposes, but never log full card numbers, CVVs, or raw webhook secrets.

## Part 5 — Input handling, XSS, injection

26. Sanitize and validate all user input at every trust boundary. Client validation is UX only, never a security control.
27. Never render user-submitted content as raw HTML/JS. Escape output by default; if rich text is required, sanitize with an allowlist-based library (e.g. DOMPurify). Never `dangerouslySetInnerHTML` on unsanitized input.
28. Use parameterized queries or ORM query builders exclusively — never string-concatenate SQL.
29. Set a Content-Security-Policy header to reduce XSS blast radius as defense-in-depth (not a substitute for sanitization).

## Part 6 — CORS and network

30. Restrict CORS to an explicit allowlist of known origins. Never `Access-Control-Allow-Origin: *` on authenticated endpoints.
31. Force HTTPS everywhere (HSTS enabled); redirect all HTTP to HTTPS.
32. Set standard security headers (`X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`).

## Part 7 — Logging, errors, monitoring

33. Never leak secrets, stack traces, internal paths, or DB errors in API responses shown to end users. Return generic error messages; log details server-side only.
34. Structured logs must redact PII and secrets before being written or shipped to a logging provider.
35. Log security-relevant events (failed logins, password resets, permission denials, payment events) for audit trails, without logging sensitive payloads.
36. **Fail closed**: if an auth or permission check errors, deny access. Never default-allow on error.

## Part 8 — Code quality and maintainability

37. Keep files reasonably sized (soft cap ~500 lines) — split into reusable modules/functions/components once a file grows past this.
38. Write clear, developer-readable comments in every file explaining **intent**, not just what the code does.
39. Use descriptive, consistent file and variable naming (kebab-case, or the project's established convention). No `temp.js`, `test2.ts`, `util_final_v2.js`.
40. Only use well-maintained, actively-supported, widely-adopted libraries; check for known CVEs before adding a new dependency.
41. Keep dependencies pinned/locked (lockfile committed) and periodically audited (`npm audit` or equivalent).
42. Write code so it's testable — pure functions where possible, side effects isolated — even if tests aren't written immediately.

## Part 9 — Rules for the AI agent itself

43. Never invent or assume a value for a secret, API key, or config. Leave a placeholder and ask the user to supply it.
44. Before writing auth/payment/security-sensitive code, state the assumptions being made (e.g. "assuming session-based auth") and flag ambiguity rather than guessing silently.
45. When editing existing security-sensitive code, do not weaken existing protections (removing a rate limiter, loosening CORS, disabling a validation check) without explicitly calling it out.
46. When adding a new dependency, name it explicitly and briefly note why it was chosen, so it can be reviewed before installing.
47. Prefer well-known framework/library security primitives (built-in CSRF protection, ORM parameterized queries) over hand-rolled security logic.
48. Any TODO or known limitation introduced must be a visible `// TODO(security):` comment, never left silent.
49. When generating example data or test fixtures, never use real-looking secrets, real emails, or realistic PII — use obviously fake placeholders.
50. Summarize security-relevant changes at the end of a task (what was added or changed touching auth, payments, or data access) so they can be reviewed before deploying.

---

## Self-check before reporting a task complete

- [ ] No secret, key, or credential is hard-coded; `.env.example` updated if a new one was introduced.
- [ ] Auth checks are deny-by-default and enforced at the data layer, not just the route.
- [ ] User-controlled and repository-derived content is escaped wherever it renders.
- [ ] New dependencies named, justified, and locked.
- [ ] Every touched file has intent comments; none exceeds the ~500-line soft cap.
- [ ] Security-relevant changes summarized in the final message (rule 50).
