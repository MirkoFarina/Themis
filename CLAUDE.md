# Themis — Claude Code Rules

## Project overview

Next.js 15 App Router · TypeScript · Tailwind CSS · NextAuth v5 · Redis (ioredis) · Gemini 2.5 API  
4-agent AI pipeline: disambiguator → analyzer → decider → formatter  
RBAC: `admin` | `normal` | `guest`

---

## Architecture: DDD bounded contexts

```
src/
  auth.ts                          # NextAuth config (root-level, Next.js convention)
  middleware.ts                    # Edge-compatible auth gate (cookie check only)

  server/                          # Backend-only — never import from components/
    domain/
      analysis/                    # Bounded context: AI pipeline
        agents/                    # disambiguator, analyzer, decider, formatter, orchestrator
        data/                      # catalog, carbon, benchmarks
      identity/                    # Bounded context: users & auth
        users.ts
        session-resolver.ts
    infrastructure/
      redis/
        client.ts                  # Redis singleton + TTL constants
        rate-limit.ts              # checkRateLimit()
      gemini/
        client.ts
        usage.ts

  shared/                          # Shared between FE and BE
    types/
      index.ts                     # ChatOption, ChatState, agent I/O schemas (Zod)
      user.ts                      # User, UserRole
      session-summary.ts           # SessionSummary
    utils/
      format.ts
      carbon-display.ts

  app/                             # Next.js pages + API routes
  components/                      # React components
  tests/
```

### Import rules (strictly enforced)

| From | May import |
|---|---|
| `app/api/*` | `server/**`, `shared/**` |
| `components/**` | `shared/**` only — never `server/**` |
| `server/domain/**` | `server/infrastructure/**`, `shared/**` |
| `server/infrastructure/**` | nothing internal |
| `shared/**` | nothing internal |

**Never** import `server/` from `components/` or `shared/`.  
**Never** import `@/lib/*` — that directory no longer exists.

### Middleware constraint

`src/middleware.ts` runs in **Edge Runtime**. It must never import anything that transitively uses Node.js APIs (`bcryptjs`, `ioredis`, crypto modules, etc.). Keep it to pure cookie/header checks only. Full auth validation happens inside route handlers via `resolveUserFromRequest()`.

---

## Code conventions

- **No comments** unless the WHY is non-obvious (hidden constraint, subtle invariant, workaround).
- **No docstrings** on functions — names and types are documentation enough.
- **No backwards-compat shims** — delete unused code outright.
- **Zod schemas** for all external inputs: include `.max()` on strings and arrays to prevent prompt injection / DoS.
- **Error responses**: use `apiError(err, label)` from `session-resolver.ts` — never leak stack traces or internal details to clients.
- **Rate limiting**: apply `checkRateLimit()` on all endpoints that trigger AI pipeline calls.
- All UI strings in **English** — no Italian in the codebase.

---

## Security review — mandatory at end of every flow

After completing any feature, fix, or refactor, run the security agent before committing:

```
/gsd-code-review
```

The agent must check at minimum:

1. **Auth**: every API route (except `/api/auth/*` and `/api/health`) calls `resolveUserFromRequest()` and handles `UnauthorizedError`.
2. **RBAC**: role checks are enforced server-side, not client-side.
3. **Input validation**: all user-supplied data passes through a Zod schema with length limits.
4. **Rate limiting**: pipeline-triggering routes use `checkRateLimit()` per role.
5. **Error leakage**: no route returns raw error messages, stack traces, or internal IDs.
6. **Edge Runtime safety**: `src/middleware.ts` contains no Node.js-only imports.
7. **Redis atomicity**: operations that must be atomic use `multi()` pipelines or `SET NX`.
8. **TTLs**: every Redis key set by the app has an explicit TTL.

If the review produces any CRITICAL or HIGH findings, they must be fixed before the PR is opened.

---

## Testing

```bash
npx tsc --noEmit     # must produce zero errors
npx vitest run       # must pass all 41+ tests
```

Tests live in `src/tests/unit/`. Redis and other infra are mocked via `vi.mock`.  
When adding new infra exports (constants, functions), update the corresponding mock in the test file.
