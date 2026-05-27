# PRD — Nest Auth: Cookie-based JWT Authentication with RBAC

## Problem Statement

As a developer learning NestJS, I have no working reference implementation that ties together cookie-based JWTs, role-based access control, and proper session revocation. Every tutorial I find either puts the JWT in an `Authorization` header (exposing it to XSS via `localStorage`), skips role guards, or relies on token expiry alone for "logout" — so a stolen cookie remains valid for the full token lifetime. I want a small but correct backend that demonstrates the full lifecycle (register → login → me → update → admin operations → logout) with the security trade-offs explicit.

## Solution

Build a NestJS service with two domain modules — `AuthModule` and `UsersModule` — backed by Prisma + PostgreSQL.

- Authentication uses JWTs stored in `httpOnly`, `sameSite=lax` cookies, signed with HS256.
- Every authenticated request looks the user up by id and compares a `tokenVersion` value in the JWT payload against the same field on the user row. Bumping `tokenVersion` (on logout, password change, or role change) invalidates all existing JWTs for that user immediately.
- A `RolesGuard` paired with a `@Roles(...)` decorator gates admin-only routes.
- Admin can list users, change a user's role, and delete a user. Service-layer guards prevent self-delete, self-demote, and any operation that would leave the system with zero admins.
- First admin is seeded by a script (`prisma/seed.ts`) reading `ADMIN_EMAIL` / `ADMIN_PASSWORD` from env; there is no public route for creating admins.

## User Stories

1. As a new user, I want to register with email, password, and name, so that I get an account and am logged in immediately via a cookie.
2. As a registered user, I want to log in with email and password, so that the server sets an `access_token` cookie I don't have to manage in JS.
3. As a logged-in user, I want to call `GET /users/me`, so that I can confirm who the server thinks I am.
4. As a logged-in user, I want to update my own name, email, or password via `PATCH /users/update`, so that I can keep my profile current.
5. As a logged-in user, when I change my password, I want my existing cookie/token to be invalidated, so that any device that previously had access loses it.
6. As a logged-in user, I want `POST /auth/logout` to invalidate my session server-side (not just clear the cookie in my browser), so that a stolen cookie can't continue to be used.
7. As a logged-in user, I want my JWT to stop working the moment my account is deleted, so that there's no grace period where a removed user can still act.
8. As a logged-in user, I want `GET /users/:id` to return another user's public profile (id, name, role, createdAt) without exposing their email, so that the API isn't a user-enumeration tool.
9. As an admin, I want `GET /users/:id` to return the full user object (minus password), so that I have the information I need to administer the system.
10. As an admin, I want `GET /users`, so that I can see every account in the system.
11. As an admin, I want `POST /users/:id/role` to change a user's role between `USER` and `ADMIN`, so that I can manage who has elevated access.
12. As an admin, when I change someone's role, I want their existing JWT to be invalidated, so that the new permissions take effect on their next request.
13. As an admin, I want `DELETE /users/:id` to permanently remove a user, so that I can clean up bad actors and test accounts.
14. As an admin, I want to be blocked from deleting or demoting myself, so that I cannot accidentally lock myself out.
15. As an admin, I want to be blocked from any role change or deletion that would leave the system with zero admins, so that the system cannot become unadministrable.
16. As an admin, I want the first admin account to be created via a seed script using env credentials, so that no public registration route can ever produce an admin.
17. As a developer, I want all env vars (`DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `CORS_ORIGIN`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`) validated at boot by a schema, so that misconfiguration fails loudly instead of at first request.
18. As a developer, I want the JWT cookie to be `httpOnly`, `sameSite=lax`, and `secure` in production, so that I'm not relying on application-layer code to keep tokens out of JS.
19. As a developer, I want CORS to use a single explicit origin plus `credentials: true`, so that cookies travel correctly when a frontend is added later and we never use a permissive `*` origin.
20. As a developer, I want every state-changing operation to be POST/PATCH/DELETE (never GET-with-side-effects), so that `sameSite=lax` is a meaningful CSRF mitigation.
21. As a developer, I want passwords hashed with bcrypt cost factor 12, never serialized in responses, and validated against a minimum complexity rule (min 8, upper + lower + digit + symbol), so that we don't ship trivially crackable credentials.
22. As a developer, I want a `409 Conflict` returned on duplicate email at register or update, so that the unique-index violation surfaces as a clean HTTP error.
23. As a developer, I want versioned Prisma migrations committed to the repo (`prisma migrate dev`), so that schema changes are reviewable and replayable.
24. As an API client (Postman/curl during this iteration), I want consistent HTTP status codes (401 / 403 / 404 / 409 / 422) for each failure class, so that error handling is predictable.

