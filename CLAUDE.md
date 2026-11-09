# @sudobility/ratelimit_service

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`, `git push`, or `gh pr create` **only when the user explicitly asks
> in that turn**. Approval for an earlier change does not carry forward, and finishing a
> task is not permission to commit it. Stay on the current branch; no feature branches.

Shared rate-limiting backend library. Resolves per-period request limits from RevenueCat
entitlements, then checks/increments period counters in Postgres. Published to **GitHub
Packages** (scope `@sudobility`, `.npmrc`).

## Stack

- TypeScript (strict), ESM-only output. Runtime/package manager: **Bun** (not npm/yarn/pnpm).
- Tests: **vitest** (`import { describe, it, expect } from "vitest"`) — NOT `bun:test`.
- Peer deps: `@sudobility/types`, `drizzle-orm`, `hono`, `postgres`.

## Commands (all verified)

```bash
bun install
bun run verify       # typecheck -> lint -> test:unit -> build (use before publish)
bun run test:unit    # vitest run  (NOT `bun test`)
bun run lint         # eslint src/
bun run typecheck    # tsc --noEmit
bun run build        # tsc (ESM) + scripts/fix-esm-imports.mjs
```

## Architecture

Data flow (both the middleware and `RateLimitRouteHandler` compose these three helpers):

```
Request → RevenueCatHelper → EntitlementHelper → RateLimitChecker → Response
          (fetch entitlements) (resolve limits)   (check/increment DB)
```

```
src/
├── index.ts               # main exports (re-exports everything)
├── types/                 # rate-limits, entitlements (NONE_ENTITLEMENT), responses (PeriodType)
├── schema/rate-limits.ts  # Drizzle schema TEMPLATE (consumers copy into their own db/schema.ts)
├── helpers/
│   ├── RevenueCatHelper.ts        # getSubscriptionInfo(userId, testMode?)
│   ├── EntitlementHelper.ts       # getRateLimits(entitlements)
│   ├── RateLimitChecker.ts        # checkAndIncrement / checkOnly / getHistory
│   └── RateLimitRouteHandler.ts   # builds /ratelimits + /ratelimits/history responses
├── middleware/hono.ts     # createRateLimitMiddleware factory
└── utils/time.ts          # UTC period-boundary helpers
tests/                     # 6 vitest files: EntitlementHelper, RateLimitChecker,
                           # RateLimitRouteHandler, RevenueCatHelper, middleware, time
```

### Exports (package.json `exports`, all ESM)
- `.` → main entry
- `./middleware/hono` → `createRateLimitMiddleware`
- `./schema` → `rateLimitCounters`

## Business logic (non-obvious)

- **`undefined` = unlimited** for a period. Do NOT confuse with `0` (zero allowed).
- **Multiple entitlements resolve to the upper bound** — the most permissive limit wins.
  `getRateLimits(["starter","pro"])` with `pro` unlimited → all `undefined`.
- **`none` tier is required** in every `RateLimitsConfig`; it's the fallback.
- **Monthly periods are subscription-relative, not calendar months.** Based on the
  subscription start day (started Mar 5 → 3/5–4/4, 4/5–5/4, …), with day-overflow handling
  (Jan 31 → Feb 28). Calendar-month fallback only when `subscriptionStartedAt` is null.
- **Internal `undefined` vs API `null`:** `RateLimitRouteHandler.convertLimits` maps internal
  `undefined` (unlimited) to `null` for the `@sudobility/types` API shape.
- **Counter history is preserved:** one row per `(user_id, period_type, period_start)`; old
  periods are never deleted (enables usage-history UI).
- RevenueCat 404 → treat as `["none"]`; other RevenueCat errors fall back to `none` tier.

## Gotchas

- **Schema is a template, not a migration.** Changes here do NOT propagate automatically;
  downstream consumers must update their copied `db/schema.ts` and run migrations.
- **ESM-only, extensionless imports.** Source uses `moduleResolution: "bundler"` (extensionless
  relative imports); `scripts/fix-esm-imports.mjs` rewrites them to explicit `.js` paths after
  `tsc`. The build intentionally leaves `./db` specifiers unresolved (consumer-provided) — those
  warnings are expected, not failures.
- **Publishing is CI-driven** (`.github/workflows/ci-cd.yml` → `johnqh/workflows` →
  GitHub Packages). Do not `npm publish` by hand.
- Installing requires a GitHub token with read access to `@sudobility` packages in `~/.npmrc`.

## Downstream impact

Consumed by backend APIs in this workspace (e.g. `shapeshyft_api`), which import the middleware
and copy the schema template. After changing the schema template, those consumers must update
their copied schema and migrate. See `../CLAUDE.md` for the broader workspace architecture.
