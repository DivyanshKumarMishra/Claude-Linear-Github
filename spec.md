# Nest Auth — Authentication System Spec

A NestJS authentication system using JWT stored in HTTP-only cookies, with role-based access control.

## Goals

- Stateless authentication via JWT, but stored in HTTP-only cookies (not Authorization headers) to keep tokens out of JS.
- Standard user lifecycle: register, login, fetch self, update self.
- Admin-only routes for listing all users, deleting any user, and changing a user's role.
- Two guards: `JwtAuthGuard` (authenticated) and `RolesGuard` (authorized by role).
- Revocable sessions via a `tokenVersion` field — logout, role change, and password change can invalidate existing JWTs immediately.

## Tech stack

- **Framework**: NestJS (current LTS)
- **DB**: PostgreSQL via **Prisma** (schema in `prisma/schema.prisma`, versioned migrations via `prisma migrate dev`)
- **Auth**: `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt`
- **Password hashing**: `bcrypt` (cost factor 12)
- **Cookies**: `cookie-parser`
- **Validation**: `class-validator` + `class-transformer`
- **Config**: `@nestjs/config` with a Zod (or Joi) schema; missing/invalid env vars must fail at boot

## Data model

### `User` model (Prisma)

| Field          | Type      | Notes                                                            |
| -------------- | --------- | ---------------------------------------------------------------- |
| `id`           | uuid      | Primary key                                                      |
| `email`        | string    | Unique, lowercased on save                                       |
| `password`     | string    | bcrypt hash; never serialized                                    |
| `name`         | string    |                                                                  |
| `role`         | enum      | `USER` \| `ADMIN`, default `USER`                                |
| `tokenVersion` | int       | Default `0`. Bumped to revoke all existing JWTs for this user.   |
| `createdAt`    | timestamp |                                                                  |
| `updatedAt`    | timestamp |                                                                  |

The `password` field must be excluded from every response (`@Exclude()` + `ClassSerializerInterceptor`, or explicit DTO mapping).

Hard delete only — no `deletedAt`. Admin deletion removes the row.

## JWT + Cookie contract

- **Algorithm**: HS256 with `JWT_SECRET` from env.
- **Payload**: `{ sub: userId, email, role, tokenVersion }`.
- **Expiry**: 7 days (configurable via `JWT_EXPIRES_IN`).
- **Cookie name**: `access_token`.
- **Cookie flags**: `httpOnly: true`, `secure: true` in prod / `false` in dev, `sameSite: 'lax'`, `path: '/'`, `maxAge` matching JWT expiry.
- On login/register success → set cookie, return user (no token in body).
- On logout → bump `user.tokenVersion`, then clear the cookie. This invalidates the JWT server-side as well, so a stolen cookie becomes useless.
- On every authenticated request:
  1. Read cookie, verify JWT signature/expiry.
  2. Look up the user by `payload.sub` (DB hit per request — accepted cost).
  3. Reject if user no longer exists.
  4. Reject if `payload.tokenVersion !== user.tokenVersion`.
  5. Attach `user` to the request.

### When `tokenVersion` is bumped

- Logout
- Password change via `PATCH /users/update`
- Admin demotes/promotes the user via `POST /users/:id/role` (role change must take effect immediately)

## Routes

All routes live under `/auth` and `/users`.

### Auth routes (`/auth`)

| Method | Path             | Guard          | Body                          | Response                            |
| ------ | ---------------- | -------------- | ----------------------------- | ----------------------------------- |
| POST   | `/auth/register` | Public         | `{ email, password, name }`   | `User` (sets cookie)                |
| POST   | `/auth/login`    | Public         | `{ email, password }`         | `User` (sets cookie)                |
| POST   | `/auth/logout`   | `JwtAuthGuard` | —                             | `{ success: true }` (clears cookie) |

### User routes (`/users`)

