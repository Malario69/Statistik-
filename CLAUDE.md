# CLAUDE.md — Steam Achievements Hub

This repository contains a security-sensitive web application that processes Steam identities, game-library data, achievement progress, and locally persisted history.

Read `PROJECT_SPEC.md` before making architectural changes.

## 1. Non-negotiable security rules

1. NEVER expose `STEAM_WEB_API_KEY`, `SESSION_SECRET`, database credentials, Redis credentials, provider secrets, or tokens to browser/client code.
2. NEVER commit real secrets, `.env`, credentials, tokens, private keys, production URLs containing credentials, or secret-bearing logs.
3. NEVER ask end users to provide their Steam Web API key.
4. Steam authentication must use Steam OpenID; the application must never collect Steam passwords.
5. Validate authorization on every user-scoped read/write operation. Never trust a user id/SteamID from the browser without matching it to the authenticated session.
6. Validate all external/provider responses at runtime before persisting them.
7. Do not log cookies, session identifiers, API keys, Authorization headers, or full URLs containing secrets.
8. Avoid SSRF: do not fetch arbitrary user-controlled URLs. External hosts/providers must be allow-listed by adapter implementation.
9. Use secure defaults for cookies/sessions (`HttpOnly`, `Secure` in production, suitable `SameSite`).
10. State-changing browser endpoints must have CSRF protections appropriate to the framework/session strategy.

If a requested implementation conflicts with these rules, stop and explain the conflict instead of implementing an insecure workaround.

## 2. Architecture boundaries

Use the monorepo boundaries from `PROJECT_SPEC.md`.

### Steam API

All official Steam Web API access belongs in `packages/steam-api`.

Do not scatter direct calls to `api.steampowered.com` through route handlers/components/workers.

### Steam Store / Community

Store metadata and Community/guide/profile-feature retrieval belong behind dedicated adapters (`packages/steam-store`, `packages/steam-community`).

Do not put HTML parsing or provider-specific selectors into core achievement/recommendation logic.

### Database

All Prisma schema/client/database logic belongs in `packages/database` or a clearly documented repository/service layer using it.

Do not instantiate uncontrolled database clients throughout the project.

### Achievement engine

Completion percentages, schema comparison, schema hashing, and completion-event derivation belong in `packages/achievement-engine` and should be pure/testable wherever practical.

### Recommendation engine

Recommendation scoring belongs in `packages/recommendation-engine`.

The first implementation must be deterministic and explainable. Return scoring factors/reasons, not only a numeric score.

### Providers

Completion time, difficulty, hunting flags, and other third-party metadata must use provider interfaces. Core code must not depend on one external website's response shape.

## 3. Synchronization rules

Steam synchronization is a background job, not a long-running request handler.

Required invariants:

- sync is idempotent
- retries do not create corrupt duplicate history
- historical successful snapshots are immutable
- a failed sync does not overwrite `lastSuccessfulSyncAt`
- partial provider failures do not destroy valid Steam data
- duplicate concurrent full syncs for one user are prevented/coalesced
- provider calls have timeouts
- transient failures use bounded exponential backoff with jitter where appropriate
- obey provider/API rate limits

Never delete old completion snapshots as part of normal synchronization.

## 4. Achievement schema rules

Never identify achievement definitions by display name. Use Steam's stable achievement API name scoped to the Steam AppID/game.

Schema hashes must be deterministic:

1. normalize relevant stable fields
2. sort by achievement API name
3. deterministic serialization
4. SHA-256

A schema hash change must not automatically imply achievements were added; perform a set diff to classify added/removed/changed definitions.

## 5. Completion semantics

Keep these concepts separate:

- `isPerfect`: user unlocked all achievements currently defined by the schema
- profile/app eligibility: whether the game counts/is usable for Steam profile features
- user privacy/private-game visibility: whether the application can observe the user's state

Do not collapse them into one boolean.

A previously perfect game becoming imperfect because new achievements were added must preserve the old perfect snapshot and create a `PERFECTION_LOST` event with useful diff metadata.

