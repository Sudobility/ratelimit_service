# @sudobility/ratelimit_service

Shared rate limiting backend library based on RevenueCat entitlements, with a Hono middleware and a Drizzle ORM schema template.

Given a user's RevenueCat entitlements, it resolves per-period request limits (hourly / daily / monthly), checks them against period-based counters in Postgres, and increments usage. Ships as an ESM-only package with sub-path exports for the middleware and the schema.

## Installation

This package is published to **GitHub Packages** under the `@sudobility` scope (see `.npmrc`), so installing it requires a `~/.npmrc` with a GitHub token that can read `@sudobility` packages:

```
@sudobility:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GH_PACKAGES_READ_TOKEN}
```

```bash
bun add @sudobility/ratelimit_service
```

### Peer Dependencies

Consumers must install these themselves:

- `@sudobility/types`
- `drizzle-orm`
- `hono`
- `postgres`

## Usage

```typescript
import {
  RevenueCatHelper,
  EntitlementHelper,
  RateLimitChecker,
  createRateLimitMiddleware,
} from '@sudobility/ratelimit_service';
import { rateLimitCounters } from '@sudobility/ratelimit_service/schema';

// Hono middleware setup
const middleware = createRateLimitMiddleware({
  revenueCatApiKey: process.env.REVENUECAT_API_KEY!,
  rateLimitsConfig: {
    none: { hourly: 5, daily: 20, monthly: 100 }, // "none" fallback is required
    pro: { hourly: undefined, daily: undefined, monthly: undefined }, // undefined = unlimited
  },
  db,                               // Drizzle PostgresJs instance
  rateLimitsTable: rateLimitCounters, // your copy of the schema table
  getUserId: (c) => c.get('firebaseUser').uid,
});

app.use('/api/*', middleware);
```

The middleware sets `X-RateLimit-{Hourly,Daily,Monthly}-Remaining` response headers and returns a `429` JSON response when a limit is exceeded. On a RevenueCat error it falls back to the `none` tier.

## API

### Key Classes

| Class | Purpose | Key Method |
|-------|---------|------------|
| `RevenueCatHelper` | Fetch user entitlements from the RevenueCat API | `getSubscriptionInfo(userId, testMode?)` |
| `EntitlementHelper` | Map entitlements to rate limits | `getRateLimits(entitlements)` |
| `RateLimitChecker` | Check limits and increment DB counters | `checkAndIncrement(userId, limits, subscriptionStartedAt)` |
| `RateLimitRouteHandler` | Build API responses for status/history endpoints | `getRateLimitsConfigData(userId)`, `getRateLimitHistoryData(userId, periodType)` |

### Middleware

`createRateLimitMiddleware(config)` -- Hono middleware factory. Required config: `revenueCatApiKey`, `rateLimitsConfig`, `db`, `rateLimitsTable`, `getUserId`. Optional: `shouldSkip`, `getTestMode`.

### Schema

`rateLimitCounters` -- Drizzle schema template. Consumers **copy** it into their own `db/schema.ts` and run their own migrations; it is not a migration itself.

### Sub-path Exports

- `@sudobility/ratelimit_service` -- main entry (classes, types, utilities)
- `@sudobility/ratelimit_service/middleware/hono` -- middleware only
- `@sudobility/ratelimit_service/schema` -- Drizzle schema template

### Key Types

`RateLimits`, `RateLimitsConfig`, `RateLimitCheckResult`, `PeriodType`

## Development

```bash
bun install          # install dependencies
bun run verify       # typecheck -> lint -> test:unit -> build (run before publishing)
bun run test:unit    # run the vitest suite
bun run test:watch   # vitest in watch mode
bun run lint         # ESLint over src/
bun run typecheck    # tsc --noEmit
bun run build        # emit ESM to dist/, then rewrite relative imports to explicit paths
```

Tests run on **vitest** (not `bun test`). The build is **ESM-only**; `scripts/fix-esm-imports.mjs` rewrites extensionless relative imports so Node's native ESM loader can resolve them.

Publishing is handled by CI (GitHub Actions `johnqh/workflows` → GitHub Packages); see `.github/workflows/ci-cd.yml`.

## License

BUSL-1.1