| Method | Path               | Guards                              | Body                            | Response               |
| ------ | ------------------ | ----------------------------------- | ------------------------------- | ---------------------- |
| GET    | `/users/me`        | `JwtAuthGuard`                      | —                               | `User` (self, full)    |
| PATCH  | `/users/update`    | `JwtAuthGuard`                      | `{ name?, email?, password? }`  | `User` (self, full)    |
| GET    | `/users/:id`       | `JwtAuthGuard`                      | —                               | `PublicUser` or `User` |
| GET    | `/users`           | `JwtAuthGuard` + `RolesGuard(ADMIN)`| —                               | `User[]`               |
| POST   | `/users/:id/role`  | `JwtAuthGuard` + `RolesGuard(ADMIN)`| `{ role: 'USER' \| 'ADMIN' }`   | `User`                 |
| DELETE | `/users/:id`       | `JwtAuthGuard` + `RolesGuard(ADMIN)`| —                               | `{ success: true }`    |

Notes:

- A regular `USER` updates **their own** profile via `PATCH /users/update`. They cannot update other users — there is no `PATCH /users/:id`.
- `GET /users/:id` returns a sanitized `PublicUser` (`{ id, name, role, createdAt }`) for callers with role `USER`, and the full `User` (minus `password`) for callers with role `ADMIN`. Branch in the controller on `req.user.role`.
- Admin-only routes: **`GET /users`** (list all), **`POST /users/:id/role`** (change a user's role), **`DELETE /users/:id`** (delete any user).
- Pagination on `GET /users` is intentionally skipped this iteration — add when the user count justifies it.
- Self-deletion is not exposed.

### Lockout / last-admin guard (service layer)

Before performing `DELETE /users/:id` or `POST /users/:id/role`, the service must reject with `409 Conflict` if any of these hold:

- The acting admin is the target (`req.user.id === params.id`) — no self-delete, no self-demote.
- The target is the last `ADMIN` and the operation would leave the system with zero admins. Compute with `prisma.user.count({ where: { role: 'ADMIN' } })`.

Apply the same helper in both routes.

## Guards

### `JwtAuthGuard`

- Extracts JWT from the `access_token` cookie.
- Verifies signature and expiry.
- Loads the user from DB by `payload.sub`.
- Rejects if user missing or `payload.tokenVersion !== user.tokenVersion`.
- Attaches `user` to the request.
- Throws `UnauthorizedException` on any failure.

Implementation: extend `AuthGuard('jwt')` from `@nestjs/passport`, with a `JwtStrategy` whose `jwtFromRequest` reads from `req.cookies.access_token` and whose `validate(payload)` performs the DB lookup + `tokenVersion` check.

### `RolesGuard`

- Reads required roles from a `@Roles(...roles)` decorator on the handler/controller.
- Compares against `req.user.role`.
- Throws `ForbiddenException` if mismatch.

Usage:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(Role.ADMIN)
@Get()
findAll() { ... }
```

## DTOs / validation

- `RegisterDto`: `email` (IsEmail), `password` (see password policy), `name` (IsString, not empty).
- `LoginDto`: `email`, `password`.
- `UpdateMeDto`: `name?`, `email?`, `password?` — all optional. If `password` is present, re-hash and bump `tokenVersion`.
- `ChangeRoleDto`: `role` (IsEnum of `Role`).

### Password policy

- Minimum length 8.
- Must contain at least one uppercase letter, one lowercase letter, one digit, and one symbol.
- Implemented via `@Matches(/regex/)` in `class-validator`.

Use a `ValidationPipe` globally with `whitelist: true, forbidNonWhitelisted: true, transform: true`.

## Error handling

- `401 Unauthorized` — missing/invalid/expired token, wrong password, `tokenVersion` mismatch.
- `403 Forbidden` — role mismatch.
- `404 Not Found` — admin lookup of nonexistent user.
- `409 Conflict` — registering or updating to an email that already exists (Prisma `P2002`); self-delete or self-demote attempt; operation would leave zero admins.
- `422 Unprocessable Entity` — DTO validation failure.

## Module layout

```
src/
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── strategies/
│   │   └── jwt.strategy.ts
│   ├── guards/
│   │   ├── jwt-auth.guard.ts
│   │   └── roles.guard.ts
│   ├── decorators/
│   │   ├── roles.decorator.ts
│   │   └── current-user.decorator.ts
│   └── dto/
│       ├── register.dto.ts
│       └── login.dto.ts
├── users/
│   ├── users.module.ts
│   ├── users.controller.ts
│   ├── users.service.ts
│   └── dto/
│       ├── update-me.dto.ts
│       ├── change-role.dto.ts
│       └── public-user.dto.ts
├── prisma/
│   ├── prisma.module.ts
│   └── prisma.service.ts
├── config/
│   └── env.schema.ts
├── common/
│   └── enums/
│       └── role.enum.ts
└── main.ts

prisma/
├── schema.prisma
├── migrations/
└── seed.ts
```

## Environment variables

```
DATABASE_URL=postgres://...
JWT_SECRET=<random 32+ byte string>
JWT_EXPIRES_IN=7d
NODE_ENV=development
CORS_ORIGIN=http://localhost:3001    # explicit dev origin for the future frontend
COOKIE_SECURE=false                  # true in prod
ADMIN_EMAIL=admin@example.com        # consumed by prisma/seed.ts
ADMIN_PASSWORD=<strong password>     # consumed by prisma/seed.ts
```

All variables are validated at boot by the `ConfigModule` schema. App fails to start if any are missing or malformed.

## First admin bootstrap

- `prisma/seed.ts` reads `ADMIN_EMAIL` and `ADMIN_PASSWORD` from env.
- Upserts a user with `role: 'ADMIN'`, bcrypt-hashed password.
- Run via `npm run seed` after `prisma migrate dev`.
- No public route for creating admins.

## CORS

- Single explicit origin from `CORS_ORIGIN` env var.
- `credentials: true` (required so the browser sends cookies cross-origin).
- Methods: `GET, POST, PATCH, DELETE, OPTIONS`.
- For now there is no real frontend — Postman/curl test against `http://localhost:3000` directly.

## CSRF

No CSRF middleware. We rely on:

- `sameSite=lax` on the auth cookie (blocks classic cross-site form CSRF).
- A discipline rule: **every state-changing operation must be POST / PATCH / DELETE — never a GET with side effects**. GET handlers must be read-only.

If a real cross-origin frontend lands later, revisit and add CSRF tokens.

## Security checklist

- [ ] Passwords hashed with bcrypt, cost ≥ 12, never logged.
- [ ] JWT secret from env, never committed; min 32 bytes; validated at boot.
- [ ] Cookie is `httpOnly`, `secure` in prod, `sameSite=lax`.
- [ ] CORS configured with explicit origin (`CORS_ORIGIN`) + `credentials: true`.
- [ ] `email` is lowercased + unique-indexed.
- [ ] `password` field excluded from responses (no leaks via admin endpoints).
- [ ] `tokenVersion` checked on each request, so deleted/logged-out/role-changed users can't keep using a valid JWT.
- [ ] Lockout guard prevents self-delete, self-demote, and zero-admin states.
- [ ] First admin seeded via `prisma/seed.ts`, not via a public route.
- [ ] All mutating operations use POST/PATCH/DELETE (CSRF discipline).
- [ ] Env vars validated at boot via `ConfigModule` schema.

## Tests

No automated tests this iteration. Manual verification via Postman against a local Postgres.

## Out of scope

- Refresh tokens (single 7-day access token is enough for this iteration).
- OAuth / social login.
- Email verification, password reset flows.
- 2FA.
- Audit logging.
- Pagination on `GET /users`.
- Rate limiting (`@nestjs/throttler`).
- Automated tests (unit / e2e).
- Helmet / security headers, Swagger docs, health check, API versioning.

These become follow-up PRDs.

## Open questions

1. Do we need a `SUPER_ADMIN` role, or is `ADMIN` enough?
2. Should email changes require re-verification? (Currently no; deferred until email verification exists.)
3. Should `tokenVersion` also bump on email change? (Currently no — only password change, logout, and role change.)