## 6. External metadata provenance

Never store difficulty/completion-time/community metadata as if it were an objective Steam fact.

Persist, where applicable:

- normalized value
- provider/source
- source URL/reference
- retrieved timestamp
- confidence

If multiple providers exist, keep provider records separately and derive a preferred/current view explicitly.

## 7. TypeScript and validation

- TypeScript strict mode must stay enabled.
- Do not use `any` unless unavoidable and documented locally.
- Use runtime schemas (for example Zod) for environment variables, HTTP inputs, queue payloads, and external responses.
- Prefer discriminated unions/enums for state machines and event types.
- Monetary values must not use floating-point assumptions if introduced later.

## 8. Error handling

Classify errors where useful:

- authentication/authorization
- privacy/not-visible
- Steam/provider rate limit
- transient upstream error
- permanent upstream/data error
- validation error
- internal error

User-visible errors should be helpful but must not reveal internals, credentials, SQL, stack traces, or provider secrets.

## 9. Testing requirements

Every feature that changes core business logic must include tests.

At minimum, heavily test:

- achievement schema canonicalization/hash
- schema set diff
- completion percentage
- zero-achievement games
- hidden achievements
- perfection gained
- perfection lost after achievements added
- achievements removed from schema
- changed definition without changed count
- retries/idempotency of sync writes
- recommendation scoring and explanation
- authorization boundaries

Use fixtures/mocks for Steam/provider APIs in automated tests. CI must not require a real Steam API key.

## 10. Database changes

Every schema change requires a Prisma migration.

Before changing the schema:

- consider uniqueness constraints
- consider indexes for common user/game/AppID queries
- preserve immutable history
- avoid destructive migration paths unless explicitly approved

Never silently drop data/history.

## 11. Frontend guidelines

- Server-side sensitive operations only.
- Client components receive only data they need.
- All loading/error/empty/privacy states should be explicit.
- The UI must distinguish unavailable data from zero values.
- Show provenance for hunting metadata where useful.
- Show why recommendations are recommended.
- Accessibility: semantic controls, keyboard support, labels, focus states, adequate contrast.

## 12. Git workflow

For non-trivial tasks:

1. Work on a feature branch.
2. Keep changes focused on the issue.
3. Run lint/typecheck/tests/build as applicable.
4. Open a pull request describing:
   - what changed
   - important design decisions
   - security/privacy impact
   - test coverage
   - follow-up work

Do not directly push large feature implementations to `main` unless explicitly instructed.

## 13. Dependency policy

Before introducing a dependency, prefer established, actively maintained libraries and explain why the dependency is needed.

Avoid packages that merely wrap a tiny amount of straightforward platform functionality.

Pin/lock dependencies with the repository's package manager lockfile.

Never use an unofficial Steam wrapper merely to avoid implementing simple official HTTP endpoints in the Steam adapter.

## 14. Environment variables

Expected initial variables:

```text
DATABASE_URL
REDIS_URL
STEAM_WEB_API_KEY
APP_BASE_URL
SESSION_SECRET
```

Provide `.env.example` with empty/example-safe values only.

Environment parsing must fail fast on missing/invalid production requirements.

## 15. Documentation expectations

When adding a provider, queue, service, non-obvious algorithm, or security-sensitive flow, update relevant documentation.

If implementation and `PROJECT_SPEC.md` diverge intentionally, update the specification or document the accepted decision.

## 16. Things Claude must not invent

Do not fabricate undocumented Steam API fields/endpoints or claim an endpoint is official without verifying official/current documentation when that distinction matters.

Do not assume a scraping selector is stable. Isolate and test scraping/parsing logic and handle changes gracefully.

Do not claim BSI certification/compliance. Use phrasing such as `BSI-oriented security baseline` unless a formal audit/certification has actually occurred.

## 17. Definition of done

A task is not complete merely because it compiles. Relevant tasks should have:

- correct architecture placement
- runtime validation
- secure secret handling
- meaningful errors
- tests
- documentation when required
- lint/typecheck/test success
- no accidental credentials or generated junk committed
