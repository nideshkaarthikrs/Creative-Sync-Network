# CSN Backend — CLAUDE.md

## Completed Services

| Service | Port | Status |
|---|---|---|
| identity-service | 3001 | Done |
| profile-service | 3002 | Done |
| tune-service | 3003 | Done |
| lyrics-service | 3004 | Done |
| voice-service | 3005 | Done |

## In-Progress Services

None.

## Pending Services

| Service | Port |
|---|---|
| video-service | 3006 |
| project-service | 3007 |
| chat-service | 3008 |
| voting-service | 3009 |
| feed-service | 3010 |
| rights-service | 3011 |
| payment-service | 3012 |

## Conventions & Patterns Established

### Source of truth
REST API contracts (`CSN REST APIs.pdf`) are authoritative. If the context PDF and the contracts conflict, always follow the contracts.

### UserId format
- PostgreSQL PK: UUID (`@id @default(uuid())`)
- Human-readable display ID: `USR` + zero-padded `sequenceNumber` (auto-increment). E.g. `USR100001`.
- The `displayId` is computed in the service layer: `"USR" + sequenceNumber.toString().padStart(6, "0")`. It is returned in all API responses as `userId`.

### Response envelope
All endpoints return the standard CSN envelope:
```json
{ "status": "SUCCESS", "message": "...", "data": {} }
{ "status": "ERROR",   "errorCode": "CSN-XXXX", "message": "..." }
```
Exceptions: register/login responses place top-level fields (userId, token, etc.) directly in the root per the contract.

### Tech stack (per service)
- Framework: NestJS + TypeScript
- ORM: Prisma (schema in `prisma/schema.prisma`)
- DB: PostgreSQL (each service owns its own DB)
- Auth: JWT via `@nestjs/jwt` + `@nestjs/passport` + `passport-jwt`
- Validation: `class-validator` DTOs with global `ValidationPipe`
- Secrets: `.env` (never committed); `.env.example` committed

### Folder structure (per service)
```
<service>/
├── src/
│   ├── main.ts               # Bootstrap + global pipes, port from config
│   ├── app.module.ts         # Root module
│   ├── config/configuration.ts
│   ├── prisma/prisma.service.ts
│   ├── shared/response.helper.ts
│   ├── auth/
│   │   ├── jwt.strategy.ts           # Validates Bearer tokens using shared JWT_SECRET
│   │   ├── jwt-auth.guard.ts         # Required JWT guard
│   │   └── optional-jwt-auth.guard.ts # Optional JWT guard (public routes that benefit from user context)
│   └── <feature>/
│       ├── <feature>.module.ts
│       ├── <feature>.controller.ts
│       ├── <feature>.service.ts
│       ├── <feature>.repository.ts
│       └── dto/
├── prisma/schema.prisma
├── Dockerfile
├── docker-compose.yml        # Local dev DB
├── .env                      # Local dev (not committed)
├── .env.example
└── package.json
```

### JWT payload
All services sign/validate JWTs with: `{ sub: uuid, userId: "USR000001", name: "Arun Kumar", roles: [...] }`.
The `name` field was added to identity-service in the profile-service iteration to support profile bootstrapping.

### JWT guard pattern
- `JwtAuthGuard` — required auth, returns 401 if no/invalid token
- `OptionalJwtAuthGuard` — passes through if no token; sets `req.user` if valid token present. Use on public GET endpoints that need owner-context (e.g., auto-bootstrap).

### Profile bootstrapping
Profile-service auto-creates a `Profile` record on the first `GET /profiles/:userId` call when the caller is the owner (JWT `userId` matches path param). Name and roles are seeded from JWT claims. Subsequent public GETs read from DB.

### PostgreSQL port convention
Each service runs its own PostgreSQL container on a unique host port:
- identity-service: 5432
- profile-service: 5433
- tune-service: 5434
- lyrics-service: 5435
- voice-service: 5436
- (next services increment by 1)

### Local development (identity-service)
```bash
cd csn-backend/identity-service
docker compose up -d          # start PostgreSQL on port 5432
npx prisma migrate dev        # run migrations
npm run start:dev             # start on port 3001
```

### Local development (profile-service)
```bash
cd csn-backend/profile-service
docker compose up -d          # start PostgreSQL on port 5433
npx prisma migrate dev        # run migrations
npm run start:dev             # start on port 3002
```

## Pending Decisions

- **OTP (mobile verification)**: Mentioned in context PDF but not in REST API contracts. Not built. Needs contract before implementation.
- **Social OAuth (Google login)**: Mentioned in context PDF but not in REST API contracts. Not built. Needs contract before implementation.
- **Photo storage**: profile-service and tune-service currently use multer disk storage (`uploads/`) as a placeholder. Real implementation needs AWS S3 + CDN URL.

### TuneId format
- `tuneId` in responses = `"TUN" + (1000 + sequenceNumber)` → e.g. `TUN1001`, `TUN1002`
- Path params use the display ID; service parses: strip `TUN`, parseInt, subtract 1000, query by `sequenceNumber`

### LyricsId format
- `lyricsId` in responses = `"LYR" + (2000 + sequenceNumber)` → e.g. `LYR2001`, `LYR2002`
- Path params use the display ID; service parses: strip `LYR`, parseInt, subtract 2000, query by `sequenceNumber`

### Approve endpoint auth pattern (MVP)
lyrics-service `POST /lyrics/:lyricsId/approve` gates on `COMPOSER` role from JWT. Full cross-service ownership check (confirm caller owns the tune) is deferred — would require an HTTP call to tune-service.

### Local development (tune-service)
```bash
cd csn-backend/tune-service
docker compose up -d          # start PostgreSQL on port 5434
npx prisma migrate dev        # run migrations
npm run start:dev             # start on port 3003
```

### Local development (lyrics-service)
```bash
cd csn-backend/lyrics-service
docker compose up -d          # start PostgreSQL on port 5435
npx prisma migrate dev        # run migrations
npm run start:dev             # start on port 3004
```

### PerformanceId format
- `performanceId` in responses = `"PER" + (3000 + sequenceNumber)` → e.g. `PER3001`, `PER3002`
- Path params use the display ID; service parses: strip `PER`, parseInt, subtract 3000, query by `sequenceNumber`

### Local development (voice-service)
```bash
cd csn-backend/voice-service
docker compose up -d          # start PostgreSQL on port 5436
npx prisma migrate dev        # run migrations
npm run start:dev             # start on port 3005
```

## What to Build Next

**project-service (port 3007) + chat-service (port 3008)** — collaboration workspace and messaging: `POST /projects`, `POST /projects/:projectId/invite`, `GET /projects/:projectId/members`, `GET /projects/:projectId/files`, `POST /projects/:projectId/messages`, `GET /projects/:projectId/messages`.