## Implementation Decisions

### Modules

- **`AuthModule`** — owns `/auth/register`, `/auth/login`, `/auth/logout`, the `JwtStrategy`, both guards (`JwtAuthGuard`, `RolesGuard`), and the `@Roles` / `@CurrentUser` decorators. **Password hashing/comparison and bcrypt cost configuration live inside `AuthService`** (no separate `PasswordService`).
- **`UsersModule`** — owns `/users/*`. The last-admin and self-op invariants are **inlined inside `UsersService`** as two private helpers (e.g. `#assertNotSelf(actor, targetId)`, `#assertNotLastAdmin(targetId)`), called from `delete` and `changeRole`. No separate `AdminLockoutService`.
- **`PrismaModule`** — globally exported `PrismaService` extending `PrismaClient`, with `onModuleInit` connect and `enableShutdownHooks` for graceful close.
- **`AppModule`** — wires `ConfigModule.forRoot({ validate: zodValidate, isGlobal: true })`, registers `JwtModule.registerAsync` reading config, and applies global `ValidationPipe` + `ClassSerializerInterceptor`. `main.ts` adds `cookie-parser` and CORS using `CORS_ORIGIN`.
- **`TokenService`** (inside `AuthModule`) — only place that knows the JWT payload shape, expiry, signing key, and cookie set/clear semantics. Surface area:
  - `issueForUser(user, res)` — signs JWT with `{ sub, email, role, tokenVersion }` and sets the cookie on `res`.
  - `clear(res)` — clears the cookie.
  - `verify(token)` — verifies signature/expiry, returns the typed payload.

### Persistence

- **Prisma** with PostgreSQL. Schema in `prisma/schema.prisma`, migrations versioned via `prisma migrate dev`.
- `User` model fields: `id (uuid)`, `email (unique, lowercased on save)`, `password (bcrypt hash)`, `name`, `role (enum USER|ADMIN, default USER)`, `tokenVersion (int, default 0)`, `createdAt`, `updatedAt`.
- **Hard delete** only — no `deletedAt`.

### JWT + cookie contract

JWT payload shape (this shape is load-bearing — `JwtStrategy.validate` and `TokenService.issueForUser` must agree on it):

```ts
type JwtPayload = {
  sub: string;          // user.id
  email: string;
  role: 'USER' | 'ADMIN';
  tokenVersion: number;
  iat: number;
  exp: number;
};
```

- Algorithm: HS256, key from `JWT_SECRET` (env, validated min 32 bytes at boot).
- Expiry: `JWT_EXPIRES_IN` (default `7d`).
- Cookie name: `access_token`. Flags: `httpOnly: true`, `sameSite: 'lax'`, `path: '/'`, `secure` driven by `COOKIE_SECURE` env, `maxAge` matching JWT expiry.
- `JwtStrategy` reads from `req.cookies.access_token`, then in `validate(payload)`:
  1. Look up user by `payload.sub`.
  2. Reject if missing.
  3. Reject if `payload.tokenVersion !== user.tokenVersion`.
  4. Return the loaded user (becomes `req.user`).

### `tokenVersion` bump points

- `POST /auth/logout`
- `PATCH /users/update` when `password` is in the body
- `POST /users/:id/role` (always)

Email change does **not** bump `tokenVersion` in this iteration.

### Routes (final)

| Method | Path              | Guards                                | Notes                                                 |
| ------ | ----------------- | ------------------------------------- | ----------------------------------------------------- |
| POST   | `/auth/register`  | Public                                | Sets cookie, returns user                              |
| POST   | `/auth/login`     | Public                                | Sets cookie, returns user                              |
| POST   | `/auth/logout`    | `JwtAuthGuard`                        | Bumps `tokenVersion`, clears cookie                    |
| GET    | `/users/me`       | `JwtAuthGuard`                        | Returns self (full)                                    |
| PATCH  | `/users/update`   | `JwtAuthGuard`                        | Self-update; bumps `tokenVersion` if `password` set    |
| GET    | `/users/:id`      | `JwtAuthGuard`                        | Public profile for USER, full for ADMIN                |
| GET    | `/users`          | `JwtAuthGuard` + `RolesGuard(ADMIN)`  | List all                                               |
| POST   | `/users/:id/role` | `JwtAuthGuard` + `RolesGuard(ADMIN)`  | Body `{ role }`; bumps target's `tokenVersion`         |
| DELETE | `/users/:id`      | `JwtAuthGuard` + `RolesGuard(ADMIN)`  | Hard delete                                            |

### Inlined admin invariants (inside `UsersService`)

Both `delete(actorId, targetId)` and `changeRole(actorId, targetId, newRole)` must:

1. Reject with `409 Conflict` if `actorId === targetId`.
2. If the operation would reduce admin count to zero (target is currently ADMIN, and either delete or demote to USER), count `prisma.user.count({ where: { role: 'ADMIN' } })`; if `<= 1`, reject with `409 Conflict`.

Both checks live as private helpers on `UsersService`. No separate service or guard.

### DTOs

- `RegisterDto` — `email` (IsEmail), `password` (min 8 + regex for upper/lower/digit/symbol), `name` (IsString, MinLength 1).
- `LoginDto` — `email`, `password`.
- `UpdateMeDto` — all optional: `name?`, `email?`, `password?`. Password if present re-validated by the same rule and re-hashed.
- `ChangeRoleDto` — `role` (IsEnum of `Role`).
- Global `ValidationPipe` with `whitelist: true, forbidNonWhitelisted: true, transform: true`. DTO failures surface as `422`.

### Env schema (validated at boot)

```
DATABASE_URL       string, URL
JWT_SECRET         string, min length 32
JWT_EXPIRES_IN     string, default "7d"
NODE_ENV           "development" | "production" | "test"
CORS_ORIGIN        string, URL
COOKIE_SECURE      boolean (coerced from "true"/"false"), default false in dev
ADMIN_EMAIL        string, email
ADMIN_PASSWORD     string, min length 8
```

Validated via Zod (`config/env.schema.ts`) on `ConfigModule.forRoot({ validate })`. App fails to boot on violation.

### CORS, CSRF, security headers

- CORS: single explicit origin from `CORS_ORIGIN`, `credentials: true`, methods `GET, POST, PATCH, DELETE, OPTIONS`.
- CSRF: no middleware; relying on `sameSite=lax` plus the discipline that no GET endpoint mutates state.
- Helmet, rate limiting, Swagger, health check, API versioning — explicitly out of scope.

### Error contract

| Status | When                                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------------------- |
| 401    | Missing/invalid/expired token; `tokenVersion` mismatch; wrong password at login                                  |
| 403    | Authenticated but role doesn't match `@Roles`                                                                    |
| 404    | Admin lookup of nonexistent user                                                                                 |
| 409    | Duplicate email at register/update (Prisma `P2002`); self-delete; self-demote; would-leave-zero-admins violation |
| 422    | `ValidationPipe` rejected DTO                                                                                    |

### Seed

`prisma/seed.ts` upserts an `ADMIN` user from `ADMIN_EMAIL` / `ADMIN_PASSWORD`, bcrypt-hashing the password. Run via `npm run seed` after migrations. Idempotent.

## Testing Decisions

**No automated tests this iteration.** Decision was made deliberately during planning — this is a learning project, the API surface is small, and the team wants to focus on exercising the auth flows via Postman.

If/when tests are added in a follow-up PRD, they should:

- Test external behavior, not internals — hit the HTTP layer with `supertest`, assert on status codes, response bodies, and `Set-Cookie` headers.
- Run against a real Postgres test schema (Docker or a separate database), not against mocked Prisma — the bugs that matter in an auth system live in the guard/strategy/cookie/JWT integration layer, which mocks hide.
- Cover the cross-cutting invariants explicitly: `tokenVersion` mismatch rejects, last-admin lockout rejects, deleted-user JWT rejects, sanitized public profile for `USER` vs full for `ADMIN`.

No prior art in the repo yet (no existing tests).

## Out of Scope

- Refresh tokens (single 7-day access token is enough for this iteration).
- OAuth / social login.
- Email verification.
- Password reset / forgot-password flows.
- 2FA.
- Audit logging.
- Pagination on `GET /users`.
- Rate limiting (`@nestjs/throttler`) on `/auth/login` and `/auth/register`.
- Automated tests (unit and e2e).
- Helmet / security headers, Swagger / OpenAPI docs, health check endpoint, API versioning (`/v1`).
- A `SUPER_ADMIN` role.
- Self-deletion (`DELETE /users/me`) — only admin can delete users.
- Email-change re-verification.
- Bumping `tokenVersion` on email change.

Each of the above is a candidate for a follow-up PRD.

## Further Notes

- This PRD assumes Linear MCP is not yet wired up in the working environment; once the `prd-to-linear-issues` skill is operational, this document is structured so it can be sliced into vertical-slice issues by that skill (auth bootstrap, user CRUD, admin endpoints, lockout invariant, seed) without further interview.
- `TokenService` is the single chokepoint for any future change to the JWT/cookie contract. If/when refresh tokens are added, every change should land inside that service.
- The decision to inline the admin invariants in `UsersService` is deliberate — at two call sites with a three-line body each, an extracted service would be over-engineering. Revisit if a third call site appears or the rules grow conditional.
